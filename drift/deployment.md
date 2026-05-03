# Deployment

## Bygga och pusha en ny version

Källkod: [ducklake-access-manager](https://github.com/wildrelation/ducklake-access-manager)

```bash
cd ducklake-access-manager

# Bygg Docker-imagen (kompilerar Java-koden inuti containern)
docker build --network=host -t ghcr.io/wildrelation/ducklake-access-manager:latest .

# Pusha till GitHub Container Registry
docker push ghcr.io/wildrelation/ducklake-access-manager:latest
```

> `--network=host` krävs på Linux om Docker inte kan skapa bridge-nätverk i din miljö.

Sedan startar om deploymentet på cbhcloud så att det hämtar den nya imagen.

---

## Miljövariabler i cbhcloud-deploymentet

| Variabel | Värde | Kommentar |
|----------|-------|-----------|
| `POSTGRES_HOST` | `ducklake-catalog` | Internt Kubernetes-servicename |
| `POSTGRES_DB` | `ducklake` | |
| `POSTGRES_ADMIN_USER` | `ducklake` | |
| `POSTGRES_ADMIN_PASSWORD` | `cbhcloud` | |
| `POSTGRES_PUBLIC_HOST` | publikt hostname | Visas i DuckDB-scriptet för studenter |
| `GARAGE_ADMIN_URL` | `http://ducklake-garage:3900` | Port 3900 (nginx), inte 3903 |
| `GARAGE_ADMIN_TOKEN` | token från garage.toml | Se nedan hur du hittar den |
| `GARAGE_S3_ENDPOINT` | `ducklake-garage:3900` | Utan protokoll, utan snedstreck |
| `GARAGE_S3_REGION` | `garage` | Måste matcha `s3_region` i garage.toml |
| `KEYCLOAK_ISSUER_URI` | `https://iam.cloud.cbh.kth.se/realms/cloud` | Standard, behöver sällan ändras |
| `PORT` | `8080` | |

> Sätt **aldrig** `POSTGRES_PORT` — Kubernetes skriver över den med ett TCP-format som Java inte förstår. Porten är hårdkodad till 5432 i koden.

### Hitta Garage Admin Token

SSH:a in i Garage-deploymentet och hämta token:

```bash
ssh ducklake-garage@deploy.cloud.cbh.kth.se
cat /tmp/garage.toml | grep admin_token
```

Använd alltid `/tmp/garage.toml`, inte `/etc/garage.toml` — det är den faktiska config som körs.

---

## Testa lokalt med SSH-tunnlar

```bash
# Terminal 1 — PostgreSQL-tunnel
ssh -L 5433:localhost:5432 ducklake-catalog@deploy.cloud.cbh.kth.se

# Terminal 2 — Garage Admin API-tunnel
ssh -L 3900:localhost:3900 ducklake-garage@deploy.cloud.cbh.kth.se

# Terminal 3 — starta access managern
cp .env.example .env   # fyll i värden
export $(cat .env | grep -v '^#' | grep -v '^$' | xargs)
mvn spring-boot:run
```

Öppna sedan `http://localhost:8080` i webbläsaren.
