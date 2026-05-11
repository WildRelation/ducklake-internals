# Frontend — React och inloggningsflödet

Källkod: [ducklake-access-manager/src/main/resources/static/index.html](https://github.com/wildrelation/ducklake-access-manager/blob/main/src/main/resources/static/index.html)

## Struktur

Frontenden har fyra vyer:

| Vy | Vem ser den | Innehåll |
|---|---|---|
| **Browse** | Alla | Publika datasets + private man har access till. Sök, filtrera, generera nycklar. |
| **My Keys** | Alla | Aktiva nycklar. Admin ser alla med Created By-kolumn. |
| **Admin → Datasets** | Admin | Skapa/uppdatera/ta bort datasets (bucket + Postgres-DB skapas atomärt). |
| **Admin → Groups** | Admin | Skapa grupper, lägg till/ta bort enskilda medlemmar, massimportera via textarea. |
| **Admin → Grants** | Admin | Tilldela access: user (e-post), group (dropdown) eller @everyone. |

---

## Credentials-dialogen (CredModal)

När en student genererar en nyckel visas en modal med:

- **Råa credentials** — Key ID, Secret Key, PG-host, PG-databas, användarnamn, lösenord
- **DuckDB-script** — redo att klistras in i DuckDB/JupyterLab, med Copy-knapp
- **Download .env** — laddar ned en fil med standardiserade miljövariabelnamn:

```bash
# PostgreSQL — psycopg2, SQLAlchemy, libpq
PGHOST=ducklake-catalog
PGPORT=5432
PGDATABASE=dl_titanic_2026
PGUSER=dl_ro_xxxxxxxx
PGPASSWORD=...

# S3 — boto3, s3fs, DuckDB
AWS_ACCESS_KEY_ID=GKxxxxxxxx
AWS_SECRET_ACCESS_KEY=...
AWS_DEFAULT_REGION=garage
AWS_ENDPOINT_URL=http://ducklake-garage:3900
S3_BUCKET=titanic-2026
```

Filen namnges `.env.<bucket-namn>` och laddas ned direkt i webbläsaren utan att skicka credentials till servern igen.

> Secret Key visas bara i den här dialogen — Garage lagrar inte plaintext. Studenten måste kopiera eller ladda ned .env-filen innan dialogen stängs.

---

## Bulk member import (Admin → Groups)

När admin expanderar en grupp visas en textarea för att klistra in en lista av e-postadresser (newline, komma eller semikolon som separator). Resultatet visas direkt: `✓ 24 added · 3 already members · 1 invalid`.

---

## Hur frontenden är byggd

Frontenden är **en enda HTML-fil** utan byggprocess (inget npm, inget webpack). React och Babel laddas direkt från CDN via `<script>`-taggar. Det gör det enkelt att ändra — öppna filen, redigera, bygg Docker-imagen.

```html
<script src="https://unpkg.com/react@18/umd/react.development.js"></script>
<script src="https://unpkg.com/@babel/standalone/babel.min.js"></script>
<script type="text/babel">
  // JSX-kod här — Babel kompilerar det i webbläsaren
</script>
```

---

## Inloggningsflödet steg för steg

Se [keycloak.md](../komponenter/keycloak.md) för bakgrund om JWT och PKCE.

```javascript
// 1. Generera ett slumpmässigt code_verifier
const verifier = generateCodeVerifier();

// 2. Hasha det till code_challenge
const challenge = await generateCodeChallenge(verifier);

// 3. Spara verifier i sessionStorage (behövs i steg 6)
sessionStorage.setItem('pkce_verifier', verifier);

// 4. Skicka användaren till Keycloak med challenge
window.location.href = `${KEYCLOAK_BASE}/auth?
  client_id=ducklake&
  response_type=code&
  code_challenge=${challenge}&
  code_challenge_method=S256`;
```

När användaren kommer tillbaka från Keycloak:

```javascript
// 5. Keycloak returnerade en engångskod i URL:en
const code = new URLSearchParams(window.location.search).get('code');

// 6. Byt ut koden + verifier mot ett riktigt token
const response = await fetch(`${KEYCLOAK_BASE}/token`, {
  method: 'POST',
  body: new URLSearchParams({
    grant_type: 'authorization_code',
    code,
    code_verifier: sessionStorage.getItem('pkce_verifier'),
    client_id: 'ducklake',
  })
});

const { access_token } = await response.json();
sessionStorage.setItem('access_token', access_token);
```

---

## sessionStorage

Frontenden lagrar JWT access token i `sessionStorage` — det försvinner när fliken stängs så att credentials inte ligger kvar. `pgUsername` lagras inte längre i `localStorage` eftersom det nu bäddas in i nyckelnamnet på serversidan och returneras av `GET /api/keys`.

---

## Hur API-anrop görs

Alla anrop till backend-API:t skickar JWT-tokenet i headern:

```javascript
const authFetch = async (url, options = {}) => {
  return fetch(url, {
    ...options,
    headers: {
      ...options.headers,
      'Authorization': `Bearer ${token}`,
    },
  });
};
```

Om backend returnerar 401 (token expired) loggas användaren ut automatiskt.

---

## Göra ändringar i frontenden

1. Redigera `src/main/resources/static/index.html` i [ducklake-access-manager](https://github.com/wildrelation/ducklake-access-manager)
2. Bygg och pusha Docker-imagen (se [deployment.md](../drift/deployment.md))
3. Starta om deploymentet på cbhcloud

Det behövs ingen byggprocess — ändringarna syns direkt i den nya imagen.
