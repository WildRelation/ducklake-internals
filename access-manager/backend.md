# Backend — Java-koden förklarad

Källkod: [ducklake-access-manager/src/main/java](https://github.com/wildrelation/ducklake-access-manager/tree/main/src/main/java/com/ducklake/accessmanager)

## Paketstruktur

```
api/                       ← REST-endpoints (vad omvärlden ser)
  AdminController.java     ← /api/admin/** (buckets + generaliserade grants, kräver admin)
  BucketController.java    ← /api/buckets (bucket-lista per användare)
  ConfigController.java    ← /api/config (publik Keycloak-konfiguration)
  DatasetController.java   ← /api/datasets/** (dataset CRUD)
  GroupController.java     ← /api/groups/** (grupp CRUD + medlemmar, kräver admin)
  HealthController.java    ← /healthz
  KeyController.java       ← /api/keys (generera, lista, ta bort nycklar)

config/
  SecurityConfig.java      ← vilka endpoints kräver inloggning, isAdmin()

service/                   ← interfaces (kontrakt)
  DatabaseAccessTokenManager.java     ← createReadOnlyUser(db), createReadWriteUser(db),
  │                                      deleteUser(user, db), listUsers()
  KeyMappingService.java              ← saveMapping, findDatabase, findOwner,
  │                                      findKeyIdsForUser, findDisplayNames, findCreatedAts
  ObjectStoreAccessTokenManager.java  ← listBuckets, createBucket, deleteBucket,
                                         createReadOnlyKey, createReadWriteKey,
                                         deleteKey, listKeys
  impl/                    ← den faktiska koden
    AccessService.java               ← dataset_grants: hasAccess(), visibleBucketNames(),
    │                                   auto-migrering från student_grants
    DatasetService.java              ← Dataset CRUD + atomär bucket+DB-livscykel,
    │                                   startup-sync av orphan-buckets
    GarageAccessTokenManager.java    ← Garage Admin API v2 (HTTP mot port 3900)
    GroupService.java                ← groups + group_members CRUD, bulk-import
    PostgresAccessTokenManager.java  ← JDBC: skapar dl_ro_/dl_rw_-användare per dataset-DB
    PostgresAdminOps.java            ← CREATE/DROP DATABASE, jdbcFor(db) för in-DB grants
    PostgresKeyMappingService.java   ← hanterar key_user_mapping-tabellen

infrastructure/garage/     ← dataklasser för Garage API-svar
  GarageKeyResponse.java
  GarageBucketResponse.java
  GarageKeyListItem.java

model/                     ← dataklasser för vårt API
  AccessKey.java
  Bucket.java
  BucketGrant.java
  Dataset.java
  DbCredentials.java
  GeneratedCredentials.java   ← s3Key, dbCredentials, duckdbScript, envFile
  Grant.java
  Group.java
  KeyListItem.java            ← används för GET /api/keys (inkl. createdBy + createdAt)
  KeyRequest.java
```

---

## KeyController — hjärtat

`KeyController.java` är den klass som tar emot HTTP-anrop och orkesterar allt.

### Varför interfaces?

`KeyController` pratar inte direkt med Garage eller PostgreSQL — det gör det via interfaces:

```java
// KeyController vet bara att det finns något som kan skapa nycklar
private final ObjectStoreAccessTokenManager objectStore;
private final DatabaseAccessTokenManager database;
```

Det här gör det enkelt att byta ut Garage mot t.ex. MinIO eller PostgreSQL mot en annan databas, utan att ändra i `KeyController`.

### Rollback-kompensation

Nyckelgenerering berör tre externa resurser. Om ett steg misslyckas rullas tidigare steg tillbaka:

```java
// Steg 1: Postgres-användare (ingen rollback — första steget)
DbCredentials dbCreds = database.createReadOnlyUser(dataset.pgDatabase());

// Steg 2: S3-nyckel — misslyckas → radera PG-användaren
try {
    s3Key = objectStore.createReadOnlyKey(bucketName, keyName);
} catch (Exception e) {
    database.deleteUser(dbCreds.username(), dataset.pgDatabase());
    throw e;
}

// Steg 3: Spara mapping — misslyckas → radera S3-nyckel + PG-användare
try {
    keyMapping.saveMapping(s3Key.keyId(), sub, displayName, dataset.pgDatabase());
} catch (Exception e) {
    objectStore.deleteKey(s3Key.keyId());
    database.deleteUser(dbCreds.username(), dataset.pgDatabase());
    throw e;
}
```

### Hur vet Spring vilken implementation som ska användas?

Spring Boot letar automatiskt efter klasser annoterade med `@Service` som implementerar interfacen:

```java
@Service
public class GarageAccessTokenManager implements ObjectStoreAccessTokenManager { ... }
```

Det kallas **dependency injection** — Spring "injicerar" rätt klass automatiskt.

---

## Hur JWT-validering fungerar

`SecurityConfig.java` konfigurerar Spring Security att kräva ett giltigt JWT på alla `/api/keys/*`-anrop:

```java
.oauth2ResourceServer(oauth2 -> oauth2.jwt(Customizer.withDefaults()))
```

Spring laddar ner Keycloaks publika nyckel från JWKS-endpoint (`/realms/cloud/protocol/openid-connect/certs`) en gång vid uppstart och verifierar sedan alla inkommande tokens lokalt. Det innebär att Keycloak inte behöver kontaktas för varje anrop.

### Hur admin-checken fungerar

```java
private boolean isAdmin(Jwt jwt) {
    Map<String, Object> resourceAccess = jwt.getClaimAsMap("resource_access");
    Map<String, Object> clientAccess = (Map<String, Object>) resourceAccess.get("ducklake");
    List<String> roles = (List<String>) clientAccess.get("roles");
    return roles != null && roles.contains("admin");
}
```

JWT:et innehåller `resource_access.ducklake.roles` — en lista med roller för just `ducklake`-klienten. Om `"admin"` finns i den listan är användaren admin.

---

## GarageAccessTokenManager — tre steg för att skapa en nyckel

Garage kräver tre separata API-anrop för att skapa en nyckel med behörighet:

```java
// Steg 1: Skapa nyckeln, få Key ID + Secret
GarageKeyResponse key = postCreateKey(keyName);

// Steg 2: Hämta bucket-ID från bucket-namn
String bucketId = getBucketId(bucketName);

// Steg 3: Ge nyckeln rätt behörighet på bucketen
grantBucketPermission(bucketId, key.accessKeyId(), read, write);
```

`RestTemplate` är Springs inbyggda HTTP-klient — vi behöver inte något externt bibliotek.

---

## PostgresAccessTokenManager — per-dataset isolering

Vi använder `JdbcTemplate` och skriver SQL direkt istället för ett ORM-verktyg (som Hibernate). Det beror på att vi kör DDL-satser (`CREATE USER`, `GRANT CONNECT`) som ORM-verktyg inte hanterar bra.

PG-användaren skapas i datasetets **egna** databas (t.ex. `dl_titanic_2026`), inte i en delad katalog. Det kräver två JDBC-anslutningar:

```java
// 1. Skapa användaren på clusternivå (ansluten till admin-DB)
jdbc.execute("CREATE USER " + username + " WITH PASSWORD '" + password + "'");
jdbc.execute("GRANT CONNECT ON DATABASE " + database + " TO " + username);

// 2. Anslut till dataset-DB:n och ge SELECT-rättigheter där
JdbcTemplate datasetJdbc = pgAdmin.jdbcFor(database);
datasetJdbc.execute("GRANT USAGE ON SCHEMA public TO " + username);
datasetJdbc.execute("GRANT SELECT ON ALL TABLES IN SCHEMA public TO " + username);
```

`PostgresAdminOps.jdbcFor(database)` skapar en ny `JdbcTemplate` ansluten till den specifika databasen.

---

## AccessService — behörighetskontroll och grants

`AccessService` ersätter gamla `GrantService` och hanterar den generaliserade grant-modellen:

```java
// Tre typer av grants — kontrolleras i tur och ordning
public boolean hasAccess(String email, String bucketName) {
    // 1. User-grant (direkt e-post)
    // 2. Everyone-grant (alla inloggade)
    // 3. Group-grant (via group_members-join)
}
```

Vid uppstart migreras automatiskt gamla `student_grants`-rader till `dataset_grants` som user-grants, och befintliga Garage-buckets registreras som datasets med `visibility=public`.

---

## PostgresKeyMappingService — ägarskapsregistret

Tabellen har utökats med `pg_database`-kolumnen för att spåra vilken dataset-DB varje nyckel tillhör:

```sql
key_user_mapping (
    garage_key_id  VARCHAR PRIMARY KEY,
    keycloak_sub   VARCHAR NOT NULL,
    display_name   VARCHAR,
    created_at     TIMESTAMP DEFAULT NOW(),
    pg_database    VARCHAR    -- vilken dataset-DB (null = legacy)
)
```

`ALTER TABLE ... ADD COLUMN IF NOT EXISTS` körs vid uppstart — ingen manuell migration krävs.

`findDisplayNames()` och `findCreatedAts()` hämtar data för en lista av nyckel-ID:n i en enda SQL-fråga (IN-clause) för att undvika N+1-problemet.

---

## GroupService — grupper och massimport

`GroupService` hanterar `groups` och `group_members`-tabellerna. Bulk-import tar en lista av e-postadresser och returnerar ett sammanfattningsobjekt:

```java
public Map<String, Integer> addMembers(String groupName, List<String> emails) {
    // För varje e-post: INSERT ... ON CONFLICT DO NOTHING
    // Räknar added / skipped (redan med) / invalid (saknar @)
    return Map.of("added", added, "skipped", skipped, "invalid", invalid);
}
```

---

## Nyckelnamnsformatet

Nyckelnamnet som skapas i Garage bäddar in metadata som behövs vid listning:

```
key-{bucketName}|{pgUsername}|{permission}
```

Exempel: `key-titanic-2026|dl_ro_a8f0d0a4|read`

Garage `ListKeys` returnerar inte behörighet per nyckel — utan detta format skulle permission alltid visas som okänd i My Keys-vyn. Gamla nycklar utan permission-fält hanteras bakåtkompatibelt.
