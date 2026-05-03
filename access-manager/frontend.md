# Frontend — React och inloggningsflödet

Källkod: [ducklake-access-manager/src/main/resources/static/index.html](https://github.com/wildrelation/ducklake-access-manager/blob/main/src/main/resources/static/index.html)

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

## localStorage vs sessionStorage

Frontenden använder **båda** för olika saker:

| Vad | Var | Varför |
|-----|-----|--------|
| JWT access token | `sessionStorage` | Försvinner när fliken stängs — credentials ska inte ligga kvar |
| Mapping keyId → pgUsername | `localStorage` | Behålls mellan sessioner — används om servern inte har pgUsername |

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
