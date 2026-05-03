# Ordlista

Enkla förklaringar av begrepp som används i den här dokumentationen.

---

## AWS (Amazon Web Services)
Amazons molnplattform. AWS skapade och populariserade många standardprotokoll som nu används allmänt — bland annat **S3** för objektlagring. När vi säger "S3-kompatibelt" menar vi att tjänsten (i vårt fall Garage) förstår samma API som Amazon S3, men vi använder inte Amazon — vi kör allt på cbhcloud.

---

## S3
Ett **protokoll** (ett sätt att kommunicera) för att lagra och hämta filer via HTTP. Tänk på det som ett filsystem på internet:
- Du skickar en fil → den lagras
- Du ber om en fil → du får tillbaka den

DuckDB, Python och de flesta programmeringsspråk förstår S3 inbyggt. Garage implementerar S3-protokollet, så allt som fungerar med Amazon S3 fungerar också med Garage.

---

## S3-nyckel
En S3-nyckel är ett **inloggningspar** för S3 — samma koncept som ett användarnamn och lösenord, fast det kallas Key ID och Secret:

```
Key ID:  GK3096272c0da3b9dc7010f806   ← "användarnamnet" (kan delas)
Secret:  xxxxxxxxxxxxxxxxxxxxxxxxxxx  ← "lösenordet" (hemlighet, visas bara en gång)
```

Varje student får sin egen S3-nyckel med begränsad åtkomst — de kan bara läsa från en specifik bucket, inte skriva eller se andra buckets.

---

## Bucket
En bucket är en **toppnivåmapp** i S3/Garage. Alla filer måste ligga i en bucket. Tänk på det som en hårddisk:

```
Garage
  └── ducklake/          ← bucket (heter "ducklake")
        ├── tabell1.parquet
        └── tabell2.parquet
```

En nyckel kan ha behörighet på en specifik bucket — att ha en nyckel till `ducklake`-bucketen ger inte åtkomst till andra buckets.

---

## PostgreSQL
En **relationsdatabas** — lagrar data i tabeller med rader och kolumner, precis som Excel fast kraftfullare. I vårt system används PostgreSQL inte för att lagra kursdata, utan som **katalog** åt DuckLake: den håller koll på vilka tabeller som finns och var filerna ligger i Garage.

---

## Token
En token är en **tillfällig nyckel** som bevisar vem du är. Tänk på det som en biljett:
- Du visar legitimation hos Keycloak (loggar in med KTH-konto)
- Keycloak ger dig en biljett (JWT-token)
- Du visar biljetten varje gång du vill använda access managern
- Biljetten slutar gälla efter en stund (du behöver logga in igen)

Det finns också en **admin-token** till Garage — det är en permanent hemlighet som access managern använder för att administrera S3-nycklar. Dessa är olika saker trots att de heter samma.

---

## Bearer
`Bearer` är ett ord i HTTP-autentisering som betyder *"den som bär det här tokenet"*. När man skickar en förfrågan till ett API med ett JWT-token ser det ut så här:

```
Authorization: Bearer eyJhbGciOi...
```

Det betyder: *"Jag är den person som bär detta token — se vad det säger om mig."* Servern läser tokenet och vet vem du är utan att behöva fråga Keycloak.

---

## Port
En port är ett **nummer som identifierar en specifik tjänst** på en dator. En dator har en IP-adress (t.ex. `10.0.0.1`), men kan köra många tjänster samtidigt — port-numret säger vilken:

```
10.0.0.1:5432  →  PostgreSQL
10.0.0.1:3900  →  Garage (nginx/S3)
10.0.0.1:3903  →  Garage Admin API (intern)
10.0.0.1:8080  →  Access Manager
```

Tänk på det som ett hyreshus: IP-adressen är adressen till huset, port-numret är lägenhetsnumret.

---

## UUID
UUID (Universally Unique Identifier) är ett **slumpmässigt genererat ID** som med extrem sannolikhet är unikt i hela världen:

```
550e8400-e29b-41d4-a716-446655440000
```

Vi använder UUID för att generera lösenord och användarnamns-suffix för student-konton. Eftersom det är slumpmässigt och nästan omöjligt att gissa är det säkert att använda som lösenord.

---

## Realm (Keycloak)
En realm i Keycloak är ett **isolerat område** med egna användare, klienter och roller. Tänk på det som ett separat företag:

```
Keycloak
  └── realm: cloud        ← cbhclouds realm (vi använder den här)
        ├── users: alla KTH-studenter och lärare
        └── clients: ducklake, jupyterhub, m.fl.
```

cbhcloud har en realm som heter `cloud`. Inom den finns vår klient `ducklake`. Det är viktigt att admin-rollen är en **klientroll** (bara för `ducklake`) och inte en **realm-roll** (som skulle ge admin-åtkomst till hela cbhcloud).

---

## JWT (JSON Web Token)
Förklaras utförligt i [komponenter/keycloak.md](komponenter/keycloak.md). Kortfattat: ett signerat dokument som bevisar vem du är och vilka roller du har, utan att servern behöver fråga Keycloak för varje anrop.
