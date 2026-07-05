---
name: security-engineering
description: Defensive security reasoning for designing, reviewing, or fixing code and systems — threat modeling, injection, authn/authz bugs, secrets, password storage, TLS, SSRF, deserialization, supply chain. Load when writing security-sensitive code, reviewing for vulnerabilities, designing auth flows, or answering "is this secure?" questions.
---

# Security Engineering

Compact review sheet. The standard expert corpus — parameterized queries, argv-vector subprocess, resolve-then-contain path checks, argon2id/bcrypt, `compare_digest`, full SSRF defense (resolved-IP pinning, redirect re-validation, metadata ranges, IMDSv2), JWT algorithm pinning, IDOR ownership-fused queries returning 404, GCM nonce catastrophe → misuse-resistant boxes, CSRF-for-cookie-auth, pickle/torch.load RCE, mass-assignment allowlists, dual-key rotation, rotate-anything-that-touched-git — is assumed known and appears below only as a completeness checklist, not as teaching material.

## Non-negotiable framing (two sentences)

Name the asset, the entry points, and the trust boundary before any control — a mitigation that doesn't map to a boundary crossing is theater. Every injection variant is one bug (data reaching an interpreter as syntax); the fix is always a separate channel, with escaping only as fallback.

## Decision one-liners (completeness checklist — audit each on review)

- SQL: parameters; dynamic identifiers via closed allowlist map (identifiers can't be bound).
- Subprocess: arg vector, `shell=False`. Templates: static template + variables (template string = code; Jinja2 SSTI → RCE).
- Paths: resolve then `is_relative_to` containment; residual TOCTOU symlink race → `O_NOFOLLOW`/`openat2(RESOLVE_BENEATH)`.
- Passwords: argon2id (or bcrypt) via a library with `check_needs_rehash`; never fast hashes however salted; never encryption.
- Secret comparison: `hmac.compare_digest`; better, compare fixed-length HMACs of tokens.
- URL fetch: scheme allowlist, resolve-and-reject private/metadata ranges, pin the resolved IP for the connect, re-check every redirect hop, egress firewall as backstop.
- Deserialization across a trust boundary: JSON/protobuf + schema. No safe untrusted pickle exists; grep for it in disguise: `torch.load` (force `weights_only=True`; default only since PyTorch 2.6), `joblib.load`, `numpy.load(allow_pickle=True)`, celery pickle content-type, session backends, `jsonpickle`.
- Tokens: `secrets.token_urlsafe(32)`; JWT with `algorithms=[...]` pinned + `iss`/`aud`/`exp` required; never trust header `alg`/`jku`/`kid` unvalidated.
- Checks live in middleware/ORM scope/lowest layer with context — per-handler checks fail open on next month's handler.
- Denylists: acceptable only as detection (WAF, alerting) on top of a structural fix.

## Authz review — the highest-yield bug class

- IDOR: every handler taking an ID scopes the fetch by principal (`WHERE id=? AND org_id=?`), 404 not 403. UUIDs are obscurity, not authz.
- Then check the *siblings*: same clause on PATCH/DELETE as on GET; API as well as UI; every route × method. Attackers use curl, not your frontend.
- Mass assignment: explicit settable-field allowlist; server-side fields (`role`, `owner_id`) come from the session, never the payload.
- Re-derive privilege server-side on elevation paths (email change, admin actions: password/MFA re-entry); role from a client-writable location is never authorization.
- Check-then-act must be atomic: check and use the same opened handle/resolved object.

## Pitfall one-liners

- `verify=False` / unchecked hostname / custom TrustManager = unauthenticated encryption; fix with a CA bundle, never by disabling.
- `NEXT_PUBLIC_`/`VITE_` means shipped-to-browser; a secret there is leaked, rotate.
- Secret in git history: rotate regardless of scrubbing; deleting commits doesn't unpublish.
- Trusting `X-Forwarded-For`/`Host` you don't control (rate limits, reset links) → header injection.
- Open redirect / OAuth `redirect_uri`: exact-match allowlist; `next=` must be a relative path, not `//`.
- XSS: escape for the *sink context*; `dangerouslySetInnerHTML`/`v-html`/`| safe` + user data = finding; rich text through DOMPurify/bleach allowlists, never regex.
- CSRF: cookie-authed state change needs token or SameSite + origin check regardless of JSON; bearer-in-header is immune; cookies `HttpOnly`+`Secure`, session ID rotated on login.
- CORS: never reflect Origin with credentials; CORS is not authn and does nothing against curl.
- Uploads: magic bytes not Content-Type, re-encode images, store outside web root under server names, sandbox ImageMagick/ffmpeg-class parsers.
- Uniform auth errors (no user enumeration via message *or* timing *or* reset flow); rate-limit login/OTP/reset per-account AND per-IP.
- Rolling your own composition from sound primitives (ECB, CBC-no-MAC, GCM nonce management) = finding; use Fernet/secretbox/XChaCha20 or AES-GCM-SIV.

## Red-flag grep (run on every security-relevant diff)

`verify=False`, `shell=True`, `pickle.loads`, `yaml.load(`, `eval(`/`exec(`, `md5`/`sha1` near password, `==` near token/secret, `random.` near anything secret, `dangerouslySetInnerHTML`/`innerHTML`/`v-html`/`mark_safe`/`| safe`/`render_template_string`, `csrf_exempt`, `NEXT_PUBLIC_`/`VITE_` near keys, `alg` handling near JWT, `torch.load` without `weights_only`, `tempfile.mktemp`, `chmod 0777`, hardcoded `AKIA`/`BEGIN PRIVATE KEY`, `--legacy-peer-deps`, `debug=True`.

## Verification / self-check

- Asset, entry point, trust boundary named for each control — else you're pattern-matching.
- Every attacker-influenced value traced to every interpreter it reaches; separate channel or (weaker) escaping at each crossing.
- Every route × method with an object ID: ownership check fused into the fetch.
- "What happens if the caller forgets?" — if the answer is "insecure," push the check down a layer.
- State residual risks explicitly; a security answer claiming completeness is usually wrong.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 14 baseline (compressed to checklists), 0 partial, 0 delta.
- Opus cold reproduced everything, often more precisely than the prior skill text (IMDSv2/dialer-level SSRF pinning, argon2 params + legacy-hash wrapping, `weights_only` default since PyTorch 2.6, openat2 RESOLVE_BENEATH, double-submit/SameSite nuances).
- Value of this file is now checklist completeness under review pressure, not knowledge transfer; biggest residual leverage: sibling-route IDOR sweep and the red-flag grep list as a mechanical pass.
