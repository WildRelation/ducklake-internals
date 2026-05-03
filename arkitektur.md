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
