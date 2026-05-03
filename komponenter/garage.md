# Garage — Objektlagring

## Vad är Garage?

Garage är en **S3-kompatibel objektlagring** som vi kör själva på cbhcloud. Tänk på det som en självhostad version av Amazon S3.

**S3** är ett protokoll (API) för att lagra och hämta filer. Det fungerar ungefär som ett filsystem via HTTP:
- Filer kallas **objekt**
- Mappar kallas **buckets**
- Du laddar upp/ner filer med HTTP-anrop

DuckDB förstår S3-protokollet inbyggt, så det kan läsa Parquet-filer direkt från Garage utan att ladda ner dem först.

---

## Varför Garage och inte MinIO?

MinIO är ett annat populärt alternativ (se [ducklake-guide](https://github.com/wildrelation/ducklake-guide) som använder MinIO). Vi valde Garage för att det är lättare att köra på cbhcloud med begränsade resurser.

---

## Hur access managern pratar med Garage

Garage har ett **Admin API** (version 2) på port 3903. Access managern använder det för att:

1. Skapa en ny S3-nyckel (`POST /v2/CreateKey`)
2. Hitta bucket-ID från bucket-namn (`GET /v2/GetBucketInfo`)
3. Ge nyckeln rätt behörighet på bucketen (`POST /v2/AllowBucketKey`)
4. Ta bort en nyckel (`POST /v2/DeleteKey`)
5. Lista alla nycklar (`GET /v2/ListKeys`)

Alla anrop kräver en admin-token i `Authorization: Bearer`-headern.

```
Access Manager  →  POST /v2/CreateKey          →  Garage Admin API (:3903 via nginx :3900)
                →  GET  /v2/GetBucketInfo       →
                →  POST /v2/AllowBucketKey      →
```

> Se [arkitektur.md](../arkitektur.md) för varför nginx behövs mellan access manager och Garage.

---

## S3-nycklar vs admin-token

Det finns två typer av "nycklar" i Garage — det är lätt att blanda ihop dem:

| Typ | Vad det är | Används till |
|-----|-----------|--------------|
| **Admin-token** | En lång hemlighet i `garage.toml` | Access manager autentiserar mot Admin API |
| **S3-nyckel** | Key ID + Secret (som AWS access keys) | Studenter ansluter till S3 för att läsa data |

Admin-token sätts som miljövariabeln `GARAGE_ADMIN_TOKEN` i access manager-deploymentet.

---

## Nyckelnamnsformatet

När access managern skapar en S3-nyckel ger den den ett strukturerat namn:

```
key-{bucket}|{pg-användarnamn}
```

Exempel: `key-ducklake|dl_ro_a3f2b1c9`

Det här är inte ett tekniskt krav från Garage — vi valde det formatet för att kunna utläsa vilket PostgreSQL-konto som hör ihop med vilken S3-nyckel, utan att behöva lagra det separat.
