# Keycloak — Autentisering

## Vad är Keycloak?

Keycloak är KTH:s **identitetsleverantör** — det är den tjänst som hanterar inloggning på cbhcloud. När studenter loggar in med sitt KTH-konto är det Keycloak som verifierar identiteten.

Vi behöver inte sätta upp Keycloak själva — vi använder den instans som redan finns på `iam.cloud.cbh.kth.se`.

---

## JWT-token — vad är det?

När en användare loggar in returnerar Keycloak ett **JWT** (JSON Web Token). Det är en lång sträng i tre delar separerade med punkter:

```
xxxxx.yyyyy.zzzzz
  │      │      │
header  data  signatur
```

Mittendelen är Base64-kodad JSON som innehåller information om användaren:

```json
{
  "sub": "a3f2b1c9-...",          ← användarens unika Keycloak-ID (ändras aldrig)
  "email": "jpsa2@kth.se",
  "preferred_username": "jpsa2",
  "resource_access": {
    "ducklake": {
      "roles": ["admin"]          ← om användaren är admin för ducklake-klienten
    }
  },
  "exp": 1234567890               ← när tokenet slutar gälla
}
```

**Signaturen** gör att vem som helst kan verifiera att tokenet är äkta utan att fråga Keycloak — access managern laddar ner Keycloaks publika nyckel en gång och verifierar sedan alla tokens lokalt.

---

## Klientroller vs realm-roller

I Keycloak finns det två typer av roller:

| Typ | Var | Vad |
|-----|-----|-----|
| **Realm-roll** | `realm_access.roles` | Gäller för hela Keycloak-realmen |
| **Klientroll** | `resource_access.{klient}.roles` | Gäller bara för en specifik klient |

Vi använder **klientroller** för admin-checken. Det innebär att man kan vara admin för `ducklake` utan att vara global admin för hela cbhcloud-realmen. Det ger mer kontroll.

För att göra någon till admin i access managern:
1. Gå till `iam.cloud.cbh.kth.se` → Keycloak Admin Console
2. Välj realm `cloud`
3. Gå till Clients → `ducklake` → Roles
4. Lägg till rollen `admin` på användarens konto

---

## PKCE-flödet — hur frontenden loggar in

Frontenden är en **public client** — den har inget hemligt lösenord. Istället används **PKCE** (Proof Key for Code Exchange) för att säkra inloggningsflödet:

```
1. Webbläsaren genererar ett slumpmässigt värde (code_verifier)
2. Webbläsaren hashar det (code_challenge = SHA256(code_verifier))
3. Webbläsaren skickar code_challenge till Keycloak + ber om inloggning
4. Användaren loggar in på Keycloak
5. Keycloak skickar tillbaka en engångskod (code) till webbläsaren
6. Webbläsaren skickar code + code_verifier till Keycloak
7. Keycloak verifierar: SHA256(code_verifier) == code_challenge?
8. Om ja → Keycloak returnerar ett access token
```

Det smarta med PKCE är att även om någon avlyssnar steg 5 och stjäl `code`, kan de inte använda den utan `code_verifier` som bara webbläsaren känner till.

---

## Vart lagras tokenet?

Tokenet lagras i `sessionStorage` i webbläsaren — det försvinner automatiskt när fliken stängs. Det är ett medvetet val: credentials ska inte ligga kvar om användaren lämnar datorn.
