# OAuth 2.0 & OpenID Connect: Technical Interview Deep-Dive

> **Scope:** the problem OAuth solves, roles and vocabulary, the Authorization Code + PKCE flow step by step (with raw HTTP), why each parameter exists (`state`, PKCE, `nonce`, `redirect_uri`, `iss`), client credentials, device flow, refresh-token rotation with reuse detection, deprecated grants and why, OpenID Connect on top, SPA architectures (BFF vs. tokens in the browser), resource-server validation (JWT vs. introspection), the attack catalogue, and how you'd **build and scale an authorization server** in a system design round.
>
> **Stack used:** Node.js 18+ (global `fetch`, `node:crypto`), Express, [`jose`](https://www.npmjs.com/package/jose) for JWT/JWKS, `redis` (node-redis v4+). In production you'd normally use a certified library ([`openid-client`](https://www.npmjs.com/package/openid-client) on the client side, a product like Keycloak / Auth0 / Okta / Cognito / Ory Hydra as the authorization server). The hand-written code here is there so you can explain **what those libraries do for you**.
>
> Related: [13 – Node.js authentication](13-nodejs-authentication-interview-guide.md) (sessions, JWT basics, cookies, CSRF, XSS) · [12 – Redis deep-dive](12-redis-nodejs-deep-dive.md) (single-flight, locks, TTL keys) · [11 – Order idempotency](11-order-concurrency-idempotency.md) (conditional `UPDATE` pattern reused for refresh-token rotation)

---

## 0. TL;DR (the 60-second interview answer)

1. **OAuth 2.0 is a delegated *authorization* framework**: a user (resource owner) lets an application (client) access their data on an API (resource server) **without giving the app their password**. The authorization server issues the client a **scoped, short-lived access token**.
2. **OAuth is not authentication.** An access token says "this client may do X", not "this user is logged in here". **OpenID Connect (OIDC)** adds authentication on top: an **ID token** (a signed JWT for the client, about the user), `nonce`, `/userinfo` and discovery.
3. The default flow for every user-facing app (web, SPA, mobile) is **Authorization Code + PKCE**. The browser carries only a **short-lived, single-use code** through the front channel. The **token exchange happens on the back channel** (direct HTTPS POST), with client authentication and/or the PKCE verifier.
4. Each parameter defends against a specific attack: **`state`** → CSRF on the callback. **PKCE** → stolen/injected authorization codes. **`nonce`** → ID-token replay. **Exact `redirect_uri` matching** → code/token theft via open redirects. **`iss` in the response** → mix-up attacks between multiple authorization servers.
5. **Implicit** and **Resource Owner Password Credentials** grants are **deprecated** (OAuth 2.0 Security BCP, RFC 9700, and the OAuth 2.1 draft). Machine-to-machine uses **Client Credentials**. Input-constrained devices (TVs, CLIs) use the **Device Authorization Grant**.
6. **Refresh tokens** keep sessions alive without long-lived access tokens. For public clients they must be **rotated on every use with reuse detection** (or sender-constrained with DPoP/mTLS).
7. **Resource servers** validate **self-contained JWT access tokens** locally (signature via JWKS, `iss`, `aud`, `exp`, scopes), which is fast but hard to revoke, or call **token introspection** for **opaque tokens**, which is revocable instantly but adds a network hop. Scopes limit what the *client* may do. You still check what the *user* may access.
8. For browser apps the current recommendation is a **Backend-for-Frontend (BFF)**: the server is the OAuth client, tokens never reach JavaScript, and the browser holds only an `HttpOnly` session cookie.

---

## 1. The problem OAuth solves

Before OAuth, if "PhotoPrint" wanted to read your Google Photos, it would ask for **your Google password**. That is bad for every party:

| Problem with password sharing | How OAuth fixes it |
|---|---|
| The app gets **full** account access | Access token is limited by **scope** (`photos.read`) |
| The only revocation is changing your password, which breaks every app | Revoke **one client's grant** without touching others |
| The app stores your password (and can leak it) | The app never sees credentials, only tokens |
| No MFA, no consent screen | The user authenticates **at the authorization server**, which owns MFA, risk checks and consent |
| No expiry | Access tokens are short-lived (minutes) |

### 1.1 The four roles

```mermaid
flowchart LR
    RO["Resource Owner<br/>(the user)"]
    C["Client<br/>(the app wanting access)"]
    AS["Authorization Server<br/>(issues tokens: Google, Okta, Keycloak)"]
    RS["Resource Server<br/>(the API holding the data)"]

    RO -- "1. authenticates and consents" --> AS
    AS -- "2. issues access token" --> C
    C -- "3. calls API with<br/>Authorization: Bearer token" --> RS
    RS -. "4. trusts tokens signed by AS<br/>(JWKS or introspection)" .-> AS
```

- **Resource owner:** the entity that can grant access, usually an end user.
- **Client:** the application. It's called "client" even if it is itself a server.
- **Authorization server (AS):** authenticates the user, gets consent and issues tokens. In OIDC it's called the **OpenID Provider (OP)**.
- **Resource server (RS):** the API. It accepts and validates access tokens.

The AS and RS can be the same deployment (GitHub's API and GitHub's OAuth server), but they're separate **logical** roles. In a microservice architecture you typically have **one AS and many RSs**.

### 1.2 Delegation vs. authentication (the classic trap)

> "Log in with Google" is **OIDC**, not plain OAuth.

If you use a plain OAuth access token as proof of login ("I got a token, so the user is X"), you're vulnerable to **token substitution**. An access token issued to *another* app (one the attacker controls) for the same user would also "log in" to yours, because access tokens aren't bound to your client as their audience. The ID token **is**: its `aud` is your `client_id`, and its `nonce` is bound to your login request. (Section 9.)

---

## 2. Vocabulary you must be fluent in

| Term | Meaning |
|---|---|
| **Client ID** | Public identifier of the app, registered at the AS. |
| **Confidential client** | Can keep a secret: a server-side web app or backend. Authenticates to the token endpoint. |
| **Public client** | Can't keep a secret: SPA, mobile app, CLI. Any embedded "secret" is extractable. Relies on **PKCE** instead. |
| **Scope** | A space-separated list of requested permissions, e.g. `openid profile orders:read`. The AS may grant fewer than requested. |
| **Grant type** | The method of obtaining a token: `authorization_code`, `client_credentials`, `refresh_token`, `urn:ietf:params:oauth:grant-type:device_code`, … |
| **Front channel** | Communication via the **browser** (redirects, URL query/fragment). Visible to the user, browser history, extensions, logs and `Referer`. |
| **Back channel** | **Direct server-to-server** HTTPS between the client and AS. Not visible to the browser. |
| **Authorization endpoint** | `GET /authorize`, front channel. The user logs in and consents here. |
| **Token endpoint** | `POST /token`, back channel. Exchanges a grant for tokens. |
| **Redirect URI** | Where the AS sends the browser back. Must be **pre-registered and matched exactly**. |
| **Access token** | A credential the client presents to the RS. A **bearer** token unless sender-constrained. |
| **Refresh token** | A long-lived credential used only at the token endpoint to get new access tokens. Never sent to the RS. |
| **ID token** | OIDC only. A JWT **for the client**, describing the authentication event. |
| **JWKS** | JSON Web Key Set: the AS's public keys (`/.well-known/jwks.json`), used to verify JWT signatures. |
| **Discovery / metadata** | `/.well-known/openid-configuration` (OIDC) or `/.well-known/oauth-authorization-server` (RFC 8414). Lists endpoints, supported algorithms, etc. |

### 2.1 The RFC map (name-drop with confidence)

| Spec | What it is |
|---|---|
| RFC 6749 | OAuth 2.0 core framework |
| RFC 6750 | Bearer token usage (`Authorization: Bearer`, `WWW-Authenticate` errors) |
| RFC 7636 | **PKCE** |
| RFC 7662 | Token **introspection** |
| RFC 7009 | Token **revocation** |
| RFC 8414 | AS metadata (discovery) |
| RFC 8628 | **Device** Authorization Grant |
| RFC 8693 | **Token Exchange** (on-behalf-of, delegation chains) |
| RFC 8705 | **mTLS**-bound tokens |
| RFC 9068 | **JWT profile for access tokens** (`typ: at+jwt`) |
| RFC 9126 | **PAR**: Pushed Authorization Requests |
| RFC 9207 | `iss` parameter in the authorization response (**mix-up** defence) |
| RFC 9449 | **DPoP**: proof-of-possession at the application layer |
| RFC 9700 | **OAuth 2.0 Security Best Current Practice** (2025) |
| OAuth 2.1 (draft) | Consolidation: PKCE mandatory, implicit/ROPC removed, exact redirect matching, refresh tokens sender-constrained or rotated for public clients |
| OpenID Connect Core 1.0 | Authentication layer (ID token, `nonce`, `/userinfo`) |

---

## 3. Authorization Code + PKCE: the flow you must be able to draw

```mermaid
sequenceDiagram
    autonumber
    actor U as User (browser)
    participant C as Client (app backend)
    participant AS as Authorization Server
    participant RS as Resource Server (API)

    U->>C: GET /login
    Note over C: generate state, nonce, code_verifier<br/>code_challenge = BASE64URL(SHA256(verifier))<br/>store all three in the user's session
    C-->>U: 302 to AS /authorize?response_type=code<br/>&client_id&redirect_uri&scope&state<br/>&nonce&code_challenge&code_challenge_method=S256
    U->>AS: GET /authorize?... (front channel)
    AS->>U: login page (+ MFA) and consent screen
    U->>AS: credentials + "Allow"
    Note over AS: store code -> {client_id, redirect_uri,<br/>user, scope, code_challenge, nonce}<br/>TTL about 60s, single use
    AS-->>U: 302 to redirect_uri?code=XYZ&state=...&iss=...
    U->>C: GET /callback?code=XYZ&state=...&iss=...
    Note over C: check state matches the session<br/>check iss is the expected AS
    C->>AS: POST /token (back channel)<br/>grant_type=authorization_code, code,<br/>redirect_uri, code_verifier + client auth
    Note over AS: verify code unused and unexpired,<br/>same client and redirect_uri,<br/>SHA256(verifier) equals stored challenge
    AS-->>C: 200 {access_token, id_token, refresh_token, expires_in}
    Note over C: validate id_token (sig, iss, aud, exp, nonce)<br/>regenerate session, store tokens server-side
    C-->>U: 302 to app, Set-Cookie: session (HttpOnly)
    C->>RS: GET /orders, Authorization: Bearer access_token
    RS-->>C: 200 orders
```

### 3.1 The raw HTTP, step by step

**Step 2: authorization request** (the browser is redirected here):

```http
GET /oauth2/authorize?response_type=code
    &client_id=photoprint-web
    &redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback
    &scope=openid%20profile%20photos.read%20offline_access
    &state=Xk8s2...random
    &nonce=n-0S6_WzA2Mj
    &code_challenge=E9Melhoa2OwvFrEMTJguCHaoeK1t8URWbuGJSstw-cM
    &code_challenge_method=S256 HTTP/1.1
Host: auth.example.com
```

**Step 7: authorization response** (the AS redirects the browser back):

```http
HTTP/1.1 302 Found
Location: https://app.example.com/callback?code=SplxlOBeZQQYbYS6WxSbIA&state=Xk8s2...random&iss=https%3A%2F%2Fauth.example.com
```

On failure (user denied, bad scope), the AS redirects with `?error=access_denied&error_description=...&state=...` instead. If the `client_id` or `redirect_uri` itself is invalid, the AS **must not redirect** (that would turn it into an open redirector). It shows an error page instead.

**Step 9: token request** (back channel, from the client's server):

```http
POST /oauth2/token HTTP/1.1
Host: auth.example.com
Content-Type: application/x-www-form-urlencoded
Authorization: Basic cGhvdG9wcmludC13ZWI6czNjcjN0

grant_type=authorization_code
&code=SplxlOBeZQQYbYS6WxSbIA
&redirect_uri=https%3A%2F%2Fapp.example.com%2Fcallback
&code_verifier=dBjftJeZ4CVP-mB92K27uhbUJU1p1r_wW1gFWFOEjXk
```

**Step 11: token response:**

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store

{
  "access_token": "eyJhbGciOiJSUzI1NiIsInR5cCI6ImF0K2p3dCIsImtpZCI6IjIwMjYtMTAifQ...",
  "token_type": "Bearer",
  "expires_in": 600,
  "refresh_token": "8xLOxBtZp8...",
  "scope": "openid profile photos.read offline_access",
  "id_token": "eyJhbGciOiJSUzI1NiIsImtpZCI6IjIwMjYtMTAifQ..."
}
```

Token endpoint errors are JSON with HTTP 400/401: `{"error":"invalid_grant"}` (bad, expired or used code, PKCE mismatch, revoked refresh token), `invalid_client` (client authentication failed), `unauthorized_client`, `unsupported_grant_type`, `invalid_scope`.

### 3.2 Why a *code* and not the token directly?

The front channel (browser URL) leaks: browser history, server access logs, proxy logs, the `Referer` header, malicious extensions, and on mobile, **other apps registering the same custom URL scheme**. So the front channel carries only something that is:

- **short-lived** (around 60 s) and **single-use**;
- **useless on its own**, because redeeming it requires **client authentication** (confidential clients) and/or the **PKCE `code_verifier`**, which never left the client.

The tokens travel only over the back channel, as a direct TLS response to the client.

### 3.3 PKCE in depth (RFC 7636)

```text
code_verifier  = high-entropy random string, 43-128 chars of [A-Z a-z 0-9 - . _ ~]
code_challenge = BASE64URL( SHA256( ASCII(code_verifier) ) )      (method S256)
```

- The **challenge** goes out through the front channel in step 2. The AS stores it with the code.
- The **verifier** goes out only through the back channel in step 9. The AS hashes it and compares.
- An attacker who steals the code from the redirect doesn't have the verifier, and can't derive it from the challenge (SHA-256 is preimage-resistant).
- `plain` method (challenge = verifier) exists only for legacy reasons. **Always use `S256`.** The AS should reject `plain`.

**What attacks does PKCE stop?**

```mermaid
sequenceDiagram
    participant V as Victim's app
    participant M as Malicious app (same custom scheme)
    participant AS as Authorization Server
    V->>AS: /authorize ... code_challenge=H(v1)
    AS-->>M: redirect myapp://cb?code=ABC (OS delivers to the wrong app)
    M->>AS: POST /token code=ABC (no verifier, or a guessed one)
    AS-->>M: 400 invalid_grant (SHA256(guess) does not match H(v1))
```

1. **Code interception** (above): mobile custom URL schemes, logs, `Referer` leaks.
2. **Code injection**: an attacker injects a code (obtained for *their* account, or stolen) into the victim's callback, so the victim's session ends up bound to the attacker's account or vice versa. With PKCE the injected code was issued against a *different* challenge, so the victim client's verifier doesn't match and the exchange fails.

That's why PKCE is **recommended for confidential clients too** (RFC 9700), not just public ones. A client secret proves *which app* is redeeming. PKCE proves it's *the same instance/session that started the flow*.

### 3.4 `state`, `nonce`, PKCE: what each one binds

| Parameter | Generated by | Sent in | Checked by | Binds | Attack prevented |
|---|---|---|---|---|---|
| `state` | Client | Auth request; echoed in the callback | **Client** at the callback | callback ↔ the browser session that started login | **CSRF on the redirect URI** (attacker makes the victim's browser complete *the attacker's* login: "login CSRF") |
| PKCE verifier/challenge | Client | Challenge in the auth request, verifier at the token endpoint | **AS** at the token endpoint | code ↔ the client instance that requested it | Code **interception and injection** |
| `nonce` | Client | Auth request; returned **inside the ID token** | **Client** after the token exchange | ID token ↔ this login attempt | **ID-token replay/injection** |
| `iss` (RFC 9207) | AS | Callback | **Client** at the callback | response ↔ the AS the client sent the user to | **Mix-up** attacks (client talks to several ASs) |

Notes:
- `state` is also commonly used to carry app context (e.g. a `returnTo` path). Keep that context **server-side**, keyed by the random `state`, and validate `returnTo` as a same-origin relative path, or you've built an open redirect.
- RFC 9700 allows clients to rely on PKCE for CSRF protection **if** they've confirmed the AS enforces PKCE. In an interview, say you'd use both: they're cheap.

### 3.5 `redirect_uri` rules

- **Exact string match** against pre-registered URIs. No wildcards, no prefix matching, no "same domain". (Exception: on native apps, loopback `http://127.0.0.1:{any port}` may vary the port.)
- Why: with prefix or wildcard matching, an attacker uses `https://app.example.com/callback/../open-redirect?to=evil.com` or a subdomain they control, and the code (or, in implicit flow, the token) lands on the attacker's server.
- The client **must resend the same `redirect_uri`** at the token endpoint, and the AS checks it matches the one bound to the code.
- Callback pages should send `Referrer-Policy: no-referrer` and redirect immediately, so the code doesn't leak via `Referer` to third-party resources on that page.

---

## 4. Node.js: an Authorization Code + PKCE client (BFF style)

This is a confidential, server-side client. The browser never sees a token, only an `HttpOnly` session cookie. (Session/cookie configuration details are in [doc 13](13-nodejs-authentication-interview-guide.md).)

```js
// client/app.js
import express from "express";
import session from "express-session";
import crypto from "node:crypto";
import { createRemoteJWKSet, jwtVerify } from "jose";

const ISSUER = "https://auth.example.com";
const CLIENT_ID = process.env.OAUTH_CLIENT_ID;
const CLIENT_SECRET = process.env.OAUTH_CLIENT_SECRET;
const REDIRECT_URI = "https://app.example.com/callback";

// Discover endpoints once at startup instead of hard-coding them.
const metadata = await fetch(`${ISSUER}/.well-known/openid-configuration`).then((r) => r.json());
if (metadata.issuer !== ISSUER) throw new Error("Issuer mismatch in discovery document");
const AS_JWKS = createRemoteJWKSet(new URL(metadata.jwks_uri)); // caches keys, refetches on unknown kid

const b64url = (buf) => Buffer.from(buf).toString("base64url");
const randomToken = () => b64url(crypto.randomBytes(32)); // 256 bits -> 43 chars

const app = express();
app.use(
  session({
    secret: process.env.SESSION_SECRET,
    resave: false,
    saveUninitialized: false,
    cookie: { httpOnly: true, secure: true, sameSite: "lax" }, // lax: the AS -> /callback redirect is a top-level GET
    // store: RedisStore(...) in production, since tokens live in the session
  })
);

// ---------------------------------------------------------------------------
// Step 1-2: start login
// ---------------------------------------------------------------------------
app.get("/login", (req, res) => {
  const state = randomToken();
  const nonce = randomToken();
  const codeVerifier = randomToken();
  const codeChallenge = b64url(crypto.createHash("sha256").update(codeVerifier).digest());

  // Only allow same-origin relative paths, otherwise /login?returnTo=https://evil.com is an open redirect.
  const returnTo =
    typeof req.query.returnTo === "string" && /^\/(?!\/)/.test(req.query.returnTo) ? req.query.returnTo : "/";

  req.session.oauth = { state, nonce, codeVerifier, returnTo, startedAt: Date.now() };

  const url = new URL(metadata.authorization_endpoint);
  url.search = new URLSearchParams({
    response_type: "code",
    client_id: CLIENT_ID,
    redirect_uri: REDIRECT_URI,
    scope: "openid profile email orders:read offline_access",
    state,
    nonce,
    code_challenge: codeChallenge,
    code_challenge_method: "S256",
  }).toString();

  res.redirect(url.toString());
});

// ---------------------------------------------------------------------------
// Step 8-13: callback
// ---------------------------------------------------------------------------
app.get("/callback", async (req, res, next) => {
  try {
    res.set("Referrer-Policy", "no-referrer");
    const { code, state, iss, error } = req.query;
    const pending = req.session.oauth;
    delete req.session.oauth; // single use, even on failure

    if (!pending || typeof state !== "string" || state !== pending.state) {
      return res.status(400).send("Invalid or expired login attempt"); // CSRF / stale tab / replay
    }
    if (Date.now() - pending.startedAt > 10 * 60_000) return res.status(400).send("Login attempt expired");
    if (iss !== undefined && iss !== ISSUER) return res.status(400).send("Unexpected issuer"); // RFC 9207 mix-up defence
    if (error) return res.status(401).send(`Login failed: ${error}`); // e.g. access_denied
    if (typeof code !== "string") return res.status(400).send("Missing code");

    // Back channel: exchange code for tokens (client_secret_basic authentication)
    const basic = Buffer.from(
      `${encodeURIComponent(CLIENT_ID)}:${encodeURIComponent(CLIENT_SECRET)}`
    ).toString("base64");

    const tokenRes = await fetch(metadata.token_endpoint, {
      method: "POST",
      headers: { "Content-Type": "application/x-www-form-urlencoded", Authorization: `Basic ${basic}` },
      body: new URLSearchParams({
        grant_type: "authorization_code",
        code,
        redirect_uri: REDIRECT_URI,
        code_verifier: pending.codeVerifier,
      }),
      signal: AbortSignal.timeout(5000),
    });
    const tokens = await tokenRes.json();
    if (!tokenRes.ok) return res.status(401).send(`Token exchange failed: ${tokens.error}`);

    // OIDC: validate the ID token. This is what proves who logged in.
    const { payload: id } = await jwtVerify(tokens.id_token, AS_JWKS, {
      issuer: ISSUER,
      audience: CLIENT_ID, // ID token must be for *this* client
      algorithms: ["RS256", "ES256"], // never let the token pick (e.g. "none")
      maxTokenAge: "10m",
    });
    if (id.nonce !== pending.nonce) return res.status(401).send("Nonce mismatch");

    // Prevent session fixation: new session ID at privilege change
    await new Promise((resolve, reject) => req.session.regenerate((err) => (err ? reject(err) : resolve())));

    req.session.user = { sub: id.sub, email: id.email, name: id.name };
    req.session.tokens = {
      accessToken: tokens.access_token,
      refreshToken: tokens.refresh_token,
      expiresAt: Date.now() + tokens.expires_in * 1000,
    };
    res.redirect(pending.returnTo);
  } catch (err) {
    next(err);
  }
});
```

Key points to call out when walking through this code:
- The **user identity** is `iss` + `sub` from the ID token (`sub` is unique only per issuer). Don't key users by `email`: it can change, and some providers let users set unverified emails.
- The **access token is opaque to the client**. The client shouldn't parse it. Its format is a contract between the AS and RS only.
- Tokens are stored **server-side** (session store), so XSS can't exfiltrate them. Encrypt refresh tokens at rest if the session store is shared.

### 4.1 Calling the API with automatic refresh (and the concurrency bug)

```js
// client/tokens.js (reuses `metadata` and `basic` from the callback code above)
const refreshing = new Map(); // sessionId -> Promise. Single-flight per session, per process.

export async function getAccessToken(req) {
  const t = req.session.tokens;
  if (!t) throw new Error("Not logged in");
  if (Date.now() < t.expiresAt - 30_000) return t.accessToken; // 30s skew buffer

  if (!refreshing.has(req.sessionID)) {
    refreshing.set(
      req.sessionID,
      refresh(t.refreshToken).finally(() => refreshing.delete(req.sessionID))
    );
  }
  const fresh = await refreshing.get(req.sessionID);
  req.session.tokens = fresh;
  return fresh.accessToken;
}

async function refresh(refreshToken) {
  const res = await fetch(metadata.token_endpoint, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded", Authorization: `Basic ${basic}` },
    body: new URLSearchParams({ grant_type: "refresh_token", refresh_token: refreshToken }),
    signal: AbortSignal.timeout(5000),
  });
  const body = await res.json();
  if (!res.ok) {
    // invalid_grant -> refresh token expired/revoked/reused: force re-login
    throw Object.assign(new Error(`refresh failed: ${body.error}`), { reauth: body.error === "invalid_grant" });
  }
  return {
    accessToken: body.access_token,
    refreshToken: body.refresh_token ?? refreshToken, // rotation: the AS may return a new one
    expiresAt: Date.now() + body.expires_in * 1000,
  };
}
```

**Interview gold:** with **refresh-token rotation**, two concurrent requests for the same session that both see an expired access token will both try to refresh with the *same* refresh token. The second looks like **token reuse**, and the AS revokes the entire token family, logging the user out at random. Fixes:
- **Single-flight per session**: the in-process `Map` above. Across multiple Node instances, use a short Redis lock keyed by session ([doc 12 §7](12-redis-nodejs-deep-dive.md)) or keep session affinity.
- **Proactive refresh** before expiry instead of on demand.
- AS-side **grace period**: accept the immediately previous refresh token for a few seconds and return the same new pair (Auth0 and Okta offer a "reuse interval").

---

## 5. Resource server: validating access tokens

Two token formats, two validation models:

```mermaid
flowchart TB
    REQ["Request with<br/>Authorization: Bearer token"] --> FMT{"Token format?"}
    FMT -- "JWT (self-contained)" --> L1["Verify signature with cached JWKS (by kid)"]
    L1 --> L2["Check iss, aud, exp/nbf, typ, alg allow-list"]
    L2 --> L3["Check scopes for this route"]
    FMT -- "Opaque (reference)" --> I1["POST /introspect to AS<br/>(cache result briefly)"]
    I1 --> I2{"active: true?"}
    I2 -- no --> R401["401 invalid_token"]
    I2 -- yes --> L3
    L3 -- missing --> R403["403 insufficient_scope"]
    L3 -- ok --> AUTHZ["Business authorization:<br/>does sub own this resource?"]
```

| | **JWT access token** | **Opaque token + introspection** |
|---|---|---|
| Validation | Local: signature + claims | Network call to AS per request (cacheable) |
| Latency / AS load | Microseconds, no AS dependency at request time | Extra hop. AS becomes a hot path and single point of failure |
| Revocation | Effective only at `exp` (unless you add a denylist) | Immediate |
| Privacy | Claims readable by anyone holding the token (base64, **not encrypted**) | Token reveals nothing |
| Size | Larger (hundreds of bytes to KBs) | Small |
| Typical use | Internal microservices at scale, short TTL (5-15 min) | High-security APIs, tokens leaving your trust boundary, or when instant revocation matters |

A common hybrid: **opaque tokens at the edge, JWTs inside**. The API gateway introspects (or uses token exchange) and forwards a short-lived internal JWT to services (the "phantom token" pattern).

### 5.1 JWT validation middleware with `jose`

```js
// api/auth.js
import { createRemoteJWKSet, jwtVerify } from "jose";

const ISSUER = "https://auth.example.com";
const AUDIENCE = "https://api.example.com/orders"; // this API's identifier at the AS
const JWKS = createRemoteJWKSet(new URL(`${ISSUER}/.well-known/jwks.json`), {
  cacheMaxAge: 10 * 60_000, // keys cached in memory; unknown kid triggers a (rate-limited) refetch
});

export function requireScopes(...required) {
  return async (req, res, next) => {
    const match = /^Bearer ([A-Za-z0-9\-._~+/]+=*)$/.exec(req.headers.authorization ?? "");
    if (!match) {
      return res.status(401).set("WWW-Authenticate", 'Bearer realm="orders"').end();
    }
    let payload;
    try {
      ({ payload } = await jwtVerify(match[1], JWKS, {
        issuer: ISSUER,
        audience: AUDIENCE, // reject tokens minted for other APIs (confused deputy)
        algorithms: ["RS256", "ES256"], // allow-list. Blocks alg=none and RS/HS confusion
        typ: "at+jwt", // RFC 9068. Stops an ID token being used as an access token (only if your AS sets it)
        clockTolerance: 30,
      }));
    } catch {
      return res.status(401).set("WWW-Authenticate", 'Bearer error="invalid_token"').end();
    }

    const granted = new Set(typeof payload.scope === "string" ? payload.scope.split(" ") : []);
    if (!required.every((s) => granted.has(s))) {
      return res
        .status(403)
        .set("WWW-Authenticate", `Bearer error="insufficient_scope", scope="${required.join(" ")}"`)
        .end();
    }

    req.auth = { sub: payload.sub, clientId: payload.client_id, scopes: granted, jti: payload.jti };
    next();
  };
}

// usage
app.get("/orders/:id", requireScopes("orders:read"), async (req, res) => {
  const order = await db.orders.findById(req.params.id);
  // Scope says the CLIENT may read orders. It does NOT say THIS USER may read THIS order.
  if (!order || order.userId !== req.auth.sub) return res.status(404).end(); // 404 avoids leaking existence
  res.json(order);
});
```

**401 vs. 403** (RFC 6750): `401` means "no valid token, authenticate again" (missing, expired, bad signature). `403 insufficient_scope` means "valid token, but it lacks the permission". Clients react differently: refresh/re-login vs. request more scope.

### 5.2 Introspection for opaque tokens (RFC 7662)

```js
// api/introspect.js
import { LRUCache } from "lru-cache";
const cache = new LRUCache({ max: 50_000 });

export async function introspect(token) {
  const hit = cache.get(token);
  if (hit) return hit;

  const res = await fetch(`${ISSUER}/oauth2/introspect`, {
    method: "POST",
    headers: {
      "Content-Type": "application/x-www-form-urlencoded",
      Authorization: `Basic ${Buffer.from(`${RS_CLIENT_ID}:${RS_SECRET}`).toString("base64")}`, // the RS authenticates too
    },
    body: new URLSearchParams({ token, token_type_hint: "access_token" }),
    signal: AbortSignal.timeout(2000),
  });
  if (!res.ok) throw new Error(`introspection unavailable: ${res.status}`); // fail closed -> 503, not 200
  const data = await res.json(); // { active, scope, client_id, sub, aud, exp, ... }

  // Cache positives briefly (bounded by exp). Revocation lag = cache TTL.
  if (data.active) {
    const ttl = Math.min(60_000, data.exp * 1000 - Date.now());
    if (ttl > 0) cache.set(token, data, { ttl });
  }
  return data;
}
```

The cache TTL is the knob: **revocation latency vs. AS load**. In an interview, state the trade-off explicitly.

### 5.3 Scopes vs. permissions vs. audience

- **Scope** limits what the **client** may do on the user's behalf (`orders:read`). It is coarse and consent-oriented.
- **Permissions/roles** describe what the **user** may do (admin, owns order 42). They're usually enforced by the RS from its own data, or from claims the AS adds.
- **Effective access = scope ∩ user's permissions.** An admin's token minted for a "read-only reporting" client must not be able to delete things.
- **Audience (`aud`)** says *which API* the token is for. Without an audience check, a token for API A replayed to API B is accepted (**confused deputy**). Use one audience per API, and use **token exchange** (RFC 8693) to get a down-scoped token when service A calls service B on the user's behalf, instead of forwarding the user's token everywhere.

---

## 6. Client Credentials (machine-to-machine)

No user is involved. The client authenticates as itself and gets a token representing *the client*.

```mermaid
sequenceDiagram
    participant S as Billing service (client)
    participant AS as Authorization Server
    participant RS as Invoices API
    S->>AS: POST /token grant_type=client_credentials<br/>scope=invoices:write + client auth
    AS-->>S: {access_token, expires_in: 3600}
    S->>RS: POST /invoices, Bearer access_token
    RS-->>S: 201
    Note over S: cache the token until shortly before expiry<br/>(no refresh token, just ask again)
```

```js
// service/serviceToken.js
let cached = null; // { token, expiresAt }
let inflight = null; // single-flight: N concurrent requests -> 1 token call

export async function getServiceToken() {
  if (cached && Date.now() < cached.expiresAt - 60_000) return cached.token;
  inflight ??= fetchToken().finally(() => (inflight = null));
  return inflight;
}

async function fetchToken() {
  const res = await fetch(TOKEN_ENDPOINT, {
    method: "POST",
    headers: { "Content-Type": "application/x-www-form-urlencoded", Authorization: `Basic ${BASIC}` },
    body: new URLSearchParams({ grant_type: "client_credentials", scope: "invoices:write" }),
    signal: AbortSignal.timeout(5000),
  });
  const body = await res.json();
  if (!res.ok) throw new Error(`client_credentials failed: ${body.error}`);
  cached = { token: body.access_token, expiresAt: Date.now() + body.expires_in * 1000 };
  return cached.token;
}
```

- **Don't fetch a token per request.** It's a classic production incident: the AS gets rate-limited or overloaded by its own services. Cache it and single-flight the fetch.
- No refresh token is issued. Just request a new one.
- **Client authentication options**, weakest to strongest: `client_secret_basic`/`client_secret_post` (shared secret) → `private_key_jwt` (client signs a JWT assertion with its private key; the AS stores only the public key) → **mTLS** (RFC 8705, the token can also be bound to the client certificate). In Kubernetes, prefer workload identity (SPIFFE/SPIRE, cloud IAM federation) over long-lived secrets.

---

## 7. Refresh tokens: rotation and reuse detection

**Why refresh tokens exist:** keep access tokens short-lived (limit the blast radius of a leaked bearer token, bound the revocation lag of JWTs) without making the user log in every 10 minutes.

| Property | Access token | Refresh token |
|---|---|---|
| Sent to | Resource servers (many) | Only the AS token endpoint |
| Lifetime | Minutes | Hours to months (sliding + absolute max) |
| Format | Often JWT | **Opaque**, stored hashed by the AS |
| Revocable | Hard if JWT | Yes, it's a DB lookup on every use |

**Rotation with reuse detection** (required for public clients unless refresh tokens are sender-constrained):

```mermaid
sequenceDiagram
    participant C as Legit client
    participant X as Attacker (stole RT1)
    participant AS as Authorization Server
    C->>AS: refresh with RT1
    AS-->>C: AT2 + RT2 (RT1 marked used, same family F)
    X->>AS: refresh with RT1 (stolen earlier)
    Note over AS: RT1 already used, so this is REUSE<br/>revoke the whole family F (RT2 included)
    AS-->>X: 400 invalid_grant
    C->>AS: refresh with RT2
    AS-->>C: 400 invalid_grant (user must re-authenticate)
```

The AS can't tell which party is legitimate, so it kills both. The attacker's window is limited to before the next legitimate refresh, and the legit user is forced to re-authenticate, which surfaces the compromise.

**Sender-constrained tokens** (the stronger alternative): bind tokens to a key the client holds, so a stolen token is useless without the private key.
- **DPoP** (RFC 9449): the client generates a key pair (in a browser, a non-extractable WebCrypto key). Every request carries a `DPoP` header: a JWT signed with that key containing the HTTP method, URL, timestamp and `jti` (plus `ath`, a hash of the access token, at the RS). The token contains `cnf.jkt`, the key thumbprint. The RS verifies the proof matches.
- **mTLS** (RFC 8705): the token contains the client certificate thumbprint (`cnf.x5t#S256`). Natural for service-to-service traffic, impractical in browsers.

---

## 8. The other grants: when to use what

```mermaid
flowchart TD
    Q1{"Is there a user?"} -- no --> CC["Client Credentials<br/>(+ private_key_jwt / mTLS)"]
    Q1 -- yes --> Q2{"Can the device show a browser<br/>and receive a redirect?"}
    Q2 -- "no (TV, CLI, IoT)" --> DEV["Device Authorization Grant (RFC 8628)"]
    Q2 -- yes --> Q3{"Where does the app run?"}
    Q3 -- "server-side web app" --> ACS["Auth Code + PKCE<br/>confidential client"]
    Q3 -- "SPA" --> BFF["Prefer BFF: Auth Code + PKCE on the backend<br/>else browser-only Auth Code + PKCE (public client)"]
    Q3 -- "mobile / desktop" --> NAT["Auth Code + PKCE in the system browser<br/>(ASWebAuthenticationSession / Custom Tabs)<br/>claimed https redirect or loopback"]
    Q1 -- "service acting for a user<br/>calling another service" --> TE["Token Exchange (RFC 8693)"]
```

### 8.1 Device Authorization Grant (RFC 8628)

For devices that can't (easily) show a browser: smart TVs, CLIs (`gh auth login`, `az login --use-device-code`).

```mermaid
sequenceDiagram
    participant D as TV / CLI
    participant AS as Authorization Server
    actor U as User on phone
    D->>AS: POST /device_authorization client_id, scope
    AS-->>D: device_code, user_code=WDJB-MJHT,<br/>verification_uri, interval=5, expires_in=900
    D->>U: "Go to example.com/device, enter WDJB-MJHT"
    loop every interval seconds
        D->>AS: POST /token grant_type=device_code, device_code
        AS-->>D: 400 authorization_pending (or slow_down: increase interval by 5s)
    end
    U->>AS: opens URL, logs in, enters code, approves
    D->>AS: POST /token grant_type=device_code, device_code
    AS-->>D: 200 access_token, refresh_token
```

Risk: **device-code phishing**. An attacker starts a flow and sends the victim the legitimate URL and a code ("enter this code to verify your account"). Mitigations: show the device/client name and location on the approval screen, short `expires_in`, and restrict which clients may use this grant.

### 8.2 Deprecated grants (and why: a frequent question)

**Implicit grant** (`response_type=token`): the AS returned the access token directly in the redirect URL fragment. It was designed for SPAs before CORS was widespread.
- The token sits in the URL, so it leaks via history, `Referer` (in some cases), extensions and logs.
- No client authentication and no PKCE binding. **Token injection** is possible.
- No refresh tokens, so SPAs used hidden-iframe "silent renew", now broken by third-party-cookie blocking.
- **Replacement:** Auth Code + PKCE (CORS made the back-channel POST possible from the browser).

**Resource Owner Password Credentials** (`grant_type=password`): the app collects the user's username and password and posts them to the token endpoint.
- It defeats the purpose of OAuth: the client sees the password.
- No MFA, no consent, no federation (you can't type your Google password into a password grant to a third-party AS).
- It trains users to type passwords into non-AS UIs, i.e. phishing.
- **Replacement:** Auth Code + PKCE. For legacy migration only, first-party only.

---

## 9. OpenID Connect: authentication on top of OAuth

OIDC = OAuth 2.0 + `scope=openid` + **ID token** + standard claims + `/userinfo` + discovery + (optional) session management/logout.

### 9.1 ID token vs. access token

| | **ID token** | **Access token** |
|---|---|---|
| Audience | The **client** (`aud = client_id`) | The **API** (`aud = api identifier`) |
| Purpose | Tell the client **who authenticated, when and how** | Authorize **API calls** |
| Format | Always a JWT | JWT or opaque |
| Who reads it | The client validates it | Only the RS. The client treats it as opaque |
| Sent to APIs? | **Never** | Yes |

A decoded ID token:

```json
{
  "iss": "https://auth.example.com",
  "sub": "248289761001",
  "aud": "photoprint-web",
  "exp": 1791110400,
  "iat": 1791109800,
  "auth_time": 1791109790,
  "nonce": "n-0S6_WzA2Mj",
  "acr": "urn:mace:incommon:iap:silver",
  "amr": ["pwd", "otp"],
  "email": "jane@example.com",
  "email_verified": true,
  "name": "Jane Doe"
}
```

**ID token validation checklist** (what `openid-client` does for you):
1. Signature valid with the AS's JWKS key matching `kid`. `alg` is in your allow-list (never `none`).
2. `iss` equals the expected issuer exactly.
3. `aud` contains your `client_id` (and if there are multiple audiences, `azp` equals your `client_id`).
4. `exp` is in the future, and `iat` is reasonable (clock skew tolerance).
5. `nonce` equals the one stored in the session.
6. If you require MFA or a recent login: check `acr`/`amr` and `auth_time` (request them with `max_age` / `acr_values`).

### 9.2 Useful OIDC request parameters

- `prompt=login` (force re-authentication), `prompt=consent`, `prompt=none` (silent: fail with `login_required` instead of showing UI).
- `max_age=300`: the user must have authenticated in the last 5 min. Use it for step-up before sensitive actions; the AS returns `auth_time`.
- `login_hint`: prefill the username.
- `acr_values`: request an authentication strength (e.g. phishing-resistant MFA).

### 9.3 Logout is hard (know why)

Logging out of the app ≠ logging out of the AS (SSO session) ≠ revoking tokens.
- **App logout:** destroy the local session and **revoke the refresh token** (RFC 7009 `/revoke`).
- **RP-initiated logout:** redirect to the AS `end_session_endpoint` with `id_token_hint` and `post_logout_redirect_uri` to end the SSO session.
- **Back-channel logout:** the AS POSTs a signed `logout_token` to each client's endpoint, and clients kill the matching sessions (look up by `sid`/`sub`). This is robust and the recommended option.
- **Front-channel logout** relies on iframes and third-party cookies, which are increasingly broken.
- Already-issued JWT access tokens stay valid until `exp`. That's the price of stateless validation.

---

## 10. Browser apps: where do the tokens live?

```mermaid
flowchart LR
    subgraph A["Option A: Backend-for-Frontend (recommended)"]
        B1["SPA (browser)"] -- "session cookie<br/>HttpOnly, Secure, SameSite" --> BFF1["BFF (Node)<br/>confidential OAuth client<br/>holds AT/RT server-side"]
        BFF1 -- "Bearer AT" --> API1["APIs"]
    end
    subgraph B["Option B: browser-only public client"]
        B2["SPA (browser)<br/>Auth Code + PKCE<br/>AT in memory, RT rotated or DPoP"] -- "Bearer AT (CORS)" --> API2["APIs"]
    end
```

| | **BFF** | **Tokens in the browser** |
|---|---|---|
| XSS impact | Attacker can make requests **through** the user's session while the page is open, but **can't steal tokens** | Attacker can **exfiltrate tokens** and use them from anywhere, including long-lived refresh tokens |
| Client type | Confidential (client secret / `private_key_jwt`) | Public |
| CSRF | Must handle it (SameSite + CSRF token / custom header, see [doc 13](13-nodejs-authentication-interview-guide.md)) | Not cookie-based, so no CSRF |
| Infra | Need a backend (often already exist) | Static hosting only |
| Third-party cookie blocking | Unaffected (first-party cookie) | Silent renew via iframe breaks. Needs refresh tokens in the browser |

If you must keep tokens in the browser: keep the access token **in memory** (not `localStorage`), use rotating refresh tokens with short absolute lifetimes or DPoP with a non-extractable key, and enforce a strict CSP. Still, "XSS = full token theft" is the core trade-off. That's why the IETF browser-based apps BCP prefers BFF.

**Mobile apps:** use the **system browser** (ASWebAuthenticationSession on iOS, Custom Tabs on Android), never an embedded WebView. With a WebView the app can read the user's password and cookies, and you lose the shared SSO session. Use claimed `https` redirect URIs (Universal Links / App Links) over custom schemes, plus PKCE.

---

## 11. Attack catalogue (with the defence)

| Attack | How it works | Defence |
|---|---|---|
| **Login CSRF via callback** | Attacker gets a code for *their* account and makes the victim's browser hit `/callback?code=attacker_code`. The victim is now logged in as the attacker and uploads data into the attacker's account | `state` bound to the session (and/or PKCE) |
| **Code interception** | Malicious app with the same custom scheme, logs, Referer | PKCE, claimed https redirects, `Referrer-Policy: no-referrer` |
| **Code injection** | Stolen code replayed into the attacker's own session with the legit client | PKCE (verifier bound to the original session), single-use codes |
| **Open redirect / redirect_uri manipulation** | Loose matching sends the code to the attacker | Exact `redirect_uri` match, no open redirects on the client domain |
| **Mix-up attack** | Client supports several ASs. Attacker's AS tricks the client into sending a code from honest AS-1 to attacker AS-2's token endpoint | Per-AS distinct redirect URIs, `iss` parameter check (RFC 9207) |
| **Token substitution ("OAuth as login")** | Access token from another client accepted as proof of login | Use OIDC, validate the ID token's `aud` and `nonce` |
| **Confused deputy / audience** | Token for API A replayed against API B | Validate `aud` per API. Token exchange for downstream calls |
| **JWT `alg` attacks** | `alg: none`, or RS256 → HS256 confusion (sign with the public key as an HMAC secret) | Explicit algorithm allow-list, key type bound to algorithm (jose enforces this) |
| **ID token used as access token** | Client sends the ID token to an API that only checks the signature | RS checks `aud` and `typ: at+jwt` |
| **Refresh token theft** | XSS or a stolen device | BFF, rotation + reuse detection, DPoP, absolute lifetime, revoke on password change |
| **Over-scoped tokens** | Client requests `*` "just in case" | Least privilege, incremental consent, per-API audiences |
| **Consent phishing** | Malicious app registered on a real AS asks the user for broad scopes ("illicit consent grant") | Publisher verification, admin consent for sensitive scopes, app reviews, monitoring |
| **Authorization request tampering** | Parameters modified in the front channel | **PAR** (RFC 9126): client POSTs the request to the AS first, then sends only a `request_uri` reference in the browser |
| **Token leakage in logs** | Bearer tokens logged by proxies or APM | Never put tokens in URLs. Redact the `Authorization` header in logs |

---

## 12. System design: building and scaling an authorization server

This is the "design Auth0 / design an SSO service" angle.

### 12.1 Components

```mermaid
flowchart TB
    subgraph Edge
        LB["Load balancer / WAF<br/>rate limit /token and /login per IP and client"]
    end
    subgraph AS["Authorization Server (stateless Node instances)"]
        AUTHZ["/authorize + login/consent UI"]
        TOKEN["/token"]
        INTRO["/introspect, /revoke, /userinfo"]
        DISC["/.well-known/* (CDN-cacheable)"]
    end
    subgraph Stores
        REDIS[("Redis<br/>auth codes, login transactions,<br/>PAR requests, device codes<br/>(TTL, single-use)")]
        PG[("Postgres<br/>users, clients, consents,<br/>refresh tokens (hashed), audit")]
        KMS[("KMS / HSM<br/>signing keys")]
    end
    RSs["Resource servers<br/>(cache JWKS, verify locally)"]

    LB --> AUTHZ & TOKEN & INTRO & DISC
    AUTHZ --> REDIS & PG
    TOKEN --> REDIS & PG & KMS
    INTRO --> PG
    RSs -. "JWKS fetch (rare)" .-> DISC
```

Why each store:
- **Redis** for authorization codes, PKCE challenges and device codes: short TTL (60 s - 15 min), very high write/read rate, data is disposable. Atomic `GETDEL` gives single-use for free.
- **Postgres** for refresh tokens, consents, clients and users: durable, needs transactions (rotation), queried for revocation ("revoke all tokens for user X").
- **KMS/HSM** for signing keys: private keys never in app memory or env vars. The cost is a signing latency per token. Alternatively, keep short-lived keys in memory, wrapped by the KMS.
- **The JWT verification path doesn't touch the AS at all.** This is what makes the system scale: the AS handles logins and token issuance (≈ one call per user per access-token lifetime), not every API request.

### 12.2 Token endpoint core: authorization-code redemption

```js
// as/token.js (authorization_code grant, simplified)
import crypto from "node:crypto";
import { SignJWT } from "jose";

const sha256b64url = (s) => crypto.createHash("sha256").update(s).digest("base64url");
const safeEqual = (a, b) =>
  a.length === b.length && crypto.timingSafeEqual(Buffer.from(a), Buffer.from(b));

// Issued at /authorize after login + consent
export async function issueCode({ clientId, redirectUri, userId, scope, codeChallenge, nonce, authTime }) {
  const code = crypto.randomBytes(32).toString("base64url");
  await redis.set(
    `authcode:${sha256b64url(code)}`, // store a hash: a Redis dump doesn't leak usable codes
    JSON.stringify({ clientId, redirectUri, userId, scope, codeChallenge, nonce, authTime }),
    { EX: 60 }
  );
  return code;
}

export async function redeemCode(client /* already authenticated, or a public client */, params) {
  const { code, redirect_uri, code_verifier } = params;
  if (!code || !code_verifier) throw oauthError("invalid_request");

  // Atomic read-and-delete: two concurrent redemptions can't both succeed
  const raw = await redis.getDel(`authcode:${sha256b64url(code)}`);
  if (!raw) {
    // Expired or already used. Per RFC 6749 §4.1.2, if a code is used twice the AS SHOULD
    // revoke tokens already issued from it (keep a short-lived "used" marker to detect this).
    throw oauthError("invalid_grant");
  }
  const grant = JSON.parse(raw);

  if (grant.clientId !== client.id) throw oauthError("invalid_grant");
  if (grant.redirectUri !== redirect_uri) throw oauthError("invalid_grant");
  if (!safeEqual(sha256b64url(code_verifier), grant.codeChallenge)) throw oauthError("invalid_grant");

  const now = Math.floor(Date.now() / 1000);
  const { kid, privateKey } = await signingKeys.current();

  const accessToken = await new SignJWT({ scope: grant.scope, client_id: client.id })
    .setProtectedHeader({ alg: "ES256", kid, typ: "at+jwt" })
    .setIssuer(ISSUER)
    .setSubject(grant.userId)
    .setAudience(audienceForScopes(grant.scope))
    .setIssuedAt(now)
    .setExpirationTime(now + 600)
    .setJti(crypto.randomUUID())
    .sign(privateKey);

  const refreshToken = grant.scope.split(" ").includes("offline_access")
    ? await refreshTokens.create({ userId: grant.userId, clientId: client.id, scope: grant.scope })
    : undefined;

  // id_token omitted for brevity: same pattern with aud=client.id, nonce, auth_time
  return { access_token: accessToken, token_type: "Bearer", expires_in: 600, refresh_token: refreshToken, scope: grant.scope };
}
```

### 12.3 Refresh token storage and atomic rotation

```sql
CREATE TABLE refresh_tokens (
  token_hash   bytea PRIMARY KEY,           -- SHA-256 of the token; the raw token is never stored
  family_id    uuid        NOT NULL,         -- all rotations from one login
  user_id      uuid        NOT NULL,
  client_id    text        NOT NULL,
  scope        text        NOT NULL,
  created_at   timestamptz NOT NULL DEFAULT now(),
  expires_at   timestamptz NOT NULL,         -- sliding expiry
  family_expires_at timestamptz NOT NULL,    -- absolute max session lifetime
  used_at      timestamptz,
  revoked_at   timestamptz
);
CREATE INDEX ON refresh_tokens (family_id);
CREATE INDEX ON refresh_tokens (user_id);    -- "log out everywhere", password change
```

```js
// as/refresh.js. Rotation with reuse detection, race-safe via a conditional UPDATE.
export async function rotate(client, presented) {
  const hash = crypto.createHash("sha256").update(presented).digest();

  return db.tx(async (tx) => {
    // Claim the token: only one concurrent caller can flip used_at from NULL
    const claimed = await tx.oneOrNone(
      `UPDATE refresh_tokens SET used_at = now()
        WHERE token_hash = $1 AND used_at IS NULL AND revoked_at IS NULL
          AND expires_at > now() AND family_expires_at > now() AND client_id = $2
        RETURNING family_id, user_id, scope, family_expires_at`,
      [hash, client.id]
    );

    if (!claimed) {
      const existing = await tx.oneOrNone(`SELECT family_id, used_at FROM refresh_tokens WHERE token_hash = $1`, [hash]);
      if (existing?.used_at) {
        // REUSE: someone presented an already-rotated token. Kill the whole family.
        await tx.none(`UPDATE refresh_tokens SET revoked_at = now() WHERE family_id = $1 AND revoked_at IS NULL`, [existing.family_id]);
        audit.warn("refresh_token_reuse", { familyId: existing.family_id, clientId: client.id });
      }
      throw oauthError("invalid_grant");
    }

    const next = crypto.randomBytes(32).toString("base64url");
    await tx.none(
      `INSERT INTO refresh_tokens (token_hash, family_id, user_id, client_id, scope, expires_at, family_expires_at)
       VALUES ($1, $2, $3, $4, $5, LEAST(now() + interval '14 days', $6), $6)`,
      [crypto.createHash("sha256").update(next).digest(), claimed.family_id, claimed.user_id, client.id, claimed.scope, claimed.family_expires_at]
    );
    return { refreshToken: next, userId: claimed.user_id, scope: claimed.scope };
  });
}
```

This is the same **conditional `UPDATE … WHERE state = expected`** pattern as inventory reservation in [doc 11](11-order-concurrency-idempotency.md). The database row is the lock.

### 12.4 Signing-key rotation without breaking verifiers

RSs cache the JWKS, so a new key must be **published before it's used**, and an old key must stay published **until every token it signed has expired**:

```text
t0   add K2 to JWKS (still signing with K1)          JWKS = [K1, K2]
t0 + JWKS cache TTL (e.g. 1h)                        all RSs now know K2
t1   start signing with K2                            JWKS = [K1, K2]
t1 + max access/ID token lifetime                     no live tokens signed by K1
t2   remove K1                                        JWKS = [K2]
```

- Each JWT header carries a `kid`. Verifiers select the key by `kid`, and on an unknown `kid` refetch the JWKS (rate-limited, as `jose`'s `createRemoteJWKSet` does).
- **Emergency rotation** (key compromise): remove the key immediately and accept that tokens it signed fail. That's why access tokens should be short-lived.

### 12.5 Scale and availability numbers to reason with

- 10 M DAU, 10-minute access tokens, 8 active hours/day → about 48 refreshes per user per day → **~480 M token requests/day ≈ 5.5 k/s average**, peaks perhaps 3-5×. Each is a DB transaction (rotation) plus a signature (ECDSA P-256 signing is roughly tens of µs per core). It's horizontally scalable, and **Postgres write throughput for rotation** is the thing to size. Partition `refresh_tokens` by `family_id` hash if needed.
- API traffic (say 500 k req/s) **never hits the AS** with JWTs. With introspection it would, which is the strongest argument for JWT access tokens at scale.
- **AS outage blast radius:** with JWTs, existing sessions keep working until their access tokens expire. New logins and refreshes fail. Short-TTL tokens make the AS's availability more critical. That's a real trade-off against revocation latency.
- **Multi-region:** signing keys shared via KMS multi-region keys (or region-specific keys, all published in one JWKS). Refresh-token rotation needs a strongly consistent write per family, so either home a family to a region or use a globally consistent DB.
- **Revocation for JWTs** when you really need it: a `jti`/`sub` denylist pushed to RSs (Redis Pub/Sub or a small replicated set), or `token_version` per user checked against a cache. Keep it small: only entries newer than the max token TTL.

---

## 13. OAuth 2.1 in one table (what changed vs. 2.0)

| OAuth 2.0 (2012) | OAuth 2.1 (draft) / Security BCP |
|---|---|
| PKCE optional, for public clients | PKCE **required** for all authorization-code clients |
| Implicit grant | **Removed** |
| Password grant | **Removed** |
| Redirect URI matching loosely defined | **Exact string matching** |
| Bearer token in query string allowed | **Not allowed** (header or form body only) |
| Refresh tokens for public clients unconstrained | Must be **sender-constrained or rotated** |

---

## 14. Interview Q&A bank

**Q: What's the difference between OAuth 2.0 and OpenID Connect?**
OAuth is delegated authorization: the client gets an access token to call an API on the user's behalf. It defines no standard way to learn who the user is. OIDC is an identity layer on OAuth: `scope=openid` returns an ID token (a JWT with audience = the client, carrying `sub`, `auth_time`, `nonce`), plus `/userinfo` and discovery. Use OIDC for login, OAuth for API access.

**Q: Walk me through the Authorization Code flow. Why two steps?**
Redirect to `/authorize` with `client_id`, `redirect_uri`, `scope`, `state`, PKCE challenge → user authenticates and consents at the AS → redirect back with a code → client POSTs the code + verifier (+ client authentication) to `/token` → receives tokens. Two steps because the front channel (browser URL) is leaky. Only a short-lived, single-use, client-bound code passes through it. Tokens come back only on the direct TLS back channel.

**Q: What does PKCE protect against, and why use it with a client secret too?**
Code interception and code injection. The verifier never travels through the browser, so a stolen code can't be redeemed, and an injected code fails because it's bound to a different challenge. A client secret proves which *application* redeems a code, not which *session* started the flow, so injection attacks still work against confidential clients without PKCE.

**Q: `state` vs. `nonce`?**
`state` is checked by the client at the callback and protects the redirect against CSRF (login CSRF). `nonce` goes into the ID token, and the client checks it after the token exchange; it binds the ID token to this login attempt, preventing replay. Different layers, different attacks.

**Q: Why was the implicit flow deprecated?**
Tokens in the URL fragment leak (history, extensions, Referer), there's no binding to the client (token injection), and there are no refresh tokens (iframe silent renew broke with third-party-cookie blocking). CORS made a browser POST to `/token` possible, so Auth Code + PKCE replaced it.

**Q: JWT vs. opaque access tokens?**
JWT: validated locally, no AS call per request, scales well, but can't be revoked before `exp` and its claims are readable. Opaque: tiny and private, instantly revocable, but each validation is an introspection call (cache it, and the cache TTL becomes your revocation latency). Common hybrid: opaque outside, JWT inside (phantom token at the gateway).

**Q: How do you revoke a JWT access token?**
You mostly don't. Keep them short (5-15 min) and revoke the refresh token, so the session dies at the next refresh. If you need faster revocation: a `jti` or per-user `token_version` denylist distributed to RSs, kept small because entries only need to live for the token's max TTL. Or switch that API to introspection.

**Q: How should an API validate an access token?**
Signature via JWKS (selected by `kid`, from an allow-listed `alg`), `iss` exact match, `aud` equals this API, `exp`/`nbf` with small clock skew, `typ` = `at+jwt` if available, then scopes for the route, then business-level authorization (does `sub` own the resource?). Return 401 for an invalid token, 403 `insufficient_scope` for missing scope.

**Q: Where should an SPA store tokens?**
Ideally nowhere: use a BFF, so the server is the OAuth client and the browser gets an `HttpOnly` `SameSite` session cookie. If tokens must be in the browser: access token in memory, rotating refresh token with an absolute lifetime or DPoP with a non-extractable key, strict CSP. `localStorage` is readable by any XSS. Either way, XSS lets the attacker act as the user while the page is open. BFF prevents token *exfiltration*.

**Q: Explain refresh-token rotation and reuse detection.**
Every refresh returns a new refresh token and invalidates the old one. All tokens from one login share a family ID. If an already-used token is presented, the AS can't tell the thief from the user, so it revokes the whole family, forcing re-login and cutting off the thief. Pitfall: concurrent refreshes from the same legitimate client look like reuse, so single-flight refreshes, or use an AS grace window.

**Q: How do two microservices authenticate to each other?**
Client Credentials grant with `private_key_jwt` or mTLS client authentication, a cached token per audience (single-flight fetch, renew before expiry), RS checks `aud` and scope. If the call is on behalf of a user, use token exchange (RFC 8693) to get a down-scoped token carrying both identities, instead of forwarding the user's token.

**Q: What's a mix-up attack?**
A client that works with several ASs is tricked by a malicious AS into sending a code issued by the honest AS to the attacker's token endpoint. Defences: a distinct redirect URI per AS, and checking the `iss` parameter in the authorization response (RFC 9207) against the AS the request was sent to.

**Q: How would you design the authorization-code storage?**
Redis key = hash of the code, value = `{client_id, redirect_uri, user, scope, code_challenge, nonce}`, TTL about 60 s, redeemed with `GETDEL` so it's atomic and single-use. Keep a short "used" marker so a second redemption attempt can trigger revocation of tokens issued from that code.

**Q: How do you rotate signing keys?**
Publish the new key in the JWKS first, wait at least the verifiers' JWKS cache TTL, switch signing to it, keep the old public key until the max token lifetime has passed, then remove it. Verifiers select by `kid` and refetch the JWKS on an unknown `kid`.

**Q: What happens to your system if the authorization server goes down?**
With JWT access tokens, APIs keep working for existing tokens until they expire. New logins and refreshes fail, so users drop off as tokens expire. With introspection, everything fails immediately (or after the cache TTL). Mitigations: multi-AZ/multi-region AS, CDN-cached discovery/JWKS, an access-token TTL chosen with outage tolerance in mind.

**Q: Is "Sign in with X" safe if I just call the `/me` API with the access token?**
Partially. It's the pattern that leads to token substitution: a token issued to another client may be accepted. Use OIDC and validate the ID token's `aud` = your client ID and the `nonce`. Key users by `(iss, sub)`, not email.

**Q: What's the difference between a scope and a role?**
Scope limits what the *client application* may do on the user's behalf and is consented by the user. Role/permission is what the *user* is allowed to do in the system. Effective permission is the intersection, and the RS enforces both.

---

## 15. Compact mental model

```text
OAuth 2.0 / OIDC
├── Roles: Resource Owner, Client (public | confidential), Authorization Server, Resource Server
├── Channels: front (browser, leaky): only codes | back (TLS POST): tokens
├── Grants
│   ├── Authorization Code + PKCE: every user-facing app (web, SPA via BFF, mobile via system browser)
│   ├── Client Credentials: service-to-service (private_key_jwt / mTLS, cache + single-flight)
│   ├── Device Code: TVs/CLIs (poll, authorization_pending/slow_down, phishing risk)
│   ├── Refresh Token: rotate + reuse detection (family revoke) or DPoP/mTLS-bound
│   ├── Token Exchange: on-behalf-of, down-scoping across services
│   └── DEPRECATED: Implicit (token in URL), Password (client sees password)
├── Bindings: state→CSRF | PKCE→code theft/injection | nonce→ID token replay | iss→mix-up | exact redirect_uri
├── Tokens
│   ├── Access: for the API (aud=API), short TTL, JWT (local verify) or opaque (introspect)
│   ├── Refresh: AS-only, opaque, hashed at rest, rotated, absolute lifetime
│   └── ID (OIDC): for the client (aud=client_id), proves authentication. Never send to APIs
├── RS validation: sig(JWKS,kid,alg allow-list) → iss → aud → exp → typ → scope → owns resource?
├── Browser: BFF (HttpOnly session, tokens server-side) > in-memory tokens + rotation/DPoP
└── AS design: Redis (codes, TTL, GETDEL) | Postgres (RT families, conditional UPDATE) | KMS keys
    └── JWKS rotation: publish → wait cache TTL → sign → wait max TTL → remove
```
