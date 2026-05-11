# Access Manager — Översikt

Källkod: [ducklake-access-manager](https://github.com/wildrelation/ducklake-access-manager)

## Vad gör access managern?

Access managern är en webbapplikation med ett enkelt jobb: **automatisera skapandet av credentials till DuckLake**. Utan den skulle vi behöva skapa PostgreSQL-användare och S3-nycklar manuellt för varje student.

---

## Exakt vad händer när en student genererar en nyckel?

```
Student klickar "Generate Key"
         │
         ▼
1. Validera att bucket-namnet är giltigt (a-z, 0-9, bindestreck)
         │
         ▼
2. Slå upp dataset-raden för bucketen
   - Finns inget dataset → returnera 404
   - Ger oss datasetets pgDatabase (t.ex. dl_titanic_2026)
         │
         ▼
3. Kontrollera behörighet
   - "readwrite" kräver admin-roll → returnera 403 om inte admin
   - Dataset är private och studenten saknar grant → returnera 403
   - Grant kan vara: user (e-post), group (gruppmedlemskap), everyone
         │
         ▼
4. Skapa PostgreSQL-användare (i datasetets egna databas)
   - Generera namn: dl_ro_{8 hex-tecken}  (eller dl_rw_ för readwrite)
   - Generera lösenord: slumpmässigt UUID
   - CREATE USER → GRANT CONNECT på dataset-DB → GRANT SELECT (alla tabeller)
         │
         ▼  ← om steg 5 eller 6 misslyckas: radera PG-användaren (rollback)
5. Skapa S3-nyckel i Garage
   - POST /v2/CreateKey  → får Key ID + Secret
     (nyckelnamn: "key-{bucket}|{pgUsername}|{permission}")
   - GET  /v2/GetBucketInfo → hämtar bucket-ID
   - POST /v2/AllowBucketKey → ger nyckeln rätt behörighet
         │
         ▼  ← om steg 6 misslyckas: radera S3-nyckel + PG-användare (rollback)
6. Spara ägarskap
   - INSERT INTO key_user_mapping (garage_key_id, keycloak_sub, email, pg_database)
         │
         ▼
7. Bygg DuckDB-script och .env-fil med alla credentials
         │
         ▼
8. Returnera { s3Key, dbCredentials, duckdbScript, envFile } till webbläsaren
```

### Rollback-logik (saga-pattern)

Skapandet av en nyckel berör tre externa resurser (PG-användare, S3-nyckel, mapping-rad). Om ett steg misslyckas rensar tjänsten upp allt som skapats dessförinnan:

| Steg som misslyckas | Rollback |
|---|---|
| S3-nyckel (steg 5) | Radera PG-användaren |
| Mapping-rad (steg 6) | Radera S3-nyckeln + PG-användaren |

`deleteUser()` är idempotent — kastar inte fel om användaren redan är borta.

---

## Behörighetsmodellen

### Grants — tre typer

| Typ | Vem matchar |
|---|---|
| `user` | Specifik e-postadress |
| `group` | Alla studenter i en namngiven grupp |
| `everyone` | Alla inloggade användare |

Grants hanteras av admin via **Admin → Grants** i UI:t eller `POST /api/admin/grants`.

### Grupper

Admin skapar grupper och lägger till studenter via e-post. Studenter kan läggas till en i taget eller massimporteras via en textarea (komma/semikolon/radbrytning-separerade e-postadresser). Resultatet rapporteras som `{added, skipped, invalid}`.

Grupptillhörighet kollas automatiskt i `AccessService.hasAccess()` — studenten behöver inte göra något.

### Per-dataset databasisolering

PG-användaren skapas i datasetets **egna** databas (`dl_titanic_2026`, `dl_iris` osv.), inte i en delad katalog-DB. Det innebär att en student med access till dataset A bokstavligen inte kan läsa dataset B:s katalogtabeller — det saknas `CONNECT`-privilegium.

### Åtkomstmatris

| Endpoint | Vanlig student | Admin |
|----------|---------------|-------|
| Generera read-only nyckel | ✅ (kräver grant eller public dataset) | ✅ |
| Generera read/write nyckel | ❌ 403 | ✅ |
| Lista nycklar | Bara egna | Alla |
| Radera nyckel | Bara egna | Alla |
| Hantera datasets | ❌ 403 | ✅ |
| Hantera groups/grants | ❌ 403 | ✅ |

Admin-rollen sätts i Keycloak, se [keycloak.md](../komponenter/keycloak.md).

---

## Varför är Secret Key synlig bara en gång?

S3 Secret Keys fungerar som lösenord — Garage lagrar bara en hash av dem. När access managern skapar en nyckel returnerar Garage secret key en enda gång. Vi lagrar den inte. Om en student stänger dialogen utan att kopiera sin secret key måste de radera nyckeln och generera en ny.

Studenten kan också ladda ned en `.env`-fil direkt från nyckeldialogen — den innehåller alla credentials i standardformat (`PGHOST`, `AWS_ACCESS_KEY_ID` m.m.) som fungerar med `psycopg2`, `boto3` och `s3fs` utan ytterligare konfiguration.
