# PostgreSQL — DuckLake-katalogen

## Vad PostgreSQL gör i DuckLake

DuckLake-formatet kräver en **katalogdatabas** som håller koll på:
- Vilka tabeller som finns
- Var i S3 filerna för varje tabell lagras
- Versionshistorik (snapshots) — DuckLake stödjer tidresande i data

Utan PostgreSQL vet inte DuckDB var filerna är. Det är PostgreSQL som svarar på frågan *"var är Parquet-filerna för tabellen `titanic`?"*

---

## Två typer av användare

PostgreSQL-databasen har två kategorier av användare:

### Admin-kontot
```
Användarnamn: ducklake
Lösenord:     cbhcloud
```
Det här kontot äger databasen och all data. Access managern använder det för att skapa och ta bort student-användare. Det sätts via miljövariabler i access manager-deploymentet.

> Ändra aldrig lösenordet utan att uppdatera miljövariablerna i access manager-deploymentet.

### Student-konton (provisionerade automatiskt)
```
Prefix:  dl_ro_  (read-only)
         dl_rw_  (read/write)
Suffix:  8 slumpmässiga hex-tecken
Exempel: dl_ro_a3f2b1c9
```

Access managern skapar dessa automatiskt när en student genererar en nyckel. Lösenordet är ett slumpmässigt UUID.

**Read-only får:**
```sql
GRANT SELECT ON ALL TABLES IN SCHEMA public TO {username};
```

**Read/write får:**
```sql
GRANT SELECT, INSERT, UPDATE, DELETE ON ALL TABLES IN SCHEMA public TO {username};
```

---

## key_user_mapping-tabellen

Det här är en tabell som access managern själv skapar och äger (inte en DuckLake-intern tabell):

```sql
CREATE TABLE key_user_mapping (
    garage_key_id VARCHAR PRIMARY KEY,  -- Garage S3-nyckelns ID (GKxxx...)
    keycloak_sub  VARCHAR NOT NULL,     -- Användarens Keycloak-UUID (ändras aldrig)
    display_name  VARCHAR,              -- Användarens email (t.ex. jpsa2@kth.se)
    created_at    TIMESTAMP DEFAULT NOW()
);
```

Den används för att:
1. Visa vem som skapat vilken nyckel i UI:t (Created By-kolumnen)
2. Kontrollera att användare bara kan radera sina egna nycklar
3. Filtrera `GET /api/keys` så studenter bara ser egna nycklar

---

## Varför är porten hårdkodad till 5432?

I Kubernetes injiceras miljövariabler automatiskt för alla services med samma namn som deploymentet. Det innebär att om man sätter `POSTGRES_PORT` som miljövariabel i cbhcloud, skriver Kubernetes över den med `tcp://10.x.x.x:5432` — ett format som Java inte förstår.

Lösningen är att **aldrig sätta `POSTGRES_PORT` som miljövariabel** och istället hårdkoda 5432 i koden. Det verkar konstigt men är nödvändigt i Kubernetes-miljöer med service-discovery.
