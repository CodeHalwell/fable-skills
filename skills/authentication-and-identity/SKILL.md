---
name: authentication-and-identity
description: Load when designing or reviewing login, sessions, tokens, OAuth/OIDC/SAML flows, passkeys/WebAuthn, MFA, multi-tenant identity, or B2B SSO/SCIM — or when auditing auth code for holes like missing state, open redirects, JWT misuse, or token-storage mistakes.
---

# Authentication and Identity

## Core mental model

- **Authentication, authorization, and delegation are three different problems; most auth bugs are category errors between them.** OAuth 2.0 is a *delegation* protocol ("this app may call this API on my behalf") — using a bare OAuth access token as proof of *who the user is* is the classic mistake (any app the user granted a token to can replay it to log into yours). OIDC exists precisely to bolt authentication onto OAuth: the ID token is signed, audience-bound proof of "who authenticated, when, to whom." SAML solves the same problem as OIDC in XML for legacy enterprise IdPs. Map: users signing in → OIDC; API access on a user's behalf → OAuth; machine-to-machine → OAuth client credentials; enterprise customers with Okta/Entra → OIDC if possible, SAML because you'll be forced to.
- **Sessions are a revocation-latency dial.** Server-side sessions: revoke instantly, cost a lookup per request. Stateless JWTs: no lookup, but *cannot be revoked* before expiry — a logout that doesn't invalidate anything, a fired employee whose token works for a week. Every session architecture is a chosen point on this dial; pretending JWTs give you statelessness *and* revocation is the lie at the center of most bad designs.
- **Don't build the protocol layer; do own the architecture decisions.** As of 2026 the ecosystem has converged: authorization code + PKCE for everything interactive (OAuth 2.1 — still an IETF draft as of mid-2026, draft-15, but its requirements are the de-facto standard: PKCE mandatory, implicit and password grants removed), certified libraries for the flows (never hand-roll: `openid-client` for Node, Authlib for Python, AppAuth for mobile), managed or self-hosted IdPs (Auth0/Cognito/Entra, Keycloak/Ory/Zitadel) for the server side. Your job is the decisions libraries can't make: token lifetimes, storage, tenancy model, recovery flows, and the validation checklist.
- **Account recovery is the real attack surface.** Attackers don't beat your cryptography; they call your helpdesk, phish a reset email, SIM-swap the SMS fallback. Every authentication strength claim is capped by the weakest recovery path — a passkey-protected account with email-link recovery *is* email-strength. Design recovery first, not last.
- **Phishing resistance is a property, not a strength level.** TOTP is "strong" but phishable in real time by any proxy kit (Evilginx-style): user types the code into the fake site, attacker relays it. WebAuthn/passkeys are phishing-resistant *structurally* — the credential is bound to the origin, so the fake site can't even ask the right question. This distinction, not "number of factors," is the modern axis (NIST SP 800-63B rev 4, published 2025, requires phishing resistance at higher assurance levels and formally classifies SMS/voice OTP as *restricted* authenticators).

## Session architecture: the reasoning chain

Questions in order:

1. **Who consumes the session — one server-rendered app, or many services?** One app: server-side sessions in Redis/Postgres, session ID in an `httpOnly` cookie. Done — do not introduce JWTs to a monolith; you'd be buying JWT's costs (revocation problem, key management, size) to solve a fan-out problem you don't have.
2. **Many services that must validate locally?** Now stateless access tokens (JWTs) earn their keep — services verify a signature instead of calling a session store. Accept the revocation gap and *bound* it: access tokens live **5–15 minutes**, and revocation happens at the refresh boundary.
3. **The compromise that is now standard: short-lived JWT access token + rotating refresh token.** Refresh tokens are stateful (stored server-side, revocable, one per device/session). Rotation: every refresh issues a new refresh token and invalidates the old; **reuse of an already-rotated refresh token is the theft signal** → revoke the whole token family and force re-auth. This gives you: local validation (JWT), bounded exposure (minutes), real revocation (refresh boundary), and theft detection (reuse). If your design has long-lived JWTs *or* non-rotating refresh tokens, it's missing the point of the pattern.
4. **Browser storage — decided by XSS blast radius:** `localStorage`/`sessionStorage` are readable by any injected script; a single XSS exfiltrates tokens for offline replay. Never store tokens there. Cookies with `httpOnly; Secure; SameSite=Lax` (or `Strict`) are invisible to script — XSS can still *ride* the session (make requests as the user) but cannot *steal* it, which is a materially smaller blast radius, and `SameSite` kills most CSRF. SPA pattern as of 2026: refresh token in an `httpOnly` cookie scoped to the token endpoint, access token held in JS memory only (lost on reload; the cookie refresh restores it) — or better, the **BFF pattern**: a thin backend holds the tokens, browser gets only an ordinary session cookie, no tokens in the browser at all. Prefer BFF when you have any server anyway.
5. **JWT hygiene when you do use them:** verify with an allowlist of algorithms pinned in code (`jwt.decode(token, key, algorithms=["RS256"], audience="api://yours", issuer="https://idp/")` — passing user-influenced algorithm choices is how `alg=none` and RS256→HS256 confusion attacks live on in 2026 in the form of misconfigured libraries); always validate `iss`, `aud`, `exp`; fetch keys by `kid` from the IdP's JWKS with caching; put a `jti` in and keep a small denylist if you need emergency revocation without waiting out the TTL.

## Flows: authorization code + PKCE is the answer to almost everything

- **Why implicit died:** it delivered tokens in the URL fragment — leaking via browser history, referer processing, and injectable redirect handling — and had no client authentication step at all. The authorization *code* flow moves token delivery to a back-channel POST; **PKCE** (client generates `code_verifier`, sends `S256(code_verifier)` as `code_challenge`, must present the verifier to redeem the code) makes an intercepted code worthless. OAuth 2.1 requires PKCE for all clients — including confidential ones, because it also mitigates authorization-code injection.
- Per client type: web app with backend → code + PKCE, client secret, tokens server-side. SPA → code + PKCE (public client, no secret) or BFF. Mobile → code + PKCE via AppAuth with system browser (never a webview — webviews let the app steal credentials and break password-manager/passkey integration; Google blocks OAuth in embedded webviews). CLI/TV → device authorization grant (RFC 8628). Service-to-service → client credentials; inside one cloud/cluster prefer workload identity (IRSA, GKE Workload Identity, SPIFFE) over long-lived secrets.
- Sender-constrained tokens exist for high-value APIs: DPoP (RFC 9449) binds the token to a client-held key so a stolen bearer token won't replay. As of 2026 it's shipping in major IdPs but not yet default — reach for it when tokens guard money movement or admin planes.

### Cheat table: client type → flow → token home

| Client | Flow | Client auth | Tokens live in |
|---|---|---|---|
| Server-rendered web app | Code + PKCE | Client secret | Server session store; browser gets session cookie |
| SPA with any backend | Code + PKCE via BFF | Secret (at BFF) | BFF; browser gets httpOnly session cookie only |
| SPA, no backend at all | Code + PKCE (public) | None | Refresh: httpOnly cookie from IdP or worker; access: JS memory only |
| Mobile app | Code + PKCE via AppAuth + system browser | None (public) | OS keystore (Keychain/Keystore), never plaintext prefs |
| CLI / TV / constrained device | Device authorization grant (RFC 8628) | None | OS keychain or 0600 file, short-lived |
| Service → service (same cloud) | Workload identity (IRSA/GKE WI/SPIFFE) | Platform-attested | Ephemeral, platform-managed |
| Service → service (cross-org) | Client credentials | Secret or private_key_jwt (prefer keys) | Secrets manager, rotated |

Lifetime defaults worth defending in review: access token 5–15 min; refresh token days–weeks *sliding* with rotation, absolute cap 30–90 days; ID token minutes (it's login evidence, not an API credential); session cookie ≤ working day for sensitive apps with sliding renewal.

## Passkeys / WebAuthn engineering

- Mechanics: per-origin asymmetric keypair; private key in the platform authenticator (or synced via iCloud Keychain / Google Password Manager / password managers like 1Password/Bitwarden); server stores the public key and verifies a signed challenge. Origin binding = structural phishing resistance; no shared secret = nothing to leak in your breach.
- Adoption as of 2026 is real but partial: platform support is effectively universal (iOS/Android/macOS/Windows + all major browsers, including conditional UI/autofill and cross-device hybrid flows), most major consumer platforms offer passkeys, and enterprise deployment is mainstream — but coverage across the long tail of sites remains a minority, so **passkeys are an addition to your auth stack, not yet a replacement**. Design as: passkey-primary, password-plus-MFA fallback, with active nudges to enroll a passkey at login.
- Implementation: use a maintained server library (`@simplewebauthn/server` for Node, `webauthn` by duo-labs lineage for Python, `go-webauthn`); decisions that matter: require user verification (`userVerification: "required"`) for sensitive apps; register the RP ID as your registrable domain (subdomain scoping follows from it); store multiple credentials per user and *encourage* multi-enrollment (second device, security key) at signup — this is your cheapest recovery insurance; handle `excludeCredentials` so users don't double-register a device.
- **The real design problem is recovery, not the ceremony.** Measurable fractions of users lose all authenticators within a year or two (lost/wiped/switched-ecosystem devices). Synced passkeys shifted the problem: recovery of your account now often reduces to recovery of the user's *Apple/Google account* — acceptable for consumer apps, unacceptable for high-assurance ones. Ladder of recovery paths, strongest first: second enrolled authenticator → hardware security key kept offline → one-time recovery codes issued at enrollment (shown once, hashed at rest) → identity-verification re-proofing (document check) for high-value accounts → email reset (weakest; if present, your account security equals email security — say so in the threat model, or remove it for admin/finance roles). Helpdesk-mediated recovery must itself be phishing-resistant: callback to number on file + verified re-enrollment, never "read me the code I just emailed you" (that's how the big 2023–2025 helpdesk-social-engineering breaches worked).

## Multi-tenancy and B2B SSO realities

- **Model tenancy in the token, decide it at login.** Every access token should carry the tenant (`org_id` claim); every authorization check is (user, tenant, resource) — a user existing in two orgs must get *different* tokens per org, never one token with ambient access to both. Home-realm discovery (email domain → which IdP) happens before authentication; cache the mapping, and beware domain-based auto-join (anyone with a `@corp.com` address joins Corp's tenant — verify domain ownership via DNS TXT before enabling).
- B2B SSO is a product surface, not a protocol checkbox: each enterprise customer brings their own IdP (Okta, Entra ID, Ping, ADFS...), quirks included. Per-tenant IdP config (metadata URL, cert rotation, attribute mapping) needs an admin UI and self-serve test mode, or your support team becomes a SAML debugger. Libraries/services that absorb this: WorkOS, Stytch B2B; self-hosted, Keycloak orgs or a SAML lib like `samlify`/`python3-saml` — never hand-parse SAML XML (XML signature wrapping attacks are exactly the kind of thing you will get wrong).
- **SCIM (RFC 7643/7644) is how enterprises expect provisioning to work:** their IdP pushes create/update/*deactivate* to your `/scim/v2/Users` endpoint. The deactivation path is the one that matters — enterprises buy SSO+SCIM largely so that offboarding an employee kills access everywhere within minutes. **JIT provisioning** (create the user on first SSO login from attributes) is the cheap alternative — fine for account *creation*, but it has no deprovisioning story (the user just never logs in again... with valid sessions until they expire). Reasoning: JIT to launch, SCIM when customers ask (they will — it's on every enterprise security questionnaire), and *always* re-evaluate sessions against user status server-side so a SCIM deactivate actually terminates live sessions.
- SSO enforcement details that bite: block password login for domains with SSO enforced (or the CISO's carefully configured Okta policy is bypassed by "forgot password"); on IdP-initiated SAML (assertion arrives unrequested), prefer forcing SP-initiated instead — IdP-initiated has no `InResponseTo` binding and is a standing gift to assertion-injection bugs.

## MFA design reasoning

Ranking as of 2026 (aligned with NIST 800-63B-4): **passkeys/WebAuthn (incl. security keys) > platform-bound push with number matching > TOTP > SMS/voice OTP**. The reasoning: origin binding beats real-time phishing (only WebAuthn has it); number-matching push resists push-bombing fatigue attacks (plain "Approve?" push does not — that's how several famous 2022–2023 breaches went); TOTP is offline and cheap but proxy-phishable and its seed is a shared secret sitting in your DB (encrypt it, and it's still phishable); SMS adds SIM-swap and SS7/interception risk and is formally *restricted* in NIST 800-63B rev 4 — keep it only as a consciously-accepted-risk fallback for populations with no alternative, never for admin or high-value accounts. Step-up, don't blanket: authenticate sessions cheaply, require the strong factor at sensitive actions (payment, key export, role change) — better security *and* less fatigue. MFA enrollment and MFA *reset* are part of the recovery attack surface: an attacker who can "lose their phone" through your helpdesk removes your MFA entirely.

## How an expert thinks through it

*"B2B SaaS, React SPA + API, needs Okta SSO for enterprise customers and email/password for small ones."*

Don't build an IdP — pick one (say Keycloak self-hosted, or Auth0 if ops budget is tight); my app is a *client* of it, and enterprise IdPs federate *into* it (per-tenant OIDC/SAML connections), so my app only ever speaks OIDC to one issuer. Flow: SPA → authorization code + PKCE. Token handling: the API is multiple services, so JWT access tokens, 10-minute expiry, `aud` per API, `org_id` claim minted at login after home-realm discovery. Considered pure SPA token handling (refresh token in httpOnly cookie, access token in memory) — workable, but we already run a gateway, so BFF: gateway holds tokens, browser gets a `SameSite=Lax; httpOnly` session cookie, zero tokens in JS. Rejected localStorage without discussion. Rejected long-lived access tokens ("saves refresh traffic") because a 24h unrevocable token turns every XSS or log-leak into a day-long breach — the refresh hop is the price of a revocation story. Session revocation: refresh rotation with family-revocation-on-reuse at the IdP; SCIM deactivate → kill refresh tokens *and* gateway sessions (write that handler now, it's the whole point of enterprise offboarding). Enforcement detail: tenants with SSO required get password auth disabled at the realm level. MFA: passkey-first for password-realm users, TOTP fallback; enterprise SSO users bring their IdP's MFA — don't double-prompt (respect `amr`/`acr` from the federated assertion, but *require* those claims in the connection contract). Recovery: recovery codes at enrollment; email reset allowed for standard users, disabled for org admins (helpdesk-verified re-proofing instead). Checklist the callbacks before shipping: exact-match redirect URI allowlist, `state` + PKCE verified, `iss`/`aud`/`exp`/`nonce` validation in one shared middleware.

## Failure modes & pitfalls

- **Using an access token as login proof** ("we get the token, call `/userinfo`, trust the result"). Any token the user granted to *any* app can be replayed to your endpoint. Require an ID token and validate its `aud` is *your* client ID — that's the entire difference between OAuth and OIDC.
- **Missing or unverified `state`** → login CSRF: attacker completes the flow in the victim's browser session, victim ends up logged into the attacker's account and saves their card into it. Generate per-request `state`, bind it to the browser session, verify on callback. PKCE does not replace it for session binding (though OIDC `nonce` covers related ground for ID tokens).
- **Open redirect in the OAuth callback chain**: `redirect_uri` validated by prefix/substring (`startswith("https://app.com")` → `https://app.com.evil.io`), or a post-login `?next=` parameter that accepts absolute URLs. Codes/tokens exfiltrate through it. Exact-match allowlists only; relative-path-only for `next`.
- **Audience confusion**: one JWT accepted by five services with no `aud` check → a token minted for the read-only analytics API is replayed against the payments API. Distinct `aud` per API; every service validates its own.
- **`alg` trusted from the token header.** Modern libraries mostly block literal `alg=none`, but its descendants persist: accepting HS256 where RS256 is expected lets the attacker sign with the *public* key as the HMAC secret; passing `algorithms=jwt.get_unverified_header(t)["alg"]` reintroduces the whole class. Pin the algorithm list in code; never derive it from input.
- **Logout that only deletes the cookie.** The JWT lives until `exp`; the refresh token still works. Logout = revoke the refresh token (family) server-side + clear cookies; for JWTs mid-flight, either accept the ≤15-min tail (fine if lifetimes are short) or check a `jti` denylist on sensitive routes.
- **Refresh tokens without rotation or reuse detection** — a single stolen refresh token is then indefinite account access with no signal. Rotation + family revocation is table stakes (every major IdP supports it; turn it on).
- **Password reset link logic**: token not single-use, not expiring, not invalidated on password change, or user enumeration via "no such account" responses. Reset tokens: ≥128-bit random, hashed at rest, 15–60 min TTL, single-use, all sessions revoked on successful reset, uniform response regardless of account existence.
- **OAuth in an embedded webview** on mobile — breaks passkeys/password managers, teaches users to type passwords into app-controlled surfaces, and is rejected by major IdPs. System browser via AppAuth (ASWebAuthenticationSession / Custom Tabs).
- **Comparing secrets with `==`** (reset tokens, API keys, HMACs) → timing side-channel. `hmac.compare_digest` / `crypto.timingSafeEqual`.
- **Storing TOTP seeds and recovery codes in plaintext** next to the password hashes they're supposed to back up. Encrypt seeds (KMS-wrapped), hash recovery codes like passwords.
- **SAML without signature-on-assertion verification, or verifying the *response* signature while reading an unsigned *assertion*** (signature-wrapping). Use a hardened library, require signed assertions, validate `InResponseTo`, and pin the IdP cert per tenant — a tenant's IdP must never be able to mint assertions for another tenant (check the `Issuer` against the tenant's registered entity ID, not a global list).

## Worked micro-examples

**1. JWT validation done right (PyJWT) — every non-default argument is a closed vulnerability:**

```python
import jwt
from jwt import PyJWKClient

jwks = PyJWKClient("https://idp.example.com/.well-known/jwks.json")  # caches keys, refetches on unknown kid

def validate(token: str) -> dict:
    key = jwks.get_signing_key_from_jwt(token)         # selects by kid from header — but alg stays pinned below
    return jwt.decode(
        token,
        key.key,
        algorithms=["RS256"],                          # pinned allowlist; never derived from the token header
        audience="api://orders",                       # this service's aud, nobody else's token accepted
        issuer="https://idp.example.com/",
        options={"require": ["exp", "iat", "aud", "iss", "sub"]},  # absent claim = invalid, not "skipped check"
        leeway=30,                                     # bounded clock skew only
    )
```

**2. Session cookie flags for the BFF/session pattern — all five, none optional:**

```
Set-Cookie: session=Zx91…; Path=/; HttpOnly; Secure; SameSite=Lax; Max-Age=28800
```

`HttpOnly` (XSS can't read it), `Secure` (never over plaintext), `SameSite=Lax` (kills cross-site POST CSRF while allowing top-level nav; use `Strict` for admin panels, add CSRF tokens if you must run `SameSite=None`), bounded lifetime, and — implied — the value is a random ID referencing server-side state, not encoded user data. For the refresh-token cookie in an SPA pattern, also scope it: `Path=/oauth/token`.

**3. Refresh rotation with theft detection (the server-side logic that matters):**

```python
def refresh(presented: str):
    rec = db.refresh_tokens.find(hash=sha256(presented))
    if rec is None:
        raise Unauthorized()
    if rec.rotated_at is not None:              # already used once -> replay of a rotated token
        db.refresh_tokens.revoke_family(rec.family_id)   # theft signal: kill every descendant
        alert_security(rec.user_id, "refresh token reuse")
        raise Unauthorized()
    db.mark_rotated(rec)
    new_rt = mint_refresh(rec.user_id, family_id=rec.family_id)
    return mint_access_jwt(rec.user_id, ttl=600), new_rt
```

The family revocation on reuse is the difference between rotation-as-ritual and rotation-as-detection: whichever of {user, attacker} presents the stale token second proves the token leaked, and both lines of descent die.

## Verification / self-check

Before signing off on any auth design or review, walk this list; every item needs a concrete answer, not a nod:
1. **Revocation drill:** an employee is fired / a user clicks "log out everywhere" — enumerate every live credential (sessions, access tokens, refresh tokens, API keys, device trust) and state the latency to death for each. Anything unrevocable and >15 min is a finding.
2. **Callback checklist:** exact-match `redirect_uri`; `state` generated, bound, verified; PKCE end-to-end; `iss`/`aud`/`exp`/`nonce` validated; algorithm pinned; JWKS fetched by `kid` with rotation handling.
3. **Phish it on paper:** run an Evilginx-style proxy through each login and recovery path — which paths survive? (Only origin-bound ones do.) Then run the *helpdesk* attack: "I lost my phone and forgot my password" — what does the weakest human path yield?
4. **Storage audit:** where does every token/secret rest — browser (which mechanism), server (encrypted how), logs (Authorization headers and URLs scrubbed?), crash reports?
5. **Tenancy cross-check:** construct a valid token for tenant A and replay against tenant B's resources; construct tenant A's IdP assertion claiming a tenant-B email. Both must fail closed.

Stopping rule: an interactive flow built on a certified library, code+PKCE, short-lived access + rotating refresh, httpOnly storage or BFF, the callback checklist green, and a written recovery ladder is *done* — further cleverness (custom crypto, exotic token schemes, extra homegrown factors) now adds attack surface faster than assurance. Spend remaining effort on the humans: recovery, helpdesk procedure, and offboarding latency.
