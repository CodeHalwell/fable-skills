---
name: privacy-engineering
description: Load when handling personal data in system design — data collection/retention/deletion decisions, PII classification and data mapping, GDPR/CCPA/AI-Act-shaped engineering requirements (DSARs, lawful basis, deletion), anonymization/pseudonymization claims, privacy in ML/LLM pipelines (prompt logging, training data), or privacy design reviews.
---

# Privacy Engineering

## Core mental model

- **Data minimization is the master principle: data you don't have can't leak, can't be breached, can't be subpoenaed, can't be mishandled, and never needs deleting.** Every other privacy control is damage limitation for having collected something. So the review order is: *don't collect* → collect but don't retain → retain but aggregate/pseudonymize → retain raw but restrict and encrypt. Most engineering cultures default to "log everything, keep forever, might be useful" — the expert's default is a required answer to "what decision will this field drive, and when can we delete it?" *before* the column/event/log line exists.
- **Personal data is a liability with interest, not an asset.** It accrues cost continuously: breach blast radius, DSAR scope, deletion-propagation work, regulator exposure, and re-identification risk that *grows* as more external data exists to link against. Model every PII store as a loan you're servicing.
- **You cannot protect data you can't find.** The shadow-copy problem: the primary table is the tip; the mass is in logs, backups, analytics events, data warehouse replicas, caches, search indexes, ML training sets, LLM prompt logs, crash reports, support-ticket screenshots, and CSVs on laptops. Every privacy obligation (delete, disclose, secure) applies to *all* copies — so data mapping (what personal data, where, why, how long, who accesses) is the load-bearing artifact, not the policy PDF.
- **Deletion is an engineering problem people mistake for a feature.** "Delete my account" fans out across every shadow copy, collides with backups and immutable logs, and must beat the ~30-day regulatory clocks. Systems that didn't design for deletion retrofit it painfully; design the propagation path (and the tombstone semantics) when you design the storage.
- **Anonymization claims are almost always overclaims.** Pseudonymized data (stable IDs swapped for hashes/tokens) is still personal data under GDPR — linkage re-identifies it. Treat "anonymized" as a *provable property against an adversary with auxiliary data*, not a synonym for "we removed the name column."

## PII classification and data mapping

- Classify by identifiability *and* harm-if-exposed, in tiers that drive concrete controls: **direct identifiers** (name, email, phone, government ID, device ID, IP — yes, IP addresses and cookie IDs are personal data under GDPR) → **indirect/quasi-identifiers** (ZIP, birthdate, gender — harmless alone, identifying in combination: the classic result is that a large majority of the US population is unique on {ZIP, birthdate, sex}) → **sensitive/special-category** (health, biometrics, sexual orientation, religion, precise location, financials, children's data — extra legal basis and controls, minimize hardest here). A field's tier decides its handling: encryption scope, access gating, retention, whether it may enter analytics at all.
- Make the map executable, not a wiki page: schema annotations (e.g., protobuf field options, dbt column `meta:` tags, BigQuery policy tags) that tooling can enforce — "no `tier=direct` column flows to the analytics dataset" as a CI check beats a data-inventory spreadsheet that rotted the week it was written.
- Hunt shadow copies proactively; the recurring offenders: request/access **logs** capturing emails and IPs in URLs and payloads; **analytics events** with user IDs plus free-text fields; **backups and warehouse snapshots**; **search indexes**; **LLM prompt/completion logs** (see below); **error trackers** (Sentry breadcrumbs with request bodies); **queues/DLQs** holding full user records for weeks. Log scrubbing at the emission point (structured logging with an allowlist of fields, not a denylist of patterns) is worth ten downstream redaction jobs.

## Deletion as an engineering problem

- **Architecture:** a deletion service that receives a `user_id`, fans out to *registered* deleters per store, tracks per-store completion, retries, and produces an auditable "deleted from N stores by date D" record. Registration is the crux — every new store that holds user data must register a deleter to pass design review; unregistered stores are how you end up telling a regulator "we missed the recommendations cache."
- **Tombstone vs hard delete:** hard-delete the payload; *keep* a tombstone (hashed ID + deletion timestamp) where you need to (a) prevent resurrection when an old backup is restored or a laggy pipeline replays events, (b) prove deletion occurred, (c) suppress re-marketing. A tombstone containing only a keyed hash of the ID is the standard compromise between "remember nothing" and "can't prevent resurrection." Re-apply tombstones after every restore — restore-runbooks that skip this step silently un-delete people.
- **Backups:** near-universally accepted practice (and consistent with regulator guidance) is: you need not scrub individual users from cold backups immediately, *provided* backups age out on a bounded schedule (e.g., 30–90 days), are access-restricted, and deletion is re-applied on restore. What's not defensible: "backups are kept indefinitely" — then deletion is a fiction. Bound backup retention or use per-user/per-tenant encryption keys so **crypto-erasure** (destroy the key, data becomes ciphertext garbage everywhere at once, including backups) does the work — crypto-erasure is the only tractable deletion story for immutable/append-only stores and tape.
- **Downstream propagation:** deletion must flow to the warehouse, derived tables, feature stores, and third-party processors (email/analytics/ads vendors — via their deletion APIs, tracked like your own stores). Streaming pipelines: emit a deletion event on the same bus so consumers apply it in-order; retention on the bus itself (e.g., Kafka topic retention) must be shorter than your deletion SLA or handled via compaction with tombstone records.
- **Retention schedules are deletion at scale, automated:** every dataset gets a TTL tied to its purpose at creation time (e.g., raw request logs 30d, support tickets 2y post-closure, financial records per statutory minimum). Enforce with platform TTLs (S3 lifecycle rules, BigQuery partition expiration, Postgres `pg_cron` jobs), not with a policy doc and good intentions. Data past its purpose is pure liability; scheduled deletion is the cheapest privacy control you'll ever ship.

## Regulatory mental model (engineering translation, not lawyer-cosplay)

Encode principles, not statute recitals — the principles are stable across the GDPR, the ~20+ US state comprehensive laws now in force (CCPA/CPRA the archetype), and their international cousins (as of 2026):

- **Lawful basis / purpose limitation →** every dataset has a declared purpose recorded in the data map, and *reuse for a new purpose is a design-review event*, not a query. The concrete version: consent-gated data (marketing analytics) is technically separated or flagged so it can be excluded per-user; "we already have the data" is never sufficient justification for a new use. Consent must be as revocable as it was grantable — which means consent state checks live in the serving/processing path, not only at collection.
- **DSARs (access/portability/deletion/correction) →** rights requests are product features with deadlines (GDPR: one month; CCPA: 45 days, extendable). Access = export everything keyed to this user, in usable form, from *all* mapped stores — which is why the data map and the deletion service share infrastructure: a DSAR exporter is the deletion fan-out running in read mode. Build both once, together. Verify requester identity proportionately (a DSAR is itself a data-exfiltration vector — don't email someone's full history to a spoofed address).
- **Automated decision-making:** meaningful-effect decisions made solely by algorithms (credit, hiring, housing) trigger explanation/opt-out/human-review rights under GDPR Art. 22 and now under US state rules too (California's ADMT regulations were finalized in 2025–2026). Engineering consequence: log model inputs/versions for decisions in scope, and build the human-review path.
- **AI-specific rules are now real, not hypothetical (as of 2026):** the EU AI Act is in force — prohibited-practice and GPAI-model obligations already apply (2025); the high-risk system obligations were slated for August 2026, with a pending "Digital Omnibus" agreement deferring some high-risk deadlines to late 2027 — treat exact dates as legal input, but the engineering posture is settled: classify your AI systems against the high-risk categories (employment, credit, biometrics, essential services...), keep technical documentation, data-governance records, and human-oversight mechanisms. GDPR applies *in parallel* (a DPIA and an AI-Act risk assessment are overlapping but distinct). Also expect: US state AI laws in the hundreds-of-bills stage with 150+ enacted, so multi-state products should build to the strictest common denominator rather than per-state toggles.
- The pragmatic engineering stance: build the *mechanisms* (data map, deletion/export service, consent state, retention TTLs, purpose tags, decision logs) that all these regimes demand in different words, and let counsel map mechanisms to statutes. When counsel and engineering disagree on feasibility, the data map is the shared ground truth — most fights dissolve into "we didn't know that store existed."

## Anonymization honesty

- **Pseudonymization ≠ anonymization.** Replacing user IDs with `sha256(user_id)` changes nothing an adversary cares about: the mapping is trivially brute-forceable for enumerable identifiers (emails, phone numbers), and even unguessable tokens preserve *linkability* — the row-set per person is intact, and linkage attacks (join on quasi-identifiers with a voter file, a data-broker file, or your own other datasets) re-identify. Pseudonymization is a good *security* control (breach of one store reveals less) and is explicitly still personal data under GDPR. Never let it flip a dataset's classification to "not PII."
- **K-anonymity and its limits:** generalize quasi-identifiers until every row is identical to ≥k−1 others. Real but weak: homogeneity (all k share the sensitive value → learned anyway), composition (two k-anonymous releases of the same data intersect to unique rows), and high-dimensional data (location traces, browsing histories, embeddings) can't be usefully k-anonymized at all — a handful of spatio-temporal points uniquely identifies most people. Use k-anonymity as a *release hygiene bar* for low-dimensional tabular exports, never as a proof.
- **Differential privacy is the only definition with an actual guarantee** (output nearly unchanged whether or not any one person's data is included, quantified by ε), and it's deployed for real (US Census 2020, Apple/Google telemetry). Practical guidance: it fits *aggregate* releases (counts, histograms, model training via DP-SGD) with a managed privacy budget; it does not give you "anonymized row-level data" — no technique honestly does. Use vetted libraries (Google DP building blocks / PipelineDP, OpenDP, PyTorch Opacus), keep ε single-digit-ish and *published*, and account for cumulative budget across repeated queries — a DP system without budget tracking is DP theater.
- The honest decision rule for "can we share this dataset?": either aggregate with DP, or treat it as personal data with contracts and access controls. "We removed the obvious columns" is the answer that ends up in a journalist's re-identification story (Netflix Prize, AOL search logs, NYC taxi logs — the genre is evergreen).

## Privacy in ML and LLM systems

- **Training-data memorization is real:** large models verbatim-regurgitate rare training strings (the extraction-attack literature is mature), so "the model isn't the data" is not a defensible position for models trained on raw PII. Controls in practice: aggressively dedupe and PII-scrub training corpora (memorization concentrates on repeated and rare-unique strings), prefer DP-SGD where the utility budget allows, red-team with extraction prompts (canary strings inserted at training time give you a measurable memorization dial), and record dataset lineage so a deletion request can answer "was this user in the training set, and which model versions?" Honest current answer to "delete me from the model": exclude from future training runs and retrain/deprecate on schedule — machine unlearning is not production-ready as of 2026, so retention policy for *models* (how long does a model trained on user data serve?) is the real mechanism.
- **Prompt/completion logs are the new shadow copy of record.** LLM apps log full prompts for debugging/evals — and prompts carry whatever users paste into them (contracts, medical notes, credentials). Policy to implement, not just write: default retention short (days–weeks) with TTL enforcement; PII scrubbing or at minimum flagged-field redaction before logs reach the eval/analytics tier; separate consent/contract terms for "may we train on your inputs" with the flag enforced *in the training-data pipeline* (an unenforced opt-out flag is a breach of contract at training time); zero-retention or in-region processing options for enterprise tiers (this is now a standard procurement demand — vendor LLM APIs offer zero-data-retention modes; know whether yours is on). RAG adds its own hole: the vector store is a copy of source documents — deletion must propagate to chunks/embeddings (delete-by-document-ID in the index), and per-user authorization must be enforced *at retrieval time*, or the index launders access control away from the source system.
- Embeddings are not anonymous: they're invertible enough to recover substantial input content and are linkable; classify embedding stores at (or near) the tier of their source text.

## Encryption scoping

- At-rest (disk/volume/bucket) and in-transit (TLS) encryption are table stakes that mostly protect against *stolen disks and passive taps* — they do nothing about the actual top risks: application bugs, over-broad internal access, and leaked credentials, because the app and anyone with DB credentials sees plaintext. Never let "the database is encrypted" close a privacy review.
- **Field-level encryption** (encrypt specific columns with application/KMS-held keys) is the control that changes the threat model: DB compromise or warehouse over-sharing yields ciphertext for the protected fields. Its real cost is key management and *lost functionality* — you can't index, search, join, or `LIKE` on ciphertext (deterministic encryption restores equality-joins at the cost of frequency-analysis leakage; blind indexes/HMACs restore exact-match lookup). So scope it: field-level for the highest-tier fields (government IDs, health data, tokens/secrets), per-tenant or per-user keys where crypto-erasure or tenant isolation is the goal, plain at-rest for the rest. Use envelope encryption via a KMS (AWS KMS/GCP KMS/Vault): data keys wrap fields, master keys wrap data keys, rotation rotates wrapping (not a full re-encrypt), access to decrypt is IAM-audited per service. That audit trail — *which service decrypted whose SSN when* — is half the value.
- Tokenization (swap value for a vault-held token) suits payment data and lets most systems handle only tokens; format-preserving variants exist for legacy schemas. Confidential computing and homomorphic schemes remain niche as of 2026 — reach for them only with a named threat they uniquely address.

## How an expert thinks through it

*"PM wants session replay on the checkout flow to debug drop-offs."*

Minimization first: the decision they'll make is "which step loses users and why" — do we need *replay* (screen recordings = keystrokes, addresses, card fields, a PII firehose) or would funnel events + rage-click/error events per step answer it? Push for the events version first; it's probably 80% of the value at 2% of the liability. Suppose replay is genuinely needed for the visual bugs: then scope it — record checkout pages only; masking is allowlist-based (mask everything, unmask specific safe elements), because denylist masking ("block fields named `card`") fails on the one custom component someone renames; verify the replay vendor's masking happens *client-side before upload* (if raw DOM leaves the browser, we've shipped PII to a processor regardless of what their dashboard hides — this is exactly the mistake in several public session-replay scandals). Vendor = processor: DPA in place, added to the data map, retention 30 days, deletion API wired into our deletion service, and replay capture gated on the analytics-consent flag in regions that require it. Rejected: "just hash the card number in the recording" (masking, not hashing, is the mechanism, and quasi-identifiers in a recording aren't hashable anyway); rejected "keep recordings 1 year for trend analysis" — the purpose was debugging *this* funnel; trends come from the aggregate events, which is what purpose limitation means in practice. Ship condition: a canary test asserting that a fake card number typed into checkout never appears in a captured replay payload — a falsifiable check, not a policy sentence.

## Failure modes & pitfalls

- **Denylist log scrubbing** (`regex out things that look like emails`) — fails on new fields, nested JSON, URLs-with-tokens, and base64 blobs. Correction: structured logging with an *allowlist* of loggable fields per event type, enforced in the logging library, plus a scanner (e.g., regular audits of log samples with a PII detector) as the backstop — the scanner is the alarm, the allowlist is the control.
- **`sha256(email)` shipped to a vendor as "anonymous matching."** Enumerable-input hashes are pseudonyms at best; industry ID-matching on hashed emails is *linkage by design*. Call it what it is in the data map and gate it on the proper consent basis.
- **Deletion job that only covers the primary DB.** The warehouse ETL re-materializes the user next run (or worse, the nightly backup restore does). Corrections: deletion fans out through the registry to every store; tombstones checked by ETL and restore runbooks; a *verification* job that greps all stores for a deleted test-user's identifiers and alarms on hits.
- **Consent recorded but not enforced**: the opt-out flag lives in the users table while the analytics SDK fires unconditionally, or the "don't train on my data" flag isn't read by the training-corpus builder. Consent state must be checked at the *point of collection/processing*, and there must be a test proving flag-off → event-absent.
- **DSAR export built as a one-off SQL script** by whoever got the first request — misses eleven stores, takes three weeks, gets emailed as an unencrypted zip. Build the exporter on the deletion fan-out, with identity verification and authenticated delivery, before the first request arrives.
- **Retention policy in a PDF, TTLs nowhere.** Five years later the "30-day" logs are a 400TB GDPR liability with unknown contents. Every dataset creation includes the platform-enforced expiry (S3 lifecycle, partition expiration) in the same PR.
- **The analytics warehouse as PII bypass:** production access is locked down, but the warehouse replicates every column to everyone with a BI seat. Corrections: column-level policy tags, tiered datasets (PII-stripped by default; identified data by grant), and the CI rule that direct identifiers don't enter the default tier.
- **LLM prompt logs with indefinite retention feeding evals** — user-pasted secrets and health data flow into eval datasets viewed by contractors and possibly into fine-tuning. Short TTL, redaction before the eval tier, and the training-opt-out flag enforced in the corpus builder (with a lineage record per training run).
- **RAG index that outlives the document** (source doc deleted, chunks still retrievable) or that ignores source ACLs (index built with admin credentials, queried by everyone). Deletion and authorization both must be first-class in the retrieval layer.
- **Backup restore resurrecting deleted users** because the runbook has no re-apply-tombstones step. Add it to the runbook *and* to the restore automation, and test it in the next restore drill.
- **Treating encryption-at-rest as the answer to any privacy question.** It answers exactly one: stolen media. Access scoping, field-level crypto, and minimization answer the rest.

## Worked micro-examples

**1. Retention enforced by the platform, in the same PR that creates the data:**

```hcl
# Terraform: raw request logs live 30 days, then Glacier for the statutory tail, then gone.
resource "aws_s3_bucket_lifecycle_configuration" "request_logs" {
  bucket = aws_s3_bucket.request_logs.id
  rule {
    id     = "retention-30d"
    status = "Enabled"
    expiration { days = 30 }
    noncurrent_version_expiration { noncurrent_days = 7 }  # versioned buckets keep "deleted" objects — expire those too
  }
}
```

```sql
-- BigQuery: analytics events partition-expire at 400 days; PII-tier columns never arrive (enforced upstream).
ALTER TABLE analytics.events SET OPTIONS (partition_expiration_days = 400);
```

The versioned-bucket rule is the one everyone forgets: with versioning on, `DELETE` writes a marker and retains the bytes.

**2. Deletion fan-out with a registry — new stores must enroll to pass review:**

```python
DELETERS: dict[str, Deleter] = {}          # store name -> deleter; design review checks membership

def register(store: str):
    def wrap(fn): DELETERS[store] = fn; return fn
    return wrap

@register("postgres.users")
def delete_users_row(user_id): ...
@register("s3.avatars")
def delete_avatar_objects(user_id): ...
@register("pinecone.support-rag")
def delete_rag_chunks(user_id): ...        # embeddings and chunks are copies too
@register("vendor.customerio")
def delete_from_email_vendor(user_id): ... # processors, via their deletion API

def delete_user(user_id: str):
    write_tombstone(hmac_sha256(TOMBSTONE_KEY, user_id))   # keyed hash: prevents resurrection, stores no identifier
    for store, fn in DELETERS.items():
        enqueue_with_retries(store, fn, user_id, sla_days=25)  # tracked per store, alarms before the 30-day clock
```

The DSAR exporter iterates the same registry in read mode — one map, two obligations.

**3. Allowlist logging — the control, not the regex backstop:**

```python
LOGGABLE = {
    "http.request": {"method", "route", "status", "duration_ms", "user_id_hash"},  # note: route template, not raw URL
}

def log_event(event: str, **fields):
    allowed = LOGGABLE[event]                       # unknown event type -> KeyError at dev time, not PII in prod
    dropped = fields.keys() - allowed
    if dropped:
        metrics.incr("log.fields_dropped", tags={"event": event})  # visible pressure to fix the call site
    logger.info(event, **{k: v for k, v in fields.items() if k in allowed})
```

Raw URLs are excluded deliberately (query strings carry emails and tokens); the route *template* plus a hashed user ID keeps logs debuggable and DSAR/deletion-scope small.

## Verification / self-check

Privacy-by-design review checklist — run it on any feature touching personal data, and demand artifact-level answers:
1. **Collection:** for each new field/event — what decision does it drive? Could a coarser/aggregated/absent version work? (Reject "might be useful.")
2. **Mapping:** is every new store/vendor in the data map with classification tiers, purpose, and owner? Is the vendor under a DPA with a deletion API?
3. **Retention:** what's the TTL, enforced by which platform mechanism, visible in this PR?
4. **Deletion & DSAR:** is a deleter registered for every new store? Does the exporter cover it? What's the end-to-end deletion SLA vs. the 30/45-day clocks?
5. **Consent/purpose:** which basis covers this, where is the flag checked in code, and what test proves opt-out works?
6. **Exposure:** who can read this data (humans and services), is that IAM-auditable, do the highest-tier fields get field-level encryption, and do logs/analytics/LLM-prompts stay clean of it?
7. **Claims audit:** does anything in the design doc say "anonymized"? Make it prove the adversary model or rename it "pseudonymized."

Falsifiable end-to-end test (the one that matters): create a synthetic user, exercise the product fully (including support tickets, analytics, an LLM feature prompt), request deletion, wait the SLA, then search *every* mapped store — and one unmapped place, the logs — for the synthetic identifiers. Zero hits = the system works; any hit = your data map was fiction. Run it quarterly.

Stopping rule: minimized collection, executable data map, TTLs enforced, deletion/export fan-out tested by the synthetic-user drill, consent enforced in code, highest-tier fields under field-level crypto — that's a defensible system; further effort goes to the *newest shadow copies* (your latest pipeline, your newest vendor, your LLM logs), because privacy posture decays by accretion, not by dramatic failure.
