# Arkitektur

## Helhetsbilden

DuckLake är ett **data lake** — ett centralt ställe där kursdatasets lagras. Studenter ansluter till det från sina egna deployments på cbhcloud och kör queries med DuckDB.

```
                        Internet
                           │
                    ┌──────▼──────┐
                    │  Keycloak   │  KTH:s identitetsleverantör
                    │  (IAM)      │  iam.cloud.cbh.kth.se
                    └──────┬──────┘
                           │ JWT-token
                           │
              ┌────────────▼────────────┐
              │   ducklake-access-      │  Webbgränssnitt + REST API
              │   manager               │  Skapar S3-nycklar och PG-användare
              │   :8080                 │  github: ducklake-access-manager
              └──────┬──────────┬───────┘
                     │          │
          ┌──────────▼──┐  ┌────▼──────────┐
          │  ducklake-  │  │  ducklake-    │
          │  catalog    │  │  garage       │
          │  PostgreSQL │  │  Garage S3    │
          │  :5432      │  │  :3900        │
          └──────┬──────┘  └──────┬────────┘
                 │                │
         ┌───────▼────────────────▼───────┐
         │     Studentens deployment      │
         │     (JupyterLab, Python m.m.)  │
         │     Kör DuckDB härifrån        │
         └────────────────────────────────┘
```

---

## Flödet — vad händer när en student genererar en nyckel?

```
1. Student besöker access manager UI
       ↓
2. Webbläsaren skickar studenten till Keycloak (KTH-inloggning)
       ↓
3. Keycloak returnerar ett JWT-token till webbläsaren
       ↓
4. Studenten klickar "Generate Key" — webbläsaren skickar token + begäran till access manager
       ↓
5. Access manager skapar en PostgreSQL-användare (dl_ro_xxxxxxxx)
       ↓
6. Access manager skapar en S3-nyckel i Garage med rätt behörighet på bucketen
       ↓
7. Access manager sparar (garage_key_id, keycloak_sub, email) i key_user_mapping
       ↓
8. Access manager returnerar ett färdigt DuckDB-script till studenten
       ↓
9. Studenten kör scriptet från sitt deployment på cbhcloud
```

---

## Varför är varje del nödvändig?

| Komponent | Varför den finns |
|-----------|-----------------|
| **Garage** | Lagrar de faktiska datafilerna (Parquet). S3-kompatibelt, kör på cbhcloud. |
| **PostgreSQL** | Lagrar metadata — vilka tabeller finns, var är filerna, versionshistorik. DuckLake-formatet kräver en katalogdatabas. |
| **Access Manager** | Utan den måste vi skapa PostgreSQL-användare och S3-nycklar manuellt för varje student. Med den sker det automatiskt. |
| **Keycloak** | Vi vill veta vem som genererar nycklar. KTH:s Keycloak ger oss KTH-identiteter gratis. |

---

## Nätverksbegränsningar på cbhcloud

cbhcloud (Kubernetes) tillåter bara trafik mellan deployments på den port som är deklarerad i `PORT`-variabeln. Det skapar ett problem:

- Garage Admin API körs på port **3903** (intern)
- Access manager kan bara nå Garage på port **3900** (den deklarerade porten)

**Lösning:** En nginx reverse proxy i Garage-containern tar emot all trafik på port 3900 och vidarebefordrar:
- `/v2/*` → port 3903 (Admin API)
- allt annat → port 3905 (S3 API, omflyttad från 3900)

Se [garage-cbhcloud-quickstart](https://github.com/wildrelation/garage-cbhcloud-quickstart) för hur Garage-deploymentet är konfigurerat.

---

## Nuvarande begränsning — studenter kan inte nå DuckLake från egna deployments

**Status (april 2026):** cbhcloud har en NetworkPolicy som fungerar som en ACL — ett deployment kan bara nå sina **egna** deployments, inte andras. Det innebär att en students deployment inte kan nå `ducklake-catalog` (PostgreSQL) eller `ducklake-garage` (S3) direkt, eftersom de ägs av oss.

Det här är anledningen till att DuckDB-scriptet för tillfället kräver att studenten kör det från ett deployment som tillhör **oss**, inte deras egna konton.

**Planerad lösning från cbhcloud-supporten:** PostgreSQL ska sättas upp som en **systemtjänst** via ett Helm chart — en speciell typ av deployment som är nåbar från alla användares deployments, inte bara ägarens. Det kommer troligen inte att ske via molnets gränssnitt utan kräver att cbhcloud-teamet gör det på infrastrukturnivå.

> Källa: cbhcloud-supporten (philipzi), 2026-04-29. Citat: *"Min tanke är att vi sätter upp en postgres som inte kräver en SSH tunnel, då kommer även den vara nåbar från andra deployments. Men detta kommer troligen sättas upp som en system tjänst isf via typ en helm chart."*

När detta är på plats kan studenter köra DuckDB-scriptet direkt från sina egna JupyterLab- eller Python-deployments utan SSH-tunnlar eller mellanhänder.
