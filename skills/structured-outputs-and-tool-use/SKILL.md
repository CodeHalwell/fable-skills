---
name: structured-outputs-and-tool-use
description: Load when getting machine-readable output from an LLM — JSON/schema-constrained generation, function/tool calling, agent tool design, parsing or validating model output, or debugging malformed/hallucinated structured responses (bad enums, stringified JSON, escaping errors, wrong tool arguments).
---

# Structured Outputs and Tool Use

## Core mental model (anchors — the field's settled doctrine; apply without exception)

- The schema is a prompt: every field name, description, enum value is read as instructions; field order = generation order = conditioning order (reasoning field first, answer last).
- Constrained decoding guarantees syntax, never semantics — and hard constraints *mask* uncertainty by forcing a legal-looking token when the model wanted to refuse; every enum gets an escape value, every schema a `cannot_comply` path, and escape-usage rates get monitored.
- Validate → repair (≤2 attempts, with the specific validator error + prior output verbatim) → typed failure, never an exception or infinite loop. Track repair rate as a regression signal; a rate pinned at zero may mean the schema is too loose to catch anything.
- Every tool call in a turn gets a result matched by id — including rejected calls (error payload), or the API 400s a turn later / the model hallucinates the missing outputs.
- Check `stop_reason` before diagnosing malformed JSON — truncation at max_tokens is the #1 phantom "parse error"; the #2 is reading the prose channel instead of `tool_use.input`.
- Pass handles, not payloads: the model regenerates every token of an echoed ID or document, with per-token corruption risk; short references dereferenced in code.
- Provider structured-output modes enforce a schema *subset*; keywords are often accepted but not enforced (`pattern`, `format`, `minimum`, `oneOf`) — contract-test which constraints actually bite, per provider per model version, and validate the full schema application-side regardless.

## The corrections (residual gaps beyond the doctrine)

**Field order has two masters — choose per use case, explicitly.** Reasoning-first ordering (reason → answer) maximizes answer quality; streaming UX wants the *displayable* field first so it renders as it generates. These directly conflict, and the default (reasoning first everywhere) silently gives streaming users a long dead pause. Decide per endpoint: machine-consumed → reasoning first; streamed-to-human → displayable field first and accept the quality tradeoff, or split into two calls.

**Track per-field error rates, not per-response.** One chronically wrong field (fix: its description) is indistinguishable from diffuse randomness (fix: decompose the task) in a response-level metric — and the fixes are opposite. A field-level error dashboard is cheap and converts "the model keeps getting it wrong" into "the `submitted_at`/`created_at` ambiguity is 80% of errors."

**Build the malformed-output corpus.** Every production response that ever failed validation goes into a fixture set; the extractor/repair path is unit-tested against all of it. This is your parser's regression suite, independent of any model — without it, every parser "improvement" is tested only against the happy path, and old failure shapes recur on model updates.

**Enum lists belong in the description too.** In prompt-and-parse mode (and as belt-and-braces under constrained mode), put the allowed values verbatim in the field description, not only in the schema — schema-only enums are the ones models drift from with plausible siblings ("urgent" for high). Never silently fuzzy-match drift to the nearest value: that's a misclassification laundering machine; use an explicit, logged alias table if you normalize at all.

## Compressed design rules (one-liners)

- Flat beats nested (≤2 levels; prefix keys instead: `shipping_city`); enums ≤~20 values, hierarchical selection past that; every field description states semantics, units, format, and the null/absent rule; nullable-with-instructions beats required-and-fabricated.
- Business validation (regex, ranges, cross-field) goes in description + code, not in the schema you send — unevenly enforced either way.
- Booleans named so `true` is unambiguous; dates pinned to one format with timezone policy.
- Split calls past ~15–20 leaf fields or mixed concerns; keep together when fields are interdependent; discriminated union (`kind` enum first) for mutually-exclusive blocks; long docs → per-chunk list extraction, merge in code.
- Tool descriptions: what + when + when NOT + returns + failure behavior; >~15–20 tools degrades selection (namespace, route, or per-phase subsets); tool *results* are prompts — actionable errors, not stack traces or 50 kB blobs.
- Streaming: incremental parser (never `json.loads`-and-retry on partials); commit a field only after its closing delimiter (streamed `1200` may become `12000`; enum prefix may match a shorter value); accumulate tool-arg deltas per call id; dispatch only on finalization.
- Parallel calls: executor handles a *list* per turn (not `tool_calls[0]`); serialize side-effecting calls (`disable_parallel_tool_use` / `parallel_tool_calls=false`) and dedupe identical mutating calls in one turn; idempotency keys in the tool implementation, not in the model's good intentions.
- Agent loops: cap iterations with one final no-tools summary turn; inject a course-correction note on repeated same-args calls; truncate old tool results, never the current turn's.
- `finish_reason == "tool_calls"` does not mean valid args — validate with full rigor before side effects.

## Worked micro-example — the redesign move that ends "the model keeps getting it wrong"

Failing pattern: 3-level nesting + free-text `category` + a field *described as* "JSON string" + unanchored 1–10 scale + no absent-value path (~12% invalid/wrong-field). Rewrite mechanically: flatten to prefixed keys, enumerate `category` with `other` + `category_other_note`, type `metadata` as a real object, replace the scale with anchored labels (`angry|frustrated|neutral|satisfied`), give every field a null rule. Typical result: ~12% → ~1–2% with no model or prompt change — schema redesign is the first move, not more prompt engineering.

## Verification checklist

- [ ] Every field described; enums escaped and listed in descriptions; refusal path in schema; escape-usage monitored.
- [ ] Field order chosen deliberately (reasoning-first vs streamed-field-first) per endpoint.
- [ ] Repair loop bounded with typed failure; repair rate and *per-field* error rates on a dashboard.
- [ ] Executor returns a result for every call id incl. rejections; side-effecting calls serialized and deduped.
- [ ] Contract test pins provider schema-subset enforcement; malformed-output corpus regression-tests the parser.
- [ ] Tested with: a refusal-tempting input, an "unknown"-correct input, a near-max_tokens input, and a streaming consumer.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 15 claims: 14 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Biggest baseline gaps found:
  - Opus reasons field-order-as-conditioning correctly but misses the direct conflict with streaming UX (displayable-field-first) and that it must be chosen per endpoint.
  - Response-level vs per-field error attribution (opposite fixes) and the malformed-output fixture corpus as a model-independent parser regression suite not surfaced.
  - Otherwise near-ceiling: constrained-decoding refusal-masking, rejected-call results, truncation-first diagnosis all produced cold.
