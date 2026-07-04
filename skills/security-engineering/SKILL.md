---
name: security-engineering
description: Defensive security reasoning for designing, reviewing, or fixing code and systems — threat modeling, injection, authn/authz bugs, secrets, password storage, TLS, SSRF, deserialization, supply chain. Load when writing security-sensitive code, reviewing for vulnerabilities, designing auth flows, or answering "is this secure?" questions.
---

# Security Engineering

## Core mental model

1. **Threat model before controls.** Never recommend a mitigation until you can name: the asset (what's worth stealing/corrupting), the entry points (every place attacker-influenced data or requests arrive), and the trust boundaries (where data crosses from less-trusted to more-trusted context). A control that doesn't map to a crossing of a trust boundary is security theater. When reviewing code, first enumerate: "what does the attacker control here?" — request params, headers, file contents, DB rows written by other users, URLs fetched, environment of a child process.
2. **The entire injection family is one bug: data interpreted as code.** SQL injection, command injection, path traversal, template injection (SSTI), XSS, LDAP injection, header injection, YAML `!!python` tags — all are the same failure: attacker-controlled bytes reach an interpreter (SQL parser, shell, filesystem path resolver, template engine, HTML parser) as *syntax* instead of *data*. The universal fix is the same shape everywhere: keep data and code in separate channels (parameterized queries, `subprocess.run([...])` argument vectors, allowlisted path roots, context-aware output encoding). Escaping/sanitizing is the fallback, not the fix — it fails on the encoding you forgot.
3. **Authentication answers "who are you"; authorization answers "may *you* do *this to that object*".** Most real-world authz bugs are not missing login checks — they are missing *object-level* checks after login (IDOR). Every handler that takes an ID must verify the authenticated principal's relationship to that specific object.
4. **Secrets are radioactive material with a half-life.** They must never enter source control, logs, error messages, or client-side code; they must live in a dedicated store; and every secret needs a rotation story *before* it's issued, because you will eventually need to rotate it under pressure.
5. **Secure by default, explicit to weaken.** The safe behavior must be what happens when a caller does nothing: deny-by-default authz, `verify=True` TLS, `autoescape=True` templates, least-privilege tokens, allowlists over denylists. If safety requires remembering to add a check at every call site, the design is already broken — someone will forget.

## Decision frameworks

| Situation | Do this | Not this | Because |
|---|---|---|---|
| Building SQL with user input | Parameterized query (`cursor.execute("... WHERE id = %s", (uid,))`) | f-strings, `.format`, manual quoting/escaping | Escaping breaks on charset tricks, second-order injection; params keep data out of the parse tree entirely |
| Dynamic table/column names (can't parameterize) | Allowlist map: `{"name": "name", "created": "created_at"}[user_key]` | Sanitizing the identifier | Identifiers can't be bound as params; only a closed allowlist is safe |
| Running an external program | `subprocess.run(["convert", in_path, out_path])` with `shell=False` | `os.system(f"convert {path}")`, `shell=True` | Arg vector never touches a shell parser; no quoting problem exists |
| User-supplied filename/path | Resolve then verify containment: `p = (root / name).resolve(); if not p.is_relative_to(root.resolve()): reject` | Stripping `../` substrings | `....//`, encoded separators, absolute paths, and symlinks defeat string filters; resolve-then-check defeats them all (symlinks need `resolve()` on both sides) |
| Rendering user data in templates | Static template + variables (`render_template("x.html", name=name)`) | Concatenating user input into the template string | Template string = code; SSTI in Jinja2 reaches `__subclasses__` → RCE |
| Storing passwords | `argon2id` (argon2-cffi) or bcrypt with per-password salt (built into both) | SHA-256/MD5 even salted, even iterated a few times; encryption of any kind | Fast hashes allow billions of guesses/sec on GPU; encryption is reversible (key theft = all passwords). Add a server-side "pepper" from the secret store if you want defense-in-depth |
| Comparing secrets/tokens/MACs | `hmac.compare_digest(a, b)` | `==` | `==` short-circuits on first mismatch → timing oracle |
| Fetching a user-supplied URL (webhooks, "import from URL", link previews) | Treat as SSRF: allowlist schemes (http/https only), resolve DNS and reject private/link-local/metadata ranges (10/8, 172.16/12, 192.168/16, 127/8, 169.254.169.254, ::1), pin the resolved IP for the actual connection, disable redirects or re-check each hop | Blocking hostnames by string match | Attacker uses redirects, `0x7f000001`, DNS rebinding, IPv6-mapped forms; cloud metadata endpoint = instance credentials |
| Deserializing data that crossed a trust boundary | JSON (or protobuf) + schema validation | `pickle.loads`, `yaml.load` (use `yaml.safe_load`), Java native serialization, PHP `unserialize` | Native deserializers instantiate arbitrary classes → RCE by design, not by bug. There is no safe way to unpickle untrusted bytes |
| Session/API tokens | Random 256-bit from `secrets.token_urlsafe(32)`, stored server-side or as signed JWT with `algorithms=["RS256"]` pinned | `random`/`uuid1`, JWT decode without pinning `algorithms` | `random` is predictable from outputs; unpinned JWT accepts `alg: none` or HMAC-with-public-key confusion |
| Choosing where a security check lives | Centralized middleware/decorator/ORM-scope, enforced at the lowest layer that has context | Repeated inline checks in each handler | Per-call-site checks fail open on the handler someone adds next month |

**When is a denylist acceptable?** Almost never for input validation. Acceptable only for *detection* (WAF signatures, log alerting) layered on top of a structural fix.

## Failure modes and pitfalls

- **IDOR via trusted IDs.** `GET /api/invoices/4211` fetches by primary key and checks only "is logged in." Correction: every object fetch must scope by owner — `Invoice.objects.get(id=iid, owner=request.user)` — or centralize via ORM default scopes / row-level security. Applies equally to UUIDs: unguessable IDs are obscurity, not authorization (they leak in logs, referers, exports).
- **Authz on the read but not the write** (or on GET but not the matching PATCH/DELETE), and **authz in the UI but not the API** (button hidden, endpoint open). Enumerate every route × method when reviewing; attackers use curl, not your frontend.
- **Mass assignment / over-binding:** framework binds request body straight to the model, so `{"is_admin": true}` in a profile-update payload works. Correction: explicit allowlist of settable fields (DTOs, serializer `fields=[...]`, `permit` lists).
- **Checking authn/authz *after* doing work,** or performing the check and the action non-atomically (TOCTOU): file permission checked, then path re-resolved through a symlink swapped in between. Correction: check and use the *same* opened handle/resolved object.
- **Secrets in "hidden" places that end up in git:** `.env` committed "temporarily", secrets in `docker-compose.yml`, in CI YAML, in Jupyter notebooks, in test fixtures, in client-side JS bundles (`NEXT_PUBLIC_`/`VITE_` prefixed vars ship to the browser — that prefix *means* public). Correction: secrets come from a secret manager or injected runtime env; `.env` in `.gitignore` from commit zero; run `gitleaks`/`trufflehog` in CI. **If a secret ever touched git history, rotate it — deleting the commit does not unpublish it** (forks, clones, GitHub cache, scrapers already have it).
- **Logging the secret you protected everywhere else:** full request dumps, `Authorization` headers in access logs, DB URLs with passwords in stack traces. Add explicit redaction to the logging layer.
- **`verify=False` left in production** (`requests.get(url, verify=False)`, `NODE_TLS_REJECT_UNAUTHORIZED=0`, `InsecureSkipVerify: true`, custom `TrustManager` that returns without checking). This reduces TLS to unauthenticated encryption — any on-path attacker terminates it. Correction for internal/self-signed CAs: distribute the CA bundle and pass `verify="/path/ca.pem"`; never disable. Also: verifying the cert chain but not the **hostname** is the same hole (e.g., raw `ssl.SSLContext()` without `check_hostname=True`; `ssl._create_unverified_context` in copy-pasted snippets).
- **Rolling your own crypto composition** even from sound primitives: AES-ECB (pattern-leaking), AES-CBC without a MAC (padding-oracle), reusing a nonce with AES-GCM (catastrophic — leaks the auth key). Correction: use a misuse-resistant box — `cryptography.fernet.Fernet`, libsodium `secretbox`/`crypto_box` — and treat any hand-assembled cipher mode in a diff as a finding.
- **Comparing/deriving instead of verifying:** trusting `X-Forwarded-For` for rate limiting or authz when you don't control the proxy chain; trusting `Host` header for password-reset links (host-header injection → account takeover). Correction: canonical hostnames from config, client IP only from the trusted proxy depth you configured.
- **Open redirect + OAuth:** `?next=` URL used unvalidated after login, or OAuth `redirect_uri` matched by prefix. Both leak tokens/sessions. Correction: exact-match allowlist of redirect targets; for `next=`, accept only relative paths that you then verify start with `/` and not `//` (protocol-relative).
- **SSRF via the "safe" library:** you validated the URL but the HTTP client follows redirects to `http://169.254.169.254/`. Or you validated the hostname, then the client re-resolved DNS (rebinding). Correction: validate the *connection target*, not the input string — resolve once, connect to that IP, set `allow_redirects=False` or validate every hop.
- **Unsafe deserialization hiding in dependencies:** Django/Flask session backends with pickle serializers, `torch.load` (pickle under the hood — use `weights_only=True`), celery with pickle content type, `jsonpickle`. Grep for these when reviewing, not just literal `pickle.loads`.
- **Supply chain blind spots:** unpinned dependencies (`>=`) so CI silently pulls a hijacked release; typosquats (`requets`); install scripts (`postinstall`) running on `npm install`; internal package names not registered publicly → dependency confusion. Correction: lockfiles committed and used in CI (`pip install -r` from a hash-pinned file / `npm ci`), private registry with scoped names, `--ignore-scripts` where feasible, and automated advisories (`pip-audit`, `npm audit`, Dependabot/Renovate) with a triage owner.
- **Verbose failure as an oracle:** login returning "user not found" vs "wrong password" (user enumeration — also via timing and via password-reset responses); stack traces to clients revealing paths, versions, queries. Return uniform errors externally, rich errors internally.
- **Rate limiting forgotten on the expensive/sensitive paths:** login, OTP verify, password reset, signup email. Brute force is an authz failure in slow motion. Lock/throttle per-account *and* per-IP (each alone is bypassable).
- **XSS via the "safe" escape hatch:** React's `dangerouslySetInnerHTML`, Vue's `v-html`, Jinja's `| safe`, `.innerHTML =` — each is a deliberate bypass of the framework's context-aware escaping, and each appears in codebases attached to user-controllable data ("it's just their display name"). Also context confusion: HTML-escaping a value then placing it inside a `<script>` block or an `href` (where `javascript:` URLs live) — escaping must match the *sink context*. Sanitize rich-text HTML with a real sanitizer (DOMPurify, bleach) with an element/attribute allowlist; never with regexes.
- **CSRF ignored because "we use JSON APIs":** state-changing endpoints authenticated by cookies are CSRF-able regardless of payload format if the server doesn't verify a CSRF token or (for same-site apps) `SameSite=Lax/Strict` cookies + origin checking. Bearer-token auth in a header is inherently CSRF-immune; cookie auth never is by itself. While there: session cookies need `HttpOnly` (XSS can't read them), `Secure`, and session ID *rotation on login* (session fixation).
- **CORS reflected-origin misconfiguration:** `Access-Control-Allow-Origin: <echo of request Origin>` plus `Allow-Credentials: true` = any website can read your authenticated API. Correction: exact-match allowlist of origins; never reflect; remember CORS protects *browsers' reads*, it is not an authn mechanism and does nothing against curl.
- **File-upload trust chain:** validating `Content-Type` header (attacker-set) or file extension instead of content; storing uploads under the web root where `shell.php` executes; passing filenames into ImageMagick/ffmpeg/LibreOffice (historically RCE-rich parsers) unsandboxed. Correction: validate magic bytes, re-encode images, store outside web root under server-generated names, serve with `Content-Disposition: attachment` and a no-sniff header, run parsers in a sandbox/least-privileged worker.
- **Privilege checks keyed on client-supplied role data:** role read from a JWT claim the client controls a copy of is fine *only if* the signature is verified and the token was issued server-side; role read from a hidden form field, cookie value, or request body never is. Re-derive privilege from the server-side session/DB on sensitive operations, and re-check (not just re-read) on privilege *elevation* paths (e.g., require password/MFA re-entry for email change, admin actions).

## Secrets lifecycle (the rotation story, concretely)

- **Issue:** generate with `secrets` module / the platform's KMS, never by a human typing; scope to least privilege (read-only DB user for the reporting service, not the app's superuser URL).
- **Store:** secret manager (Vault, AWS Secrets Manager, GCP Secret Manager, Doppler) or platform-injected env at deploy time. Config files hold *references* ("`db_password: vault:app/db`"), not values.
- **Deliver:** runtime fetch or injected env var; never build-time bake into images (images get pushed to registries and cached on laptops), never client bundles.
- **Rotate:** dual-secret overlap pattern — issue new, deploy consumers accepting both, flip producers, revoke old. Design APIs to accept two valid keys during the window; a rotation that requires simultaneous everywhere-at-once cutover will never be run until the breach forces it.
- **Detect:** pre-commit + CI secret scanning; canary tokens for high-value stores; alert on use of a revoked credential (that's your breach indicator).

## Worked micro-examples

**1. Path traversal fixed structurally (not by string filtering):**
```python
from pathlib import Path
UPLOAD_ROOT = Path("/srv/uploads").resolve()

def open_upload(user_supplied_name: str):
    candidate = (UPLOAD_ROOT / user_supplied_name).resolve()   # collapses .. and symlinks
    if not candidate.is_relative_to(UPLOAD_ROOT):              # containment check AFTER resolution
        raise PermissionError("outside upload root")
    return candidate.open("rb")
```
`name = "../../etc/passwd"` resolves to `/etc/passwd`, fails containment. `"....//secret"` and encoded variants also fail because we never pattern-match the string — we check where it actually lands. Residual risk to note: a symlink *created inside* the root after this check (TOCTOU) — on Linux, open with `os.open(..., os.O_NOFOLLOW)` for the final component if uploads are attacker-writable.

**2. IDOR review, before/after:**
```python
# BEFORE — authenticated ≠ authorized; any logged-in user reads any invoice
@app.get("/invoices/<int:iid>")
@login_required
def get_invoice(iid):
    return db.session.get(Invoice, iid).to_json()

# AFTER — object-level check fused into the query; absence == 404
@app.get("/invoices/<int:iid>")
@login_required
def get_invoice(iid):
    inv = db.session.execute(
        select(Invoice).where(Invoice.id == iid,
                              Invoice.org_id == current_user.org_id)
    ).scalar_one_or_none()
    if inv is None:
        abort(404)          # 404, not 403: don't confirm the object exists
    return inv.to_json()
```
Then check the *sibling* routes: does `PATCH /invoices/<iid>` have the same `org_id` clause? That's where the bug usually survives.

**3. Password verification done right (argon2-cffi):**
```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

ph = PasswordHasher()          # argon2id, sane memory/time defaults, random salt per hash

def register(pw: str) -> str:
    return ph.hash(pw)         # store the full encoded string (params+salt+hash)

def login(stored: str, pw: str) -> bool:
    try:
        ph.verify(stored, pw)
    except VerifyMismatchError:
        return False
    if ph.check_needs_rehash(stored):   # transparently upgrade old cost params
        save_new_hash(ph.hash(pw))
    return True
```
Note what's absent: no manual salt handling, no hex comparisons, no SHA anywhere. `check_needs_rehash` is the rotation story for cost parameters.

## Verification / self-check

Before presenting a security answer or approving a design:
- Can you name the asset, entry point, and trust boundary the control protects? If not, you're pattern-matching, not threat modeling.
- For every attacker-influenced value, trace it to every interpreter it reaches (SQL, shell, path, template, HTML, URL fetcher, deserializer). Did each crossing use a separate-channel mechanism, or escaping?
- For every route × method that takes an object ID: is there an object-level ownership check, and is it fused into the fetch (not a separate lookup)?
- Grep the diff for the red-flag tokens: `verify=False`, `shell=True`, `pickle.loads`, `yaml.load(`, `eval(`, `md5`/`sha1` near "password", `==` near "token", `random.` near anything secret, `NEXT_PUBLIC_`/`VITE_` near keys, `alg` handling near JWT.
- Ask "what happens if the caller forgets?" for each safety mechanism. If the answer is "insecure," push the check down a layer (middleware, ORM scope, type that can't be constructed unsafely).
- State residual risks explicitly (e.g., "this stops traversal but not a malicious symlink race") — a security answer that claims completeness without caveats is usually wrong.
