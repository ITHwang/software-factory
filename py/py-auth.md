# FastAPI Authentication

> Last updated: 2026-05-15

## TL;DR

Comparison-driven FastAPI auth guide. Names the architectural dimensions (identity source, token model, MFA layer, provider / user-store, wire format, signing algorithm), enumerates options per dimension, recommends per situation, prescribes a greenfield default. Assumes the reader knows OAuth 2.0, OIDC, JWT, and JWKS basics — this doc skips terminology and goes straight to architectural decisions.

**Use this when:**
- picking an auth architecture for a new FastAPI service
- comparing identity sources (email/password, Google OIDC, magic link, passkey, SSO)
- evaluating the JWT-vs-session trade-off
- choosing a provider (self-managed vs Cognito vs Auth0 vs Keycloak)
- debugging a suspected security misconfiguration on an auth path

**Don't use this for:**
- OAuth / OIDC / JWT terminology — assumed prior knowledge
- login UI / provider console configuration → upstream provider docs
- rate-limiting `/login` `/register` `/refresh` → [`./py-rate-limit.md`](./py-rate-limit.md)
- persistence of auth tables → [`./sqlalchemy.md`](./sqlalchemy.md)
- container wiring for the verifier → [`./dependency-injector.md#cognito-auth`](./dependency-injector.md#cognito-auth)

## Table of Contents

| Phase | Section |
|-------|---------|
| 1. Frame | [Core Tenets](#core-tenets), [Decision Axes](#decision-axes) |
| 2. Compare Options | [Identity Source Options](#identity-source-options), [Token Model Options](#token-model-options), [Provider / User-Store Options](#provider--user-store-options), [Wire Format](#wire-format), [Signing Algorithm](#signing-algorithm) |
| 3. Apply | [Situation-Based Recommendations](#situation-based-recommendations), [Greenfield Default Stack](#greenfield-default-stack) |
| 4. Build | [JWT Mechanics](#jwt-mechanics), [Refresh Token Rotation + Reuse Detection](#refresh-token-rotation--reuse-detection), [Password Hashing](#password-hashing) |
| 5. Secure | [Security Attacks](#security-attacks), [Cookies vs Authorization Header](#cookies-vs-authorization-header) |
| 6. Wire | [FastAPI Wire-Up](#fastapi-wire-up), [Cognito Worked Example](#cognito-worked-example) |
| 7. Survey | [Library Landscape](#library-landscape), [Evolution Path](#evolution-path) |
| 8. Operate | [Testing](#testing), [Production Checklist](#production-checklist), [Pitfalls](#pitfalls) |

## Core Tenets

Three principles that shape every decision below.

- **Authentication is not authorization.** The JWT carries identity; authorization policy lives in server code, not in the token claims. Treat scopes / groups inside the token as inputs to authorization, not as the authorization decision.
- **JWT's stateless-validation strength is its revocation weakness.** Mitigations are short access TTL + refresh rotation + reuse detection, not "make JWT revocable." If you need instant revocation more than you need statelessness, switch token models — don't bolt revocation onto a JWT.
- **Whatever the login method, the backend converges on one token shape.** Email/password, Google OIDC, magic link, passkey, SAML — every flow funnels into either an internally-issued JWT or one externally-trusted JWT/session shape. Downstream services must not branch on token format.

## Decision Axes

Eight factors. Each is independently selectable; the doc returns to each below.

1. **Identity source** — who proves identity? Self-hosted email/password, OIDC (Google etc.), magic link, passkey/WebAuthn, SSO (SAML or enterprise OIDC), API key for machine-to-machine.
2. **Token model** — JWT (stateless verification, weak revocation) or session (cookie + server store, instant revoke). A genuine architectural choice — not a row in a comparison table.
3. **MFA layer** — none, TOTP, WebAuthn-as-2FA, SMS-as-last-resort. Additive to any identity source.
4. **Provider / user-store** — self-managed (your DB) or managed IdP (Cognito, Auth0, Clerk, Stytch, Keycloak, WorkOS).
5. **Wire format** — `Authorization: Bearer` header (CSRF-immune; client must store) or cookie (`HttpOnly` makes XSS-safe; CSRF-sensitive without `SameSite`).
6. **Signing algorithm** — HS256 (symmetric, single backend) or RS256 / EdDSA (asymmetric, multi-service / JWKS publisher).
7. **Token lifecycle** — TTL × refresh strategy × revocation strategy. Short access TTL + refresh rotation + reuse detection is the standard JWT trade-off.
8. **Compliance / sovereignty** — vendor lock-in tolerance, GDPR / data-residency constraints, air-gapped requirements. Often the deciding factor between self-managed and managed IdP.

## Identity Source Options

Each identity source gets a sub-section with when-to-use, sequence diagram (where the flow is non-obvious), trade-offs, library pointer. The sources are not mutually exclusive — a real service usually combines two or three (e.g. email/password + Google OIDC + TOTP MFA).

### Email / Password (self-hosted)

When: full control over the user lifecycle; no vendor dependence; you accept building registration, verification, password reset, lockout, and breach response yourself. Greenfield default for consumer-facing services without a B2B SSO requirement.

Sequence diagram (registration + email verification):

```text
[ Frontend ──POST /register {email, password}──> Server ]
[                                                hash password (argon2id via pwdlib) ]
[                                                create user(verified=False) ]
[                                                generate random token (secrets.token_urlsafe(32)) ]
[                                                store SHA-256(token) + expiry in DB ]
[                                                send verification email ──> User mailbox ]
[ ────────────────────────────────────────────────────────────────────────────────── ]
[ User mailbox ──click link──> Server (GET /verify?token=<raw>) ]
[                              hash incoming token ]
[                              match against DB + check exp + check used_at IS NULL ]
[                              mark verified=True + mark token used_at=now ]
[                              redirect to login ]
```

Verification-token storage rule: **random opaque (NOT a JWT)**, SHA-256-hashed in DB (raw never persisted), single-use (`used_at` column), 24-hour TTL. Same pattern reused for password reset and magic link.

Trade-offs: maximum control, maximum build cost. Build registration, verification, login, password reset, account lockout, password rotation, and breach detection. Sustaining cost is non-trivial; weigh against a managed IdP if the team is small.

### OIDC (Google, Microsoft, Apple, GitHub, etc.)

When: users already have accounts at the OIDC provider; you want zero password-management cost for that segment; no B2B SSO requirement (else use the SSO option).

Sequence diagram (Google OIDC login, ending with own-JWT re-issuance):

```text
[ Frontend ──"Login with Google"──> Google (client_id, redirect_uri, scope=openid email profile) ]
[                                   user consents ]
[ Google ──authorization code──> Frontend (redirect_uri) ]
[ Frontend ──POST /auth/google {code}──> Server ]
[                                        exchange code for {access_token, id_token, refresh_token} ──> Google ]
[                                        verify id_token via Google JWKS: ]
[                                          - signature (RS256) ]
[                                          - iss = https://accounts.google.com ]
[                                          - aud = your Google client ID ]
[                                          - exp not passed ]
[                                        extract sub, email ]
[                                        get_or_create user in YOUR DB ]
[                                        issue YOUR JWT ──> Frontend ]
```

**Principle:** don't use Google's `id_token` directly on downstream requests. Verify it once at login, get-or-create the user, issue your own JWT. That re-issuance layer lets the backend control claim shape, TTL, and decoupling from Google's per-request availability.

Trade-offs: zero password management for OIDC users; but adding another provider (Apple, GitHub, Microsoft) means building each integration. For three or more providers, prefer SSO via a managed IdP and let the IdP federate.

### Magic Link (passwordless)

When: small services that want zero password management. First click logs the user in; subsequent visits reuse a long-lived session or refresh token. Pairs well with passkey for re-visits to skip the email round-trip.

Sequence diagram:

```text
[ Frontend ──POST /auth/magic {email}──> Server ]
[                                        generate random token (secrets.token_urlsafe(32)) ]
[                                        store SHA-256(token) + 15-min exp + user_id ]
[                                        send email with link /auth/magic/verify?token=<raw> ──> User ]
[ User ──click link──> Server (GET /auth/magic/verify?token) ]
[                      hash incoming token, look up by hash ]
[                      check exp + not used + not revoked ]
[                      mark used_at=now ]
[                      issue JWT or set session cookie ──> Frontend ]
```

Trade-offs: phishing-resistant when implemented correctly (link works once, expires fast); but awful UX in some email clients — Outlook ATP scanners pre-visit URLs and burn the single-use token before the user clicks. Always pair with a short retry path or a passkey re-visit option.

### Passkey / WebAuthn

When: new services where phishing-resistance and UX both matter; mobile-first apps; identity-paranoid sectors (banking, healthcare, government). The 2026 trend is passkey-primary with email/password as a fallback, not the other way around.

Sequence diagram (registration then login):

```text
[ Registration: ]
[ Frontend ──POST /auth/passkey/register-options──> Server ]
[ Server ──{challenge, rp, user, pubKeyCredParams}──> Frontend ]
[ Browser ──navigator.credentials.create──> Authenticator (Face ID / Windows Hello / hardware key) ]
[ Browser ──POST /auth/passkey/register {attestation}──> Server ]
[                                                        verify attestation ]
[                                                        store {credential_id, public_key, sign_count} ──> DB ]

[ Login: ]
[ Frontend ──POST /auth/passkey/auth-options {email}──> Server ]
[ Server ──{challenge, allowCredentials: [credential_ids]}──> Frontend ]
[ Browser ──navigator.credentials.get──> Authenticator (signs challenge with private key) ]
[ Browser ──POST /auth/passkey/auth {assertion}──> Server ]
[                                                  verify signature with stored public key ]
[                                                  update sign_count (replay protection) ]
[                                                  issue JWT ──> Frontend ]
```

Library: `py_webauthn` on the server side. Trade-offs: phishing-immune (private key never leaves the device); cross-device sync via Apple iCloud Keychain, Google Password Manager, or 1Password. Browser support is universal in 2026. The only real downside is bootstrapping users who don't yet have an authenticator on the device they're using right now — keep an email/password or magic-link fallback.

### SSO (SAML / Enterprise OIDC)

When: B2B contexts. Almost mandatory once enterprise customers appear. The customer's IdP (Okta, Azure AD, Google Workspace, Auth0-acting-as-IdP) authenticates the user; you only verify the resulting assertion and map it to a tenant.

Architectural picture: structurally identical to the OIDC sub-section above. SAML uses XML assertions instead of JWTs; enterprise OIDC uses JWTs. The verification step is the same shape — verify the assertion signature against the IdP's certs / JWKS, trust the claims, map to your user-and-tenant model.

Library landscape: `python3-saml` if you implement SAML directly; for enterprise OIDC the same OIDC libraries (`PyJWT` + `Authlib`) apply. In practice, it is usually cheaper to use a managed IdP (WorkOS, Auth0, Stytch B2B) that handles SAML + enterprise OIDC + SCIM provisioning in one place rather than maintain the SAML stack yourself.

### API Key (machine-to-machine)

When: server-to-server integrations, external partner APIs, CI tokens. Not for human-bearing tokens — use JWT or session.

Pattern: a long opaque random string; SHA-256-hashed in DB (raw never persisted); scoped permissions on each key; per-key rate-limit and per-key revocation. The key arrives in `Authorization: Bearer <key>` or a custom `X-Api-Key` header.

Trade-offs: simpler than JWT for service callers (no expiry math, no refresh flow); rotation requires the caller's cooperation (no automatic refresh). Cross-link to [`./py-rate-limit.md`](./py-rate-limit.md) for per-key throttling.

### MFA Layer (TOTP / WebAuthn-as-2FA / SMS)

**MFA is additive to any of the identity sources above — it is not an alternative.** Email/password + TOTP is two factors; Google OIDC + WebAuthn-as-2FA is two factors; passkey alone is one strong factor that you may or may not want to layer another on top of.

- **TOTP (authenticator app)** — `pyotp` does the heavy lifting; user scans a QR code; the app generates a 6-digit code every 30 seconds; the server verifies within a 30-second ± 1-window tolerance. The mainstream MFA second factor.
- **WebAuthn-as-2FA** — same library (`py_webauthn`); phishing-immune; preferred over TOTP when the audience has authenticators on their devices.
- **SMS OTP** — vulnerable to SIM-swap attacks; NIST discourages. Use only when nothing else is operationally available, and document the risk acceptance.

## Token Model Options

The choice between JWT and session is structural — it changes how every authenticated request is processed. Treat it as a first-class decision, not as a row in a comparison table.

### JWT (stateless validation)

Pattern: the backend issues a signed token at login; verifies the signature + iss + aud + exp on every request using only a public key / shared secret + an explicit algorithm allowlist; no DB read on the auth path.

Sequence diagram (API call + access expiry + refresh):

```text
[ Frontend ──GET /api/me  Authorization: Bearer <access>──> Server ]
[ Server ──200 OK──> Frontend ]
[ ............ time passes, access expires ............ ]
[ Frontend ──GET /api/me  Authorization: Bearer <expired access>──> Server ]
[ Server ──401──> Frontend ]
[ Frontend ──POST /auth/refresh  Cookie: refresh=<refresh>──> Server ]
[                                                              verify refresh in DB (hash match, not used, not revoked, not expired) ]
[                                                              issue new access ]
[                                                              ROTATE refresh: mark old.used_at=now; persist new ]
[ Server ──200 {access}  Set-Cookie: refresh=<new>; HttpOnly; Secure; SameSite=Lax──> Frontend ]
[ Frontend ──GET /api/me  Authorization: Bearer <new access>──> Server (retry original request) ]
[ Server ──200 OK──> Frontend ]
```

**Important:** refresh is frontend-driven (the client receives a 401, calls `/auth/refresh`, retries). The server never pushes a new access token.

Strengths: stateless verification (no DB hit per request), horizontally scalable, easy multi-service via shared public key (RS256 + JWKS), tokens portable across mobile and SPA clients.

Weakness: **revocation is hard.** Once issued and exp-bounded, a JWT can't be invalidated mid-life without a per-request DB check, which defeats the stateless gain. Standard mitigations:

| Option | How | Trade-off |
|---|---|---|
| Short access TTL (5–15 min) | Bound the exposure window | Most common; usually enough |
| Token blocklist (`jti` in Redis) | Per-request Redis check | Defeats stateless verification |
| `tokens_valid_after` on user | Per-request user DB read | Lighter than blocklist; still per-request |

If you genuinely need instant logout across a fleet of services, switch to the session model — it's the simpler answer.

### Session (cookie + Redis)

Pattern: the backend issues an opaque session ID at login; stores `{user_id, expires_at, fingerprint, ...}` server-side (Redis, DB); a cookie carries the ID; each authenticated request looks the session up and refreshes its sliding expiry.

When: same-domain SPA; monolith; backend-for-frontend; instant-logout requirement; small distributed teams where Redis ops are routine. Common in Rails / Django classic setups; less common in 2026 FastAPI services but still legitimate.

Trade-offs vs JWT:

- Pro: instant revoke — delete the session row.
- Pro: smaller cookie — only the ID, not the claims.
- Pro: easy "log me out of all devices" — delete all sessions for `user_id`.
- Pro: easy "force re-auth after password change" — invalidate all sessions atomically.
- Con: every authenticated request hits Redis / DB (~1 ms).
- Con: cross-domain SPA or mobile clients — cookies don't traverse cleanly; you fall back to JWT or to session-ID-in-header (which loses the `HttpOnly` benefit).
- Con: multi-region — session store needs replication and careful failover semantics.

### Comparison

| Concern | JWT | Session |
|---|---|---|
| Stateless verification | Yes | No |
| Per-request DB / Redis hit | No | Yes |
| Instant revoke | No (bounded by TTL) | Yes |
| Horizontal scale | Trivial | Needs shared session store |
| Mobile / cross-domain | Native | Awkward (cookie domain limits) |
| Multi-service | Trivial via shared JWKS | Each service needs session-store access |
| "Log out of all devices" | Per-user `tokens_valid_after` (extra DB hit) | Delete all rows for user_id |

Pick JWT for: stateless services, mobile clients, multi-service architectures, public APIs. Pick session for: same-domain SPA monolith with an instant-logout requirement, or where Redis is already a hard dependency and statelessness isn't a goal.

## Provider / User-Store Options

Five provider architectures. The user's three-architecture comparison goes verbatim into A–D; E is a brief survey of the 2026 managed-IdP landscape.

### Architecture A — Self-managed (your DB)

```text
[ Frontend ──login──> FastAPI Backend ]
[                     verify password (argon2id) or verify OIDC id_token ]
[                     load user from YOUR Postgres ]
[                     issue YOUR JWT ──> Frontend ]
[ Frontend ──API──> Backend ──verify YOUR JWT (HS256 or RS256)──> 200 ]
```

- **Backend trusts:** itself.
- **User store:** your Postgres.
- **Identity sources:** any (email/password, OIDC, magic link, passkey, SAML) — all converge on your JWT.

### Architecture B — Direct OIDC (Google / GitHub / etc.) + own JWT re-issuance

```text
[ Frontend ──login──> Google ──id_token──> Frontend ]
[ Frontend ──send id_token──> FastAPI Backend ]
[                              fetch JWKS from Google ]
[                              verify id_token (iss, aud, exp, signature) ]
[                              get_or_create user in YOUR DB ]
[                              issue YOUR JWT ──> Frontend ]
[ user DB: you manage ]
```

- **Backend trusts:** Google for identity verification at login only; itself for the issued JWT on every subsequent request.
- **User store:** your Postgres.

### Architecture C — Cognito as central IdP

```text
[ Frontend ──login──> Cognito (user pool) ]
[                     - self-service registration ]
[                     - password management ]
[                     - MFA ]
[ Cognito ──Cognito access_token──> Frontend ]
[ Frontend ──send Cognito token──> FastAPI Backend ]
[                                   fetch JWKS from Cognito ]
[                                   verify Cognito token (iss = cognito-idp.{region}.amazonaws.com/{user_pool_id}, client_id) ]
[ user DB: Cognito owns auth identity; your DB has user_id mapping joined on sub ]
```

- **Backend trusts:** Cognito only.
- **User store:** Cognito + your DB for domain data joined on `sub`.
- **Own JWT re-issuance:** optional. Most setups use Cognito tokens directly.

### Architecture D — Cognito + federation

```text
[ Frontend ──login──> Cognito ──federate──> Google ──Google id_token──> Cognito ]
[ Cognito ──Cognito access_token──> Frontend ]
[ Frontend ──send Cognito token──> FastAPI Backend ]
[                                   fetch JWKS from Cognito ]
[                                   verify Cognito token (Google tokens never reach backend) ]
```

- **Backend trusts:** Cognito only. Google's tokens are invisible to your code.
- **Verification logic:** identical to Architecture C.
- **Adding Google login** = a Cognito console toggle. Same for Apple, Facebook, SAML.

### Architecture E — Other managed IdPs

Brief survey of the 2026 managed-IdP landscape. All slot into Architecture C / D structurally — backend trusts the IdP and verifies its JWTs or SAML assertions. The choice between them is operational, not architectural.

| Provider | Best for | Notes |
|---|---|---|
| **Auth0** | Enterprise OIDC + SAML in one place; broad SDK support | Strong B2B story; per-MAU pricing |
| **Clerk** | React / Next.js apps; passkey-first UI | Modern UX; per-MAU pricing |
| **Stytch** | Passwordless-first (magic link, passkey, OTP) | Strong passwordless primitives |
| **Keycloak** | Self-hosted IdP | Open-source; ops-heavy; full control; on-prem-friendly |
| **WorkOS** | B2B SSO + SCIM as a service | Specialized B2B; you stay on your own user store |
| **Cognito** | AWS-native | Tight AWS IAM integration; per-MAU pricing |

### Comparison

| Axis | A. Self-managed | B. Direct OIDC + own JWT | C. Cognito only | D. Cognito + federation | E. Other managed IdP |
|---|---|---|---|---|---|
| Backend trusts | itself | Google | Cognito | Cognito | The IdP |
| JWKS source | own | Google | Cognito | Cognito | The IdP |
| User store | your DB | your DB | Cognito + DB join | Cognito + DB join | IdP + DB join |
| Self-hosted email/password | build | build | provided | provided | usually provided |
| Adding another social IdP | build per-IdP | build per-IdP | (not applicable) | console toggle | console toggle |
| MFA / password reset | build | build | provided | provided | provided |
| Cost | none | none | per-MAU | per-MAU | per-MAU |
| Vendor lock-in | none | none | strong (AWS) | strong (AWS) | strong (vendor-specific) |
| Best for | small + full control | consumer + Google audience | AWS-native quickstart | AWS-native + multi-IdP | per-vendor strength |

## Wire Format

Where the credential travels. Two choices for human-bearing tokens; one for machine-to-machine.

- **`Authorization: Bearer <token>` header** — CSRF-immune (the browser does not auto-attach the header to cross-site requests); but the client must store the token, and `localStorage` is XSS-readable. Keep access tokens in memory and rely on the refresh path to rehydrate on reload.
- **Cookie** — `HttpOnly` makes the value unreadable from JS (XSS-resistant); but the browser auto-attaches it to every same-origin request, which is the CSRF attack surface. `SameSite=Lax` or `Strict` closes most of that gap.

Greenfield default (consumer-facing): `Authorization: Bearer` for the access token (CSRF-immune, in-memory); `HttpOnly + Secure + SameSite=Lax` cookie for the refresh token, scoped `path="/auth/refresh"` so it only attaches to the refresh endpoint. Both flags matter — neither defends alone.

## Signing Algorithm

| Algorithm | Type | Use when |
|---|---|---|
| **HS256** | Symmetric (shared secret) | Single backend issues + verifies; no JWKS publication |
| **RS256** | Asymmetric (RSA key pair) | Multiple services verify (shared public key via JWKS); standard for Cognito, Google, Auth0 |
| **EdDSA** | Asymmetric (Ed25519) | Same as RS256, smaller keys, faster signing; supported by PyJWT 2.x |

**Verification rule:** always pass `algorithms=[...]` explicitly to `jwt.decode`. Never read the `alg` field from the token header — that's the `alg=none` attack surface.

JWKS endpoint structure: a JSON document at `/.well-known/jwks.json` returning `{"keys": [{...}]}`. Key rotation = publisher adds a new key, retires the old; `PyJWKClient` looks up the right key by `kid`. Library handles the lookup; don't roll it by hand.

## Situation-Based Recommendations

| Situation | Recommended stack |
|---|---|
| Greenfield consumer SaaS, no AWS preference | **A or B** (self-managed) + email/password + Google OIDC + JWT + refresh rotation. The greenfield default below. |
| Greenfield, AWS-native, fast start | **D** (Cognito + federation) + Cognito-provided email/password + Google federation + Cognito JWT directly. |
| B2B with SSO requirement | **E** (WorkOS or Auth0) + SAML / enterprise OIDC + vendor JWT or your re-issued JWT. WorkOS if you only need SSO; Auth0 if you also need consumer-facing login in one place. |
| Mobile-first (iOS / Android first-class) | **A or B** + JWT (cookies don't traverse cleanly on mobile) + refresh rotation + passkey (uses platform authenticators). |
| Air-gapped / on-prem | **A** (self-managed) + email/password + TOTP MFA + JWT (no external IdP dependency). If a managed IdP is required but cloud is forbidden, use **Keycloak**. |
| Internal admin tool, monolith, instant-logout requirement | **A** + **session model** (Redis-backed) + email/password + TOTP MFA. JWT's revocation weakness is not worth the complexity in a monolith with low scale and a hard "log them out now" requirement. |
| Service-to-service / partner integration | **API key** (long random + SHA-256 hash in DB + per-key rate-limit + per-key revoke). Not JWT, not OIDC. |

## Greenfield Default Stack

Prescriptive — for a new consumer-facing FastAPI service being built today (the "no AWS preference" case above).

- **Architecture:** **B** (Direct OIDC + own JWT re-issuance). No vendor lock-in; control over claim shape; can pivot to D later if AWS-native becomes a requirement.
- **Identity sources:** self-hosted email/password (with email verification) + Google OIDC. Add Apple, GitHub on demand.
- **MFA:** TOTP via `pyotp` as opt-in; promote to required for sensitive accounts. WebAuthn-as-2FA when the audience supports it.
- **Token model:** JWT.
- **Password hash:** **argon2id via `pwdlib`**. Not bcrypt-as-new, not PBKDF2-as-new, never plain MD5 / SHA-256.
- **Access token:** 15-min TTL; HS256 for single backend (RS256 if you'll ever publish JWKS to other services).
- **Refresh token:** 30-day TTL; DB-stored with rotation + reuse detection (family revoke).
- **Wire:** `Authorization: Bearer` for the access token (CSRF-immune; client memory); `HttpOnly + Secure + SameSite=Lax` cookie for the refresh token, scoped `path="/auth/refresh"`.
- **Algorithms:** always pass `algorithms=[...]` to `jwt.decode` — never trust the token's own `alg` header.
- **Rate limits:** `/login`, `/register`, `/forgot-password`, `/verify`, `/refresh` all rate-limited (see [`./py-rate-limit.md`](./py-rate-limit.md)).
- **Email-verification token:** random opaque (`secrets.token_urlsafe(32)`), SHA-256-hashed in DB, single-use, 24-hour TTL. **Never a JWT.**

## JWT Mechanics

Load-bearing verification snippet — the algorithms allowlist is the difference between a verified token and an `alg=none` exploit:

```python
from jwt import PyJWKClient
import jwt

jwks = PyJWKClient("https://<issuer>/.well-known/jwks.json")
signing_key = jwks.get_signing_key_from_jwt(token)
claims = jwt.decode(
    token,
    signing_key.key,
    algorithms=["RS256"],  # NEVER read alg from the token header
    audience="<expected-aud>",  # required for ID tokens
    issuer="<expected-iss>",  # required
    options={"require": ["exp", "iss", "sub", "aud"]},
)
```

Rules:

- **Always** pass `algorithms=[...]` explicitly.
- **Always** verify `iss`, `aud` (or manual `client_id` for Cognito access tokens), and `exp`.
- **Never** read `alg` from the token header to decide the algorithm — that's how `alg=none` slips through.
- HS256 for single-backend setups; RS256 / EdDSA when JWKS publication is needed.

## Refresh Token Rotation + Reuse Detection

Rotation alone closes the "stolen access token" window; reuse detection closes the "stolen refresh token" window. Both are necessary.

Sequence diagram (theft detection via family revoke):

```text
[ Attacker steals refresh_A from the user's machine ]
[ Attacker ──POST /auth/refresh  refresh_A──> Server ]
[                                              mark refresh_A.used_at=now ]
[                                              issue access_B + refresh_B (same family_id, parent_id=A) ]
[ Server ──{access_B, refresh_B}──> Attacker ]
[ ............ time passes ............ ]
[ Real user ──POST /auth/refresh  refresh_A──> Server ]
[                                               look up refresh_A → already used (used_at IS NOT NULL) ]
[                                               THEFT DETECTED ]
[                                               revoke ENTIRE family_id (refresh_A, refresh_B, future descendants) ]
[                                               force re-login ]
```

DB schema (load-bearing — the family fields are non-obvious):

```sql
CREATE TABLE refresh_tokens (
  id UUID PRIMARY KEY,
  token_hash TEXT NOT NULL UNIQUE,
  user_id UUID NOT NULL,
  family_id UUID NOT NULL,        -- groups tokens from one login session
  parent_id UUID,                 -- which token this rotated from
  used_at TIMESTAMPTZ,            -- single-use enforcement
  revoked_at TIMESTAMPTZ,         -- explicit revoke (family-wide on reuse)
  expires_at TIMESTAMPTZ NOT NULL
);
```

On every refresh: look up by `token_hash`; if `used_at IS NOT NULL`, revoke the entire `family_id` and 401. Otherwise mark used, issue a new token with the same `family_id` and `parent_id = this.id`.

## Password Hashing

- Hashing must be **slow** by design — that's what defeats brute force.
- History: MD5 / SHA-1 unsafe; plain SHA-256 too fast; PBKDF2 legacy-acceptable; bcrypt still safe; **argon2id is OWASP's #1 recommendation and the greenfield default**.
- Library: `pwdlib` (argon2id by default). `passlib(bcrypt)` is legacy-acceptable but `passlib` has Py 3.13 compatibility issues — migrate.

Minimal usage:

```python
from pwdlib import PasswordHash

hasher = PasswordHash.recommended()  # argon2id
hashed = hasher.hash(password)
ok = hasher.verify(password, hashed)
```

## Security Attacks

Compact reference. Every row pairs an attack with the defense the rest of this doc has already named.

| Attack | What | Defense |
|---|---|---|
| **XSS** | Attacker JS reads `localStorage` tokens | Input escape, CSP header, `HttpOnly` cookie for refresh |
| **CSRF** | Logged-in user tricked into a cross-site state-changing request | `SameSite=Strict/Lax`, CSRF token, Origin/Referer check, OR use `Bearer` header (CSRF-immune) |
| **Credential stuffing** | Leaked email/password pairs tried at scale | Rate limit, CAPTCHA, HIBP-style breached-password check, MFA |
| **Brute force** | One account, many guesses | Rate limit + progressive lockout; argon2id slow hash |
| **Token theft** | Attacker exfiltrates a valid token | HTTPS, `HttpOnly` cookie for refresh, refresh rotation + reuse detection, short access TTL |
| **`alg=none`** | Attacker changes JWT header `alg` to `none` | `jwt.decode(..., algorithms=["HS256"])` — **always** pass `algorithms=` explicitly |
| **Refresh reuse** | Stolen refresh used after legitimate rotation | Family-based revoke on reuse |

## Cookies vs Authorization Header

| Concern | `Authorization: Bearer` | Cookie |
|---|---|---|
| CSRF | Immune | Vulnerable without `SameSite` |
| XSS exposure | Token in JS memory; XSS reads it | `HttpOnly` cookie unreadable by JS |
| Client storage | Memory or `localStorage` (XSS risk) | Browser-managed |
| Auto-send to subdomains | No | Yes (domain scope) |
| Native mobile | First-class | Awkward (cookie jar semantics) |

Greenfield default: `Authorization: Bearer` for the access token; `HttpOnly + Secure + SameSite=Lax` cookie for the refresh token.

Cookie flags:

- `HttpOnly` — JS can't read → defends XSS.
- `Secure` — HTTPS only.
- `SameSite=Lax` — cross-site GETs allowed, cross-site POSTs blocked → CSRF defense.
- `SameSite=Strict` — even safer, but breaks navigation flows that need the cookie on first arrival from another origin.

CSRF token: necessary if you're cookie-based with cross-site scenarios; you can skip it if you're same-domain + `SameSite=Strict`.

## FastAPI Wire-Up

- **Auth boundary:** routes use `Depends(get_current_user)`; routes **never** decode JWT inline. This is one of the two documented FastAPI-`Depends` exceptions to the DI container rule (see [`./py-backend-architecture.md#services-and-fastapi`](./py-backend-architecture.md#services-and-fastapi)).
- **Domain error type:** `AuthenticationError` (a `CustomError` subclass — see [`./py-guidelines.md`](./py-guidelines.md) Exception Handling). Verifiers raise it; the global exception handler maps it to 401.
- **Port naming:** the capability port is `JWTVerifier` (provider-agnostic). Concrete adapters are `CognitoJWTVerifier`, `GoogleOIDCVerifier`, `Auth0JWTVerifier`. The test double is `MockJWTVerifier`.
- **`OAuth2PasswordBearer` clarification:** despite the name, this class is *just* an `Authorization: Bearer` header parser with OpenAPI metadata attached. It is NOT a directive to implement the OAuth 2.0 Password Grant (ROPC, removed in OAuth 2.1). For most services prefer `HTTPBearer` — smaller, clearer name.

Recommended package layout:

```text
app/auth/
  ├── router.py          # /login, /register, /verify, /refresh, /logout
  ├── service.py         # login, register, refresh orchestration
  ├── jwt.py             # JWTVerifier port + concrete adapter
  ├── password.py        # pwdlib wrapper
  ├── dependencies.py    # get_current_user, require_scope, require_group
  ├── oauth_google.py    # Authlib-based OIDC client
  └── schemas.py         # request/response Pydantic models
```

### `get_current_user` pattern

Load-bearing — preserved verbatim from the prior Cognito doc, generalized to the `JWTVerifier` port name. This is the only place in the codebase that touches `Authorization` headers and `jwt.decode`.

```python
# dependencies.py
from typing import Annotated

from dependency_injector.wiring import Provide, inject
from fastapi import Depends, HTTPException, Security, status
from fastapi.security import HTTPAuthorizationCredentials, HTTPBearer, SecurityScopes

from .auth import AuthenticationError, CurrentUser
from .containers import Container
from .jwt import JWTVerifier

bearer = HTTPBearer(scheme_name="AccessToken", auto_error=False)


@inject
async def get_current_user(
    security_scopes: SecurityScopes,
    credentials: Annotated[HTTPAuthorizationCredentials | None, Depends(bearer)],
    verifier: Annotated[JWTVerifier, Depends(Provide[Container.jwt_verifier])],
) -> CurrentUser:
    if credentials is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Missing bearer token",
            headers={"WWW-Authenticate": _challenge(security_scopes)},
        )
    try:
        user = verifier.verify(credentials.credentials)
    except AuthenticationError as exc:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail=str(exc),
            headers={"WWW-Authenticate": _challenge(security_scopes)},
        ) from exc
    if not set(security_scopes.scopes).issubset(user.scopes):
        raise HTTPException(
            status_code=status.HTTP_403_FORBIDDEN,
            detail="Insufficient scope",
            headers={"WWW-Authenticate": _challenge(security_scopes)},
        )
    return user


def require_group(group: str):
    async def dependency(
        user: Annotated[CurrentUser, Security(get_current_user)],
    ) -> CurrentUser:
        if group not in user.groups:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN, detail="Missing group"
            )
        return user

    return dependency
```

### Route usage

Routes stay thin: pass typed `CurrentUser` to the service; the service never touches `request.state`.

```python
# routes.py
from typing import Annotated

from dependency_injector.wiring import Provide, inject
from fastapi import APIRouter, Depends, Security

from .auth import CurrentUser
from .containers import Container
from .dependencies import get_current_user, require_group
from .services import DocumentService

router = APIRouter(prefix="/documents", tags=["documents"])


@router.get("/{document_id}")
@inject
async def read_document(
    document_id: str,
    user: Annotated[CurrentUser, Security(get_current_user, scopes=["documents:read"])],
    service: Annotated[DocumentService, Depends(Provide[Container.document_service])],
) -> dict: ...


@router.post("/{document_id}/reindex")
@inject
async def reindex_document(
    document_id: str,
    user: Annotated[CurrentUser, Depends(require_group("ADMIN"))],
    service: Annotated[DocumentService, Depends(Provide[Container.document_service])],
) -> dict: ...
```

### Request-context middleware

Lightweight per-request metadata (request ID, tenant domain). Middleware writes it on `request.state` at the boundary; a dependency exposes a typed `RequestContext` to routes that ask for it. Services receive `RequestContext` or `CurrentUser` as explicit arguments — they never read from `request.state` themselves.

```python
# request_context.py
from dataclasses import dataclass
from typing import Annotated
from uuid import uuid4

from fastapi import Depends, Request

from .auth import CurrentUser
from .dependencies import get_current_user


@dataclass(frozen=True)
class RequestContext:
    request_id: str
    tenant_domain: str | None
    user: CurrentUser


async def request_context_middleware(request: Request, call_next):
    request_id = request.headers.get("x-request-id") or str(uuid4())
    request.state.request_id = request_id
    request.state.tenant_domain = request.headers.get("x-tenant-domain")

    response = await call_next(request)
    response.headers["x-request-id"] = request_id
    return response


async def get_request_context(
    request: Request,
    user: Annotated[CurrentUser, Depends(get_current_user)],
) -> RequestContext:
    return RequestContext(
        request_id=request.state.request_id,
        tenant_domain=request.state.tenant_domain,
        user=user,
    )
```

Register the middleware at app creation:

```python
from fastapi import FastAPI

from .request_context import request_context_middleware


def create_app() -> FastAPI:
    app = FastAPI()
    app.middleware("http")(request_context_middleware)
    return app
```

`request.state` is appropriate at the HTTP boundary. It is not appropriate inside repositories, clients, or domain services — pass `CurrentUser`, `tenant_domain`, or a typed `RequestContext` explicitly.

## Cognito Worked Example

This section preserves the AWS Cognito verifier as the canonical OIDC-provider verifier shape. Swapping the provider (Cognito → Google → Auth0 → Keycloak) means changing the JWKS URL + the `iss` value + the audience-check logic; the rest of the shape is identical.

Domain types — preserved verbatim from the prior doc.

```python
# auth.py
from dataclasses import dataclass
from typing import Any


class AuthenticationError(Exception):
    """Raised by any JWTVerifier adapter when verification fails.

    Carried up to the global FastAPI exception handler, which maps it to 401.
    Subclass of CustomError in real codebases — see py-guidelines.md.
    """


@dataclass(frozen=True)
class CurrentUser:
    subject: str
    username: str
    scopes: frozenset[str]
    groups: frozenset[str]
    claims: dict[str, Any]
```

Settings — passive data; the container assembles the verifier from these.

```python
# settings.py
from pydantic_settings import BaseSettings, SettingsConfigDict


class CognitoSettings(BaseSettings):
    model_config = SettingsConfigDict(env_prefix="COGNITO_")

    region: str
    user_pool_id: str
    app_client_id: str
    token_use: str = "access"
    jwks_cache_ttl_seconds: int = 300
    jwks_timeout_seconds: float = 5.0
    leeway_seconds: int = 30

    @property
    def issuer(self) -> str:
        return f"https://cognito-idp.{self.region}.amazonaws.com/{self.user_pool_id}"

    @property
    def jwks_url(self) -> str:
        return f"{self.issuer}/.well-known/jwks.json"
```

`CognitoJWTVerifier` — preserved verbatim. Load-bearing: the `algorithms=["RS256"]` allowlist, the explicit `iss` pin, the manual `client_id` check (Cognito access tokens use `client_id`, not `aud`), the JWKS cache TTL, and the leeway are all security-critical.

```python
# jwt.py
import jwt
from jwt import PyJWKClient
from jwt.exceptions import InvalidTokenError, PyJWKClientError

from .auth import AuthenticationError, CurrentUser
from .settings import CognitoSettings


class CognitoJWTVerifier:
    def __init__(self, settings: CognitoSettings) -> None:
        self._settings = settings
        self._jwks = PyJWKClient(
            settings.jwks_url,
            cache_jwk_set=True,
            lifespan=settings.jwks_cache_ttl_seconds,
            cache_keys=True,
            timeout=settings.jwks_timeout_seconds,
        )

    def verify(self, token: str) -> CurrentUser:
        if not token:
            raise AuthenticationError("Missing bearer token")

        try:
            signing_key = self._jwks.get_signing_key_from_jwt(token)
            claims = jwt.decode(
                token,
                signing_key.key,
                algorithms=["RS256"],
                issuer=self._settings.issuer,
                options={
                    "require": ["exp", "iss", "sub", "token_use"],
                    "verify_aud": False,
                },
                leeway=self._settings.leeway_seconds,
            )
        except (InvalidTokenError, PyJWKClientError) as exc:
            raise AuthenticationError("Invalid Cognito token") from exc

        token_use = claims.get("token_use")
        if token_use != self._settings.token_use:
            raise AuthenticationError("Invalid Cognito token use")

        client_id = claims.get("client_id") or claims.get("aud")
        if client_id != self._settings.app_client_id:
            raise AuthenticationError("Token was not issued for this app client")

        scope = claims.get("scope", "")
        groups = claims.get("cognito:groups", [])

        return CurrentUser(
            subject=claims["sub"],
            username=claims.get("username", claims["sub"]),
            scopes=frozenset(scope.split()) if isinstance(scope, str) else frozenset(),
            groups=frozenset(groups if isinstance(groups, list) else []),
            claims=claims,
        )
```

**Why manual `client_id` validation:** Cognito access tokens identify the app client in `client_id`; ID tokens use `aud`. API backends should normally accept access tokens and reject ID tokens unless the endpoint explicitly needs identity-token semantics.

**Why hard-coded `algorithms=["RS256"]`:** never compute the allowed algorithms from the token header — that is the `alg=none` attack vector. Cognito user pools sign with RS256; the allowlist is pinned next to the trusted issuer.

A `CognitoAdminClient` exists for operational workflows (group membership management, user listing) and is wired separately as a `Factory`. Its surface is SDK-shaped and not load-bearing for the auth path; see the wiring section below.

Container wiring (the verifier as `Singleton`, the `cognito-idp` boto client as `Resource`, the admin client as `Factory`) lives in [`./dependency-injector.md#cognito-auth`](./dependency-injector.md#cognito-auth). The `CognitoJWTVerifier` slots in as the concrete `JWTVerifier` adapter behind the port.

See also: [AWS Cognito JWT verification](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-tokens-verifying-a-jwt.html), [Cognito access token claims](https://docs.aws.amazon.com/cognito/latest/developerguide/amazon-cognito-user-pools-using-the-access-token.html), [PyJWT JWKS usage](https://pyjwt.readthedocs.io/en/latest/usage.html#retrieve-rsa-signing-keys-from-a-jwks-endpoint), [FastAPI security tools](https://fastapi.tiangolo.com/reference/security/), [FastAPI OAuth2 scopes](https://fastapi.tiangolo.com/advanced/security/oauth2-scopes/).

## Library Landscape

| Library | Purpose | License | 2026 status | Use when |
|---|---|---|---|---|
| `PyJWT` | JWT encode/decode + JWKS client | MIT | Active (v2.12.1) | Primary JWT library |
| `Authlib` | OAuth 2.0 + OIDC client/server toolkit | BSD-3-Clause | Active (v1.7.2) | OIDC client flow; resource-server validation with provider-aware helpers |
| `pwdlib` | Password hashing (argon2id default) | MIT | Active | Greenfield default |
| `pyotp` | TOTP MFA | MIT | Active | TOTP second factor |
| `py_webauthn` | WebAuthn / Passkey server | BSD-3-Clause | Emerging | Passkey support |
| `python3-saml` | SAML SP / IdP | MIT | Active | Custom SAML integration |
| `fastapi-users` | Full auth solution | MIT | **Maintenance mode** (security only; no new features) | Prototypes; weigh maintenance risk |
| ~~`python-jose`~~ | JWT (legacy) | MIT | **Deprecated** — 2021 last release; Py 3.10+ compatibility broken | **Do not use** — migrate to PyJWT |
| ~~`passlib`~~ | Password hash (legacy) | BSD-3-Clause | **Slowing** — Py 3.13 compatibility issues | Migrate to `pwdlib` for new code |

## Evolution Path

Incremental adoption order — the order in which a real product typically grows its auth surface.

1. Email/password + email verification (random opaque token, 24h TTL).
2. Add Google OIDC (or another social IdP).
3. Add TOTP MFA as opt-in.
4. Add Passkey (eventually allows email/password to retire as a primary factor).
5. Add SAML / enterprise OIDC SSO when B2B customers appear.
6. Add API key when machine-to-machine integrations appear.

## Testing

Mock the verifier at the port boundary (`MockJWTVerifier` — the test-double name for the `JWTVerifier` port). Never call Google / Cognito / fetch JWKS in unit tests.

```python
# tests/test_documents_api.py
from collections.abc import AsyncIterator
from dataclasses import dataclass

import pytest_asyncio
from dependency_injector import providers
from httpx import ASGITransport, AsyncClient

from app.auth import AuthenticationError, CurrentUser
from app.containers import Container
from app.main import create_app_with_container


class MockJWTVerifier:
    def __init__(self, user: CurrentUser) -> None:
        self.user = user

    def verify(self, token: str) -> CurrentUser:
        if token == "bad":
            raise AuthenticationError("invalid")
        return self.user


@dataclass
class ApiContext:
    client: AsyncClient
    user: CurrentUser


@pytest_asyncio.fixture
async def api_context() -> AsyncIterator[ApiContext]:
    user = CurrentUser(
        subject="user-123",
        username="user-123",
        scopes=frozenset({"documents:read"}),
        groups=frozenset({"MEMBER"}),
        claims={},
    )
    container = Container()
    container.jwt_verifier.override(providers.Object(MockJWTVerifier(user)))

    app = create_app_with_container(container)
    transport = ASGITransport(app=app)
    async with AsyncClient(transport=transport, base_url="http://test") as client:
        yield ApiContext(client=client, user=user)

    container.jwt_verifier.reset_override()


async def test_read_document_accepts_valid_token(api_context: ApiContext) -> None:
    response = await api_context.client.get(
        "/documents/doc-123",
        headers={"Authorization": "Bearer test-token"},
    )

    assert response.status_code == 200
```

Required test scenarios:

- Missing `Authorization` header → 401.
- Invalid / expired / tampered token → 401.
- Valid token missing the required scope → 403.
- Valid token missing the required group → 403.
- Request-ID middleware echoes `x-request-id` in the response.
- Service receives the typed `CurrentUser`, not raw claims.

Cross-link to [`./py-tests.md`](./py-tests.md).

## Production Checklist

| Item | Default |
|---|---|
| JWT algorithms allowlist | `jwt.decode(..., algorithms=["HS256"])` or `["RS256"]`; never trust the token header |
| Password hash | argon2id via `pwdlib` |
| Refresh token | DB-stored, rotation + reuse detection (family revoke) |
| Auth endpoint rate limits | `/login`, `/register`, `/forgot-password`, `/verify`, `/refresh` |
| Refresh cookie | `HttpOnly + Secure + SameSite=Lax`, `path="/auth/refresh"` |
| HTTPS enforced | HSTS header |
| Email-verification token | random opaque, SHA-256-hashed in DB, single-use, ≤24h TTL |
| Route auth boundary | `Depends(get_current_user)`; routes never decode JWT inline |
| OIDC `id_token` verification | `iss`, `aud`, `exp` all checked; `algorithms=["RS256"]` explicit |
| JWKS cache | TTL between 5 min and 1 h; refresh on key rotation; don't fetch per request |
| MFA option | TOTP via `pyotp` for users opting in |
| Auth failures logged | Without logging raw tokens |
| CORS | Configured separately — not by the auth middleware |
| Admin API calls | Never on the request path; verify the JWT, trust the claims |

## Pitfalls

- **`jwt.decode` without `algorithms=`** — the `alg=none` attack vector. Always pass the allowlist explicitly.
- **Trusting unverified claims** — never read identity / groups / scopes / tenant from a token until signature + iss + aud + exp have all passed.
- **Cognito ID token vs access token confusion** — ID tokens use `aud`; access tokens use `client_id`. Pick what the endpoint expects; reject the other.
- **JWT for email verification or password reset** — revocation is awkward; use opaque random + DB-stored hash + `used_at`.
- **Refresh rotation without reuse detection** — half the security gain; a stolen old refresh stays valid until natural expiry.
- **Caching token claims longer than `exp`** — extends the credential's effective lifetime past its design.
- **Reading user context from `request.state` in services / repositories** — passes hidden state through the call graph. Pass typed `CurrentUser` explicitly.
- **Per-request admin-API calls** (Cognito `GetUser`, Auth0 management API, etc.) — admin APIs are not for the auth path. Verify the JWT; trust the claims.
- **`SameSite=None` without understanding the trade-off** — opens cross-site cookie attachment. Almost never the right answer for auth cookies.
- **`localStorage` for access tokens** — XSS-readable. Use memory or an `HttpOnly` cookie.
- **MD5 / SHA-1 / plain SHA-256 password hash** — either broken or too fast to be safe.
- **Mixing identity sources without converging on one downstream token shape** — services then need to handle N different token formats. Always converge on either your own JWT or one trusted IdP's JWT.
