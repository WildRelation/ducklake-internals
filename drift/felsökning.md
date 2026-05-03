# Felsökning

## S3-signaturen matchar inte (403 från Garage)

**Symptom:** `AccessDenied: Invalid signature` trots rätt Key ID och Secret.

**Orsak:** nginx vidarebefordrar bara hostname utan port i `Host`-headern. S3-signaturen inkluderar `Host`-headern — klienten signerar med `host:3900` men Garage tar emot bara `host` → signaturen matchar inte.

**Lösning:** nginx.conf ska ha `proxy_set_header Host $http_host` (inte `$host`). Dokumenterat i [garage-cbhcloud-quickstart](https://github.com/wildrelation/garage-cbhcloud-quickstart).

---

## DuckDB-scriptet fungerar inte (S3-fel)

**Symptom:** `AuthorizationHeaderMalformed` eller liknande när studenten kör scriptet.

**Troliga orsaker och lösningar:**

| Fel | Lösning |
|-----|---------|
| `REGION 'local'` i scriptet | Sätt `GARAGE_S3_REGION=garage` i access manager-deploymentet |
| `ENDPOINT 'https://...'` med protokoll | Sätt `GARAGE_S3_ENDPOINT=ducklake-garage:3900` (utan protokoll) |
| Studenten kör scriptet lokalt | Scriptet måste köras från ett deployment **på cbhcloud** |

---

## `null value in column "keycloak_sub"` vid uppstart

**Symptom:** Access managern kraschar med constraint-fel direkt efter en uppdatering.

**Orsak:** `key_user_mapping`-tabellen skapades i en äldre version med ett annat kolumnnamn. `CREATE TABLE IF NOT EXISTS` skapar inte om tabellen om den redan finns.

**Lösning:**
```sql
DROP TABLE key_user_mapping;
```
Starta sedan om access manager-deploymentet — tabellen återskapas automatiskt.

---

## Rensa alla student-credentials (PostgreSQL + Garage)

Se till att du har SSH-access till deploymentens. Kör i rätt ordning:

**Steg 1 — Radera PostgreSQL-användare:**
```sql
-- SSH:a in i ducklake-catalog och kör psql -U ducklake
DO $$
DECLARE r RECORD;
BEGIN
  FOR r IN SELECT usename FROM pg_user WHERE usename LIKE 'dl_%' LOOP
    EXECUTE 'REVOKE ALL PRIVILEGES ON ALL TABLES IN SCHEMA public FROM ' || quote_ident(r.usename);
    EXECUTE 'REVOKE ALL ON SCHEMA public FROM ' || quote_ident(r.usename);
    EXECUTE 'REVOKE ALL ON DATABASE ducklake FROM ' || quote_ident(r.usename);
    EXECUTE 'DROP USER ' || quote_ident(r.usename);
  END LOOP;
END $$;

TRUNCATE key_user_mapping;
```

**Steg 2 — Radera Garage-nycklar:**
```bash
# SSH:a in i access manager-containern (eller kör varifrån GARAGE_ADMIN_TOKEN är tillgänglig)
wget -q -O - --header "Authorization: Bearer $GARAGE_ADMIN_TOKEN" $GARAGE_ADMIN_URL/v2/ListKeys \
  | grep -o '"id": "[^"]*"' | grep -o 'GK[^"]*' \
  | grep -v "GK6759c84613351dac5b8367db" \   # ← behåll systemnyckel ducklake-key
  | while read id; do
      wget -q -O - --header "Authorization: Bearer $GARAGE_ADMIN_TOKEN" \
        --post-data "" "$GARAGE_ADMIN_URL/v2/DeleteKey?id=$id"
      echo "Deleted $id"
    done
```

> `curl` finns inte i Alpine-containern — använd `wget`.

---

## Kontrollera vilka nycklar som finns

**PostgreSQL — lista alla student-användare:**
```sql
SELECT usename FROM pg_user WHERE usename LIKE 'dl_%';
```

**PostgreSQL — lista autentiserade nycklar med ägare:**
```sql
SELECT garage_key_id, display_name, created_at FROM key_user_mapping ORDER BY created_at;
```

**Garage — lista alla S3-nycklar:**
```bash
wget -q -O - --header "Authorization: Bearer $GARAGE_ADMIN_TOKEN" $GARAGE_ADMIN_URL/v2/ListKeys
```
