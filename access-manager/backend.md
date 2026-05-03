# Backend — Java-koden förklarad

Källkod: [ducklake-access-manager/src/main/java](https://github.com/wildrelation/ducklake-access-manager/tree/main/src/main/java/com/ducklake/accessmanager)

## Paketstruktur

```
api/                    ← REST-endpoints (vad omvärlden ser)
  KeyController.java    ← /api/keys/*
  HealthController.java ← /healthz

config/
  SecurityConfig.java   ← vilka endpoints kräver inloggning

service/                ← interfaces (kontrakt)
  ObjectStoreAccessTokenManager.java
  DatabaseAccessTokenManager.java
  KeyMappingService.java
  impl/                 ← den faktiska koden
    GarageAccessTokenManager.java    ← pratar med Garage
    PostgresAccessTokenManager.java  ← pratar med PostgreSQL
    PostgresKeyMappingService.java   ← hanterar key_user_mapping

infrastructure/garage/  ← dataklasser för Garage API-svar
  GarageKeyResponse.java
  GarageBucketResponse.java
  GarageKeyListItem.java

model/                  ← dataklasser för vårt API
  AccessKey.java
  DbCredentials.java
  GeneratedCredentials.java
  KeyListItem.java       ← används för GET /api/keys (inkl. createdBy)
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

## PostgresAccessTokenManager — rå SQL

Vi använder `JdbcTemplate` och skriver SQL direkt istället för ett ORM-verktyg (som Hibernate). Det beror på att vi kör DDL-satser (`CREATE USER`, `DROP USER`) som ORM-verktyg inte hanterar bra.

```java
jdbc.execute("CREATE USER " + username + " WITH PASSWORD '" + password + "'");
```

Det ser ut som SQL-injektion men är säkert här eftersom `username` och `password` alltid är maskingenerade (UUID-baserade) — aldrig användarinmatning.

---

## PostgresKeyMappingService — ägarskapsregistret

Den här klassen skapar sin tabell automatiskt vid uppstart om den inte redan finns:

```java
jdbc.execute("""
    CREATE TABLE IF NOT EXISTS key_user_mapping (
        garage_key_id VARCHAR PRIMARY KEY,
        keycloak_sub  VARCHAR NOT NULL,
        display_name  VARCHAR,
        created_at    TIMESTAMP DEFAULT NOW()
    )
""");
```

`findDisplayNames()` hämtar emails för en lista av nyckel-ID:n i en enda SQL-fråga (IN-clause) för att undvika N+1-problemet:

```java
"SELECT garage_key_id, display_name FROM key_user_mapping WHERE garage_key_id IN (?, ?, ...)"
```
