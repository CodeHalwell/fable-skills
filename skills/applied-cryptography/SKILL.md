---
name: applied-cryptography
description: Load when writing or reviewing code that encrypts, hashes, signs, stores passwords, generates tokens, configures TLS, or handles JWTs/keys — or when choosing crypto primitives, libraries, or parameters. Also load when auditing code for timing side channels, nonce reuse, or key-management flaws.
---

# Applied Cryptography

## Core mental model

1. **Never design, only compose.** Do not invent constructions ("hash it twice with a secret prepended", "XOR with a keystream I derived"). Every custom composition of secure primitives should be presumed broken — that's how length-extension, padding-oracle, and nonce-reuse disasters happen. Choose a vetted construction that matches the goal, use a high-level library API, and pass it correct inputs.
2. **Encryption without authentication is a vulnerability, not a feature.** Attackers who can flip ciphertext bits (CBC bit-flipping) or probe padding errors (padding oracles) decrypt or forge without the key. Default to AEAD: AES-256-GCM or ChaCha20-Poly1305. There is no modern use case for unauthenticated CBC/CTR in application code, and none *ever* for ECB.
3. **Nonces are load-bearing.** AEAD security proofs assume the (key, nonce) pair is never repeated. One GCM nonce reuse under the same key leaks the XOR of plaintexts *and* the authentication key (forgeries forever after). Treat nonce management as the hardest part of using AEAD, because it is.
4. **Match the primitive to the security goal, not the vibe.** Integrity against tampering by outsiders → MAC. Non-repudiation / verification by parties without the secret → signature. Deduplication/fingerprint of public data → plain hash. Password storage → deliberately slow, memory-hard KDF. "I hashed it" answers none of these questions by itself.
5. **Secrets leak through time and errors.** Any branch, early-exit comparison, or distinguishable error message that depends on secret data is an oracle. Compare MACs/tokens with constant-time functions; return one uniform error for all decryption/auth failures.
6. **Key management dominates.** Most real-world breaks are keys in git, keys reused across environments, no rotation plan, or predictable "randomness" — not broken math. Where the key lives, how it rotates, and what happens on compromise are design requirements, not afterthoughts.

## Decision frameworks

**"I need to..." → construction (Python `cryptography` unless noted):**

| Goal | Use | Never |
|---|---|---|
| Encrypt data at rest / in a message | `AESGCM` or `ChaCha20Poly1305` (AEAD), random 96-bit nonce stored alongside ciphertext | ECB; CBC without HMAC; homebrew CTR |
| Encrypt with a simple safe API | `cryptography.fernet.Fernet` (AES-CBC+HMAC, handles IV/MAC/versioning for you) | Rolling your own envelope format |
| Integrity/auth of a message with shared secret | HMAC-SHA256 (`hmac` stdlib or `cryptography.hazmat...HMAC`) | `sha256(key + msg)` — length-extension forgeable; `sha256(msg + key)` — still nonstandard, just use HMAC |
| Signature (verifier has no secret) | Ed25519 (`Ed25519PrivateKey`); ECDSA P-256 if ecosystem requires | RSA with PKCS#1 v1.5 for new designs; DSA |
| Password storage | argon2id (`argon2-cffi`, defaults are sane) or scrypt/bcrypt | SHA-256(password), even salted+iterated by hand; MD5/SHA1 anything |
| Derive keys from a *high-entropy* secret (DH output, master key) | HKDF-SHA256, distinct `info=` label per purpose | Using argon2 here (pointless cost); using the raw secret directly for multiple purposes |
| Derive an encryption key from a *password* | argon2id / scrypt (needs salt, stored with ciphertext) | HKDF (no work factor — offline brute force is cheap) |
| Random tokens, session IDs, nonces, salts | `secrets.token_urlsafe(32)` / `os.urandom()` | `random.*` (Mersenne Twister — state recoverable from 624 outputs), `uuid.uuid4()` for secrets is acceptable-entropy but `secrets` is clearer, `uuid.uuid1()` never (MAC+timestamp) |
| Key exchange | X25519 + HKDF; or just use TLS/libsodium `crypto_box` | Textbook DH with homemade parameters |
| Checksums for accidental corruption only | CRC32/xxHash — and label it non-cryptographic | Claiming integrity vs adversaries from a CRC |

**Hash vs MAC vs signature — decide by who verifies:** verifier shares a secret → HMAC. Verifier must not be able to forge (or is the public) → signature. Nobody is adversarial → plain hash. If a hash of secret-influenced data is exposed to attackers, it needed to be a MAC.

**Nonce strategy for AEAD:** random 96-bit nonce per message is fine up to ~2³² messages per key (birthday bound on 96 bits); beyond that, rotate keys or use a counter with *persistent, crash-safe* state, or XChaCha20-Poly1305 (192-bit nonce, random is always fine). A counter in memory that resets on restart is a reuse bomb. Never derive the nonce from the plaintext or a timestamp with second granularity.

**JWT decision rules:** prefer not to hand-roll; if you must: pin the algorithm on *verify* (`jwt.decode(tok, key, algorithms=["EdDSA"])` — the list is mandatory, never taken from the token header), reject `alg: none` (PyJWT does if you pass `algorithms`), never verify an RS256 token with the public key while also accepting HS256 (key-confusion: attacker signs HS256 using the public key as the HMAC secret), always validate `exp`/`aud`/`iss` explicitly (`options={"require": ["exp"]}`, pass `audience=`). Symmetric (HS256) only when issuer == sole verifier. Keep lifetimes short; revocation needs a server-side list — JWTs are bearer tokens.

**TLS configuration sanity (server):** TLS 1.2 minimum (prefer 1.3), certificates from a real CA (or internal CA with proper distribution — not `verify=False`), enable HSTS. Client code: never disable verification "temporarily" (`verify=False`, `CERT_NONE`, custom `check_hostname=False`) — these ship to prod. If you must trust a private CA, add it to the trust store (`REQUESTS_CA_BUNDLE`), don't turn verification off. Certificate pinning only with a rotation story.

## Failure modes & pitfalls

- **GCM nonce reuse.** Two messages under (key, nonce): XOR of keystreams cancels → plaintext XOR revealed; worse, GHASH key recoverable → attacker forges arbitrary valid ciphertexts. Seen in practice as: hardcoded IV, `iv = b"\x00"*12`, counter reset on process restart, fork() duplicating counter state, random nonce of only 8 bytes. Check every AEAD call site: where does the nonce come from, and can two calls ever collide?
- **Encrypting each chunk/field independently without binding context.** Attacker swaps ciphertext blobs between records (your encrypted `role` field from an admin row into their row). Fix: AEAD *associated data* — pass record ID/purpose as `aad`: `AESGCM(key).encrypt(nonce, plaintext, b"user:42:role")`. If you're not using the AAD parameter, ask what cross-protocol / cut-and-paste swaps are possible.
- **MAC-then-encrypt or separate-then-compose.** Composing CBC + HMAC yourself invites padding oracles (decrypt-then-check ordering bugs) — the Lucky13/POODLE family. Use an AEAD mode or Fernet; if legacy forces CBC+HMAC, it must be encrypt-then-MAC over IV‖ciphertext, verified before any decryption or padding inspection, constant-time compare.
- **`hmac_calculated == hmac_received` with `==`.** Short-circuit comparison leaks match length via timing. Use `hmac.compare_digest(a, b)` (also for API keys, tokens, password-reset codes). Same for any secret-dependent early return.
- **Distinguishable failure messages.** "Invalid padding" vs "invalid MAC" vs "invalid user" — each distinction is an oracle (padding oracle attacks decrypt CBC at ~128 queries/byte; user enumeration). Return one generic error and identical response timing for auth failures.
- **`random` for secrets.** `random.choices(alphabet, k=32)` for an API key: Mersenne Twister internal state is fully recoverable from 624 consecutive 32-bit outputs, and is often seeded from time. Grep for `import random` in any code path making tokens, salts, nonces, or passwords. Also: NumPy RNG, `rand()` in C, `Math.random()` in JS — all non-cryptographic.
- **Password hashing with fast hashes, or with HKDF/PBKDF1-style DIY loops.** A GPU rig does ~10⁹–10¹⁰ SHA-256/s; iterating SHA-256 100k times yourself gets you PBKDF2-without-review. Use argon2id (memory-hardness defeats GPUs/ASICs). Per-user random salt (the library does this); a global "salt" constant is a pepper, not a salt, and doesn't stop rainbow/parallel attacks across your own user table.
- **Same key for multiple purposes.** One key used for both encryption and MAC, or across services, couples their compromise and can enable cross-protocol attacks. Derive purpose-specific subkeys: `HKDF(master, info=b"svc-A/encryption/v1")`. The `info` label is what makes keys independent — empty `info` everywhere defeats the point.
- **Storing the key next to the ciphertext.** Key in the same DB/table/bucket as the data, key in source code, key in a client app "obfuscated". Encryption then adds nothing against the realistic attacker (DB dump). Keys belong in a KMS/HSM/secret manager, or at minimum a separate store with separate access control; the design question is "who can read ciphertext but not the key?"
- **No key rotation plan → no key rotation.** Prefix ciphertexts with a key-version byte (Fernet's `MultiFernet` does this pattern) so you can add key N+1, decrypt with any, encrypt with newest, and re-encrypt lazily. Retrofitting versioning after a compromise is miserable. Rotation ≠ re-encrypting everything on day one; it's the *ability* to.
- **JWT `alg` taken from the token.** Libraries historically honored `{"alg":"none"}` or attacker-chosen HS256-with-public-key. In PyJWT the `algorithms=[...]` allowlist parameter is your only defense — code review any `jwt.decode` without it (older PyJWT allowed omitting it). Also check: `verify_exp` not disabled, `aud` checked (token minted for service A replayed to service B).
- **"Encrypted" meaning encoded.** base64, hex, zlib, XOR-with-fixed-byte are not encryption. Conversely, don't claim confidentiality from an HMAC or integrity from encryption alone.
- **Compressing before encrypting attacker-influenced + secret data in the same stream.** Ciphertext length leaks shared substrings (CRIME/BREACH pattern). If attacker input and secrets co-reside in a compressed-then-encrypted payload, disable compression or segregate.
- **RSA misuse cluster.** Raw/textbook RSA (deterministic, malleable), PKCS#1 v1.5 encryption (Bleichenbacher oracle), encrypting bulk data with RSA directly (size limits, no forward motion) — use hybrid: X25519/RSA-OAEP to wrap a symmetric AEAD key. For new systems just use Ed25519/X25519.

## Worked micro-examples

The correct password-file-encryption envelope (argon2id from the password → per-message random 12-byte nonce → `AESGCM(key).encrypt(nonce, pt, purpose)` with the purpose bound as AAD → serialize `version‖salt‖nonce‖ct+tag`) is a construction a strong model reproduces cold; what it under-emphasizes and this skill enforces:
- **The AAD parameter is not optional decoration.** Its job is to bind the ciphertext to its context (`b"user:42:role"`) so an attacker cannot cut-and-paste an encrypted `role` blob from an admin row into their row. If you are calling an AEAD and passing no associated data, ask out loud what cross-record/cross-protocol swap that permits.
- **A version byte is the whole rotation story.** `b"v1"‖…` (or `MultiFernet`) is what lets you add key N+1, decrypt with any, encrypt with newest — retrofitting this after a compromise is the miserable path. "No rotation story" is a finding, not a footnote.
- **One exception type for all failures** (argon2id-mismatch, bad tag, wrong version) so failure paths stay indistinguishable — timing and message.

Not probed, and a common miss: **compressing attacker-influenced + secret data in one stream before encrypting leaks secrets via ciphertext length** (CRIME/BREACH). Segregate or disable compression when both co-reside.

## Verification / self-check

Before presenting crypto code or review conclusions:
1. **Nonce audit:** for every AEAD/cipher call — where is the nonce born, can any two calls under the same key collide (restart, fork, thread, hardcode)? Say the answer explicitly.
2. **Primitive-goal match:** for each primitive used, state the goal it serves (confidentiality / integrity / non-repudiation / slow-KDF / entropy) and confirm no goal is served by the wrong tool.
3. **Grep-level checks:** `import random` near secrets; `==` comparing MACs/tokens; `verify=False` / `CERT_NONE`; `ECB` / `MODE_ECB`; `md5|sha1` in security contexts; `jwt.decode` without `algorithms=`.
4. **Failure-path uniformity:** do all decrypt/auth failures produce indistinguishable errors and similar timing?
5. **Key lifecycle:** where is the key stored, who can read it, how does rotation work, is there a version byte in the ciphertext format? "No rotation story" is a finding, not a footnote.
6. **Did I invent anything?** If the design contains any construction you can't name in a textbook/RFC (encrypt-then-MAC, HKDF-expand, envelope encryption), replace it with one you can.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 10 claims: ~9 baseline (compressed), ~1 partial. This skill is ≥90% BASELINE — Opus 4.8 cold correctly produced AES-256-GCM/XChaCha for data-at-rest, the ~2³² birthday bound with AES-GCM-SIV/XChaCha alternatives, length-extension→HMAC, Argon2id with current OWASP params, HKDF-vs-Argon2 for high-entropy-vs-password inputs, the full JWT verify checklist (alg pinning, alg-confusion, exp/aud/iss), `compare_digest`, GCM-nonce-reuse leaking BOTH plaintext XOR and the GHASH forgery key, and `secrets` vs Mersenne-Twister.
- The residual value is a checklist/discipline: the two things Opus under-weighted are (a) using the AEAD **AAD** parameter to stop cut-and-paste swaps, and (b) baking a **key-version byte** in from day one for rotation. CRIME/BREACH (compress-then-encrypt) is the one unprompted fact worth keeping.
- Worked examples collapsed to prose anchors; the code was boilerplate any strong model emits.
