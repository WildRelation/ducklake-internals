# Nästa steg — DuckLake Dataset Hub

> **Förkrav:** Dataset Hub förutsätter att studenter kan nå `ducklake-catalog` och `ducklake-garage` direkt från sina egna deployments. Det är för tillfället inte möjligt på cbhcloud på grund av NetworkPolicy-begränsningar. Se [arkitektur.md](../arkitektur.md#nuvarande-begränsning--studenter-kan-inte-nå-ducklake-från-egna-deployments) för detaljer och planerad lösning.

En Kaggle-liknande plattform där studenter kan ladda upp egna datasets, välja om de ska vara publika eller privata, och se varandras publika datasets.

---

## Arkitektur

```
┌─────────────────────────────────────────┐
│         ducklake-dataset-hub            │  Nytt deployment på cbhcloud
│                                         │
│  Frontend:  Dataset-browser (React)     │
│  Backend:   Spring Boot API             │
│             - Upload dataset            │
│             - Lista mina datasets       │
│             - Gör publik/privat         │
│             - Lista alla publika        │
└──────────┬──────────────────┬───────────┘
           │                  │
    ┌──────▼──────┐    ┌──────▼──────┐
    │  PostgreSQL │    │   Garage    │
    │  datasets-  │    │   S3        │
    │  tabell     │    │             │
    └─────────────┘    └─────────────┘
```

Autentisering återanvänds från `ducklake-access-manager` — samma Keycloak-klient och JWT-flöde.

---

## Databas — ny tabell

```sql
CREATE TABLE datasets (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    owner_sub   VARCHAR NOT NULL,      -- Keycloak sub
    owner_name  VARCHAR NOT NULL,      -- jpsa2@kth.se
    name        VARCHAR NOT NULL,      -- "Titanic survivors"
    description TEXT,
    s3_key      VARCHAR NOT NULL,      -- students/jpsa2/titanic.parquet
    size_bytes  BIGINT,
    is_public   BOOLEAN DEFAULT false,
    created_at  TIMESTAMP DEFAULT NOW()
);
```

---

## S3-struktur

En separat bucket `ducklake-datasets`:

```
s3://ducklake-datasets/
  students/
    jpsa2/
      titanic.parquet
    ana123/
      iris.parquet
```

Åtkomst styrs av backend-API:t — inte S3-behörigheter. Backend använder ett admin-konto för all S3-åtkomst och kontrollerar i koden vem som får se vad.

---

## API-endpoints

| Endpoint | Beskrivning |
|----------|-------------|
| `POST /api/datasets/upload` | Ladda upp dataset (multipart) |
| `GET /api/datasets/mine` | Lista mina datasets |
| `PATCH /api/datasets/{id}/visibility` | Gör publik eller privat |
| `GET /api/datasets/public` | Lista alla publika datasets |
| `GET /api/datasets/{id}/download` | Hämta fil (presigned URL) |
| `DELETE /api/datasets/{id}` | Radera dataset |

---

## Frontend — tre sidor

```
/            →  Publika datasets (startsidan, Kaggle-liknande)
/mine        →  Mina datasets + upload-knapp
/dataset/id  →  Dataset-detaljsida (namn, beskrivning, storlek, download)
```

---

## Totalt antal deployments

När Dataset Hub är byggt kommer infrastrukturen bestå av 4 deployments:

| Deployment | Vad |
|------------|-----|
| `ducklake-catalog` | PostgreSQL — DuckLake-katalog + ägarskapsregister |
| `ducklake-garage` | Garage S3 — lagrar Parquet-filer |
| `ducklake-access-manager` | Credential-tjänst — genererar nycklar till studenter |
| `ducklake-dataset-hub` | Kaggle-liknande plattform för att dela datasets |

Keycloak räknas inte — det är KTH:s befintliga tjänst som vi ansluter till.

---

## Vad kan återanvändas från ducklake-access-manager?

| Del | Kan återanvändas |
|-----|-----------------|
| Spring Boot + Maven-setup | ✅ |
| Spring Security + OAuth2/JWT | ✅ |
| Garage S3-integration | ✅ (delvis — ny bucket, admin-nyckel) |
| React + PKCE-inloggningsflöde | ✅ |
| Docker + cbhcloud-deployment | ✅ |
| Filuppladdning (multipart) | ❌ Nytt |
| Dataset-metadata (databas) | ❌ Nytt |
| Publik/privat-toggle | ❌ Nytt |
| Dataset-browser UI | ❌ Nytt |
