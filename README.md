# DuckLake Internals

Dokumentation för oss som bygger och underhåller DuckLake-infrastrukturen på cbhcloud.

> Letar du efter studentguiden? Den finns i [ducklake-guide](https://github.com/wildrelation/ducklake-guide).

---

## Vad är det här repot?

Det här repot förklarar hur hela infrastrukturen fungerar — varje komponent, varför vi valde den, hur delarna pratar med varandra och hur koden i access managern är uppbyggd. Målet är att vem som helst i gruppen ska kunna sätta sig in i systemet och göra ändringar.

---

## Innehåll

### 0. Ordlista
**[ordlista.md](ordlista.md)** — Förklaringar av begrepp: AWS, S3, S3-nyckel, bucket, token, Bearer, port, UUID, realm, m.m.

### 1. Arkitektur
**[arkitektur.md](arkitektur.md)** — Helhetsbilden. Hur alla delar hänger ihop.

### 2. Komponenter
- **[komponenter/garage.md](komponenter/garage.md)** — Vad är Garage och hur fungerar S3?
- **[komponenter/postgres.md](komponenter/postgres.md)** — PostgreSQL som DuckLake-katalog
- **[komponenter/keycloak.md](komponenter/keycloak.md)** — Autentisering med Keycloak och JWT

### 3. Access Manager
- **[access-manager/overview.md](access-manager/overview.md)** — Vad access managern gör, steg för steg
- **[access-manager/backend.md](access-manager/backend.md)** — Java-koden förklarad
- **[access-manager/frontend.md](access-manager/frontend.md)** — React-frontenden och inloggningsflödet

### 4. Drift
- **[drift/deployment.md](drift/deployment.md)** — Bygga och driftsätta
- **[drift/felsökning.md](drift/felsökning.md)** — Kända problem och lösningar

### 5. Nästa steg
- **[naasta-steg/dataset-hub.md](naasta-steg/dataset-hub.md)** — Plan för DuckLake Dataset Hub: Kaggle-liknande plattform där studenter laddar upp och delar datasets

---

## Relaterade repon

| Repo | Beskrivning |
|------|-------------|
| [ducklake-access-manager](https://github.com/wildrelation/ducklake-access-manager) | Källkoden för access managern |
| [ducklake-guide](https://github.com/wildrelation/ducklake-guide) | Studentguide — hur man ansluter och kör queries |
| [garage-cbhcloud-quickstart](https://github.com/wildrelation/garage-cbhcloud-quickstart) | Hur Garage är uppsatt på cbhcloud |
