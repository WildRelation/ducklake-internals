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
2. Kontrollera behörighet
   - "readwrite" kräver admin-roll → returnera 403 om inte admin
         │
         ▼
3. Skapa PostgreSQL-användare
   - Generera namn: dl_ro_{8 hex-tecken}
   - Generera lösenord: slumpmässigt UUID
   - CREATE USER ... WITH PASSWORD ...
   - GRANT SELECT ON ALL TABLES ...
         │
         ▼
4. Skapa S3-nyckel i Garage
   - POST /v2/CreateKey  → får Key ID + Secret
   - GET  /v2/GetBucketInfo → hämtar bucket-ID
   - POST /v2/AllowBucketKey → ger nyckeln rätt behörighet
         │
         ▼
5. Spara ägarskap
   - INSERT INTO key_user_mapping (garage_key_id, keycloak_sub, email)
         │
         ▼
6. Bygg DuckDB-script med alla credentials
         │
         ▼
7. Returnera { s3Key, dbCredentials, duckdbScript } till webbläsaren
```

---

## Behörighetsmodellen

| Endpoint | Vanlig student | Admin |
|----------|---------------|-------|
| Generera read-only nyckel | ✅ | ✅ |
| Generera read/write nyckel | ❌ 403 | ✅ |
| Lista nycklar | Bara egna | Alla |
| Radera nyckel | Bara egna | Alla |

Admin-rollen sätts i Keycloak, se [keycloak.md](../komponenter/keycloak.md).

---

## Varför är Secret Key synlig bara en gång?

S3 Secret Keys fungerar som lösenord — Garage lagrar bara en hash av dem. När access managern skapar en nyckel returnerar Garage secret key en enda gång. Vi lagrar den inte. Om en student stänger dialogen utan att kopiera sin secret key måste de radera nyckeln och generera en ny.
