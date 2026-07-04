---
name: structured-outputs-and-tool-use
description: Load when getting machine-readable output from an LLM — JSON/schema-constrained generation, function/tool calling, agent tool design, parsing or validating model output, or debugging malformed/hallucinated structured responses (bad enums, stringified JSON, escaping errors, wrong tool arguments).
---

# Structured Outputs and Tool Use

## Core mental model

- **The schema is a prompt.** The model reads every field name, description, enum value, and nesting level as instructions. `"category"` with no description invites hallucination; `"category": one of the listed support-queue names; use "other" if none clearly applies` is a micro-prompt that fixes it. Design schemas for the model first, your type system second.
- **Constrained decoding guarantees syntax, never semantics.** Grammar/schema-enforced generation ("structured outputs" modes, `outlines`, `xgrammar`) makes invalid JSON impossible, but the model can still put the wrong value in every field — and hard constraints can *mask* confusion by forcing a token the model didn't want (the model "wanted" to refuse or hedge; the grammar forced a legal-looking answer). Syntactic validity is the floor, not the goal.
- **Tool-definition quality is the main lever on tool-use reliability.** When an agent picks the wrong tool or malforms arguments, the fix is almost always in the tool's name/description/parameter schema — not in the system prompt and not a model upgrade. Budget prompt-engineering effort accordingly: descriptions of tools are the highest-leverage text in an agentic system.
- **Validate, then repair, then bound the loop.** Every parse must be followed by semantic validation; every validation failure should get one or two targeted repair attempts; the loop must terminate in a typed error, never an exception or an infinite retry.
- **Every enum needs an escape hatch.** A forced choice among N wrong options produces a confidently wrong answer. `"other"`/`"unknown"` variants plus a place to say why convert silent misclassification into detectable uncertainty.

## Decision framework: constrained decoding vs prompt-and-parse

| Situation | Choose | Reasoning |
|---|---|---|
| Output feeds code directly (API args, DB writes, pipelines) | Schema-constrained mode (API "structured outputs" / tool-use with `input_schema`) | Parse failures become impossible; retries and repair code disappear. |
| Task needs reasoning quality and structure | Two-step: free-form reasoning first, then a second constrained call (or a reasoning field placed *before* the answer fields) | Forcing immediate structure suppresses chain-of-thought; measurable quality drop on hard tasks. Field order is generation order — the model writes fields in schema order, so put `reasoning` first and `answer` last. |
| Model/provider has no constrained mode | Prompt-and-parse: demand fenced JSON, parse leniently, validate strictly | Extract the first balanced `{...}` block instead of `json.loads(raw)` on the whole response; models add prose despite instructions. |
| Schema is highly dynamic/recursive or constrained mode rejects it | Prompt-and-parse + validation loop | Provider schema subsets are limited (recursion, complex unions, format keywords often unsupported); a clean prompt beats a mangled schema. |
| You need a refusal/uncertainty path | Either mode, but the schema must include it | Add `{"status": "ok" | "cannot_comply", "reason": ...}`. Under hard constraints a model that wants to refuse otherwise emits fabricated-but-valid data — the worst failure class because nothing flags it. |

## Schema design rules for LLMs

1. **Flat beats nested.** Every nesting level multiplies structural error modes (in prompt-and-parse) and dilutes attention (in both modes). Two levels is a practical ceiling; if you're at four, redesign — usually by splitting into multiple calls or flattening with prefixed keys (`shipping_city`, not `shipping.address.city`).
2. **Enums beat free text** for anything downstream code branches on. Free-text `"severity": "kinda bad"` is unusable; `enum: ["low","medium","high","unknown"]` is testable. Keep enums ≤ ~20 values in one field; past that, use hierarchical selection (two calls) or retrieval over the option list.
3. **Every field gets a description** stating semantics, units, format, and edge-case behavior: `"delay_minutes": integer minutes, 0 if on time, null if not yet departed`. Ambiguity between similar fields (`created_at` vs `submitted_at`) is where wrong-but-valid values breed.
4. **Prefer `null`-able fields over omitted fields** and say explicitly when to use null. Models fill "required but sometimes unanswerable" fields with plausible fabrications rather than violating the schema.
5. **Don't encode business validation in the schema you send the model** (regex `pattern`, `minimum`, cross-field rules are unevenly enforced by providers and poorly obeyed as instructions). State the rule in the description AND check it in code.
6. Booleans: name them so `true` is unambiguous (`is_refund_requested`, never `status_flag`). Dates: demand one format explicitly (ISO 8601, with timezone policy stated) — otherwise you get a locale lottery.

## Tool/function definition quality

- **Description formula:** what it does + when to use it + when NOT to use it + what it returns + failure behavior. The "when not to use" line is what prevents the classic overlapping-tools bug (model calls `search_web` when `search_docs` was right). If two tools overlap, either merge them or make the boundary explicit in both descriptions.
- Fewer, well-named tools beat many granular ones; past roughly 15–20 tools, selection accuracy degrades — group with namespaced names (`db_query`, `db_insert`) and consider exposing subsets per task state.
- Tool *results* are prompts too: return concise, structured, model-readable results with explicit errors (`{"error": "no rows matched filter X — try broadening the date range"}`), not raw stack traces or 50 kB blobs. A good error message is a course correction; an empty `[]` with no explanation causes retry loops with the same bad arguments.
- Never make the model echo large IDs/documents through tool arguments — pass short handles (`"doc_3"`) and dereference in code. Long copied strings get truncated or corrupted mid-token.

## Parallel vs sequential tool calls

- Models emit **parallel calls** (several tool calls in one assistant turn) when calls look independent. Your executor must handle a *list* of calls per turn — the classic bug is executing only `tool_calls[0]` and dropping the rest, which desyncs the conversation state.
- Every emitted call needs a result message matched by its `id`, in the same order/turn structure the API expects — including calls you *rejected*: return an error result for them, never silently omit (most APIs hard-error on missing tool results; the failure surfaces as a confusing 400 one turn later).
- Force sequential execution when calls have side effects or ordering dependencies (write-then-read, transfer-then-confirm): either disable parallel calls via the API option (e.g., Anthropic `disable_parallel_tool_use`, OpenAI `parallel_tool_calls=false`) or execute serially and fail-fast on the first error.
- When the model *should* parallelize (3 independent lookups) but issues them one per turn, say so in the system prompt ("issue independent tool calls together in a single turn") — it's a latency win worth prompting for.

## Validation-and-repair loops

```python
from pydantic import BaseModel, ValidationError

def structured_call(prompt: str, model_cls: type[BaseModel], max_repairs: int = 2):
    raw = llm(prompt)
    for attempt in range(max_repairs + 1):
        try:
            data = extract_json_block(raw)          # first balanced {...}, not json.loads(raw)
            obj = model_cls.model_validate(data)    # syntax + types + enums
            check_semantics(obj)                    # cross-field rules, ranges, referential checks
            return obj
        except (ValidationError, SemanticError) as e:
            if attempt == max_repairs:
                return Failure(raw=raw, error=str(e))   # typed failure, never raise to caller
            raw = llm(f"{prompt}\n\nYour previous output failed validation:\n{e}\n"
                      f"Previous output:\n{raw}\nReturn corrected JSON only.")
```

- Repair with the *specific* error and the previous output — a blind "try again" resample fixes random flakes but not systematic misunderstandings; the error message fixes both.
- Two repairs max. If it fails twice, the schema or prompt is wrong; log these cases — they are your schema-improvement backlog.
- Validate semantics, not just shape: enum drift, IDs that must exist in your DB, sums that must match, dates in valid ranges. Pydantic passing means the JSON is shaped right, nothing more.
- Track repair rate as a metric. A rising repair rate after a prompt/model change is a regression signal even when end success rate looks flat (you're paying 2–3× tokens and latency).

## Streaming / partial structured output

- Never feed a partial stream to `json.loads` and retry on failure as a "progress" mechanism — use an incremental parser (`jiter.from_json(..., partial_mode=True)`, or best-effort partial-JSON parsers) or an SDK's typed streaming helpers.
- Design schemas so the *displayable* field streams well: put the long free-text field (`answer_markdown`) last is wrong for streaming UX — put it **first** if you want to render it as it streams, since fields generate in schema order. (Note the tension with reasoning-first ordering; pick per use case.)
- Don't act on any value until its closing delimiter has arrived: a streamed `"amount": 1200` might still become `12000`. Commit field-by-field only when the parser confirms the field is complete.
- Tool-call arguments stream as string deltas that must be accumulated per call `id` (parallel calls interleave); dispatch only on the stop/finalization event, never on a "looks complete" heuristic.

## Decomposition: when one call should be several

- Split when: the schema exceeds ~15–20 leaf fields, mixes unrelated concerns (extract entities AND classify AND summarize), or requires reasoning quality on one field while others are mechanical. Accuracy per field degrades as schema size grows; two focused calls routinely beat one omnibus call on both quality and debuggability, at modest extra cost.
- Keep together when: fields are strongly interdependent (a classification that determines which other fields make sense) — splitting forces you to thread state between calls and re-send context.
- For long-document extraction, chunk the document and extract per chunk into a *list* schema, then merge/dedupe in code. Asking for one giant object over a 100-page input maximizes both truncation risk and missed items.
- Conditional structure: instead of one schema with many mutually-exclusive optional blocks, use a discriminated union pattern — first field is `"kind"` (enum), description states which fields apply per kind, validator enforces the correlation. Models handle "fill only the fields for your chosen kind" poorly without this explicit structure.

## Agent-loop robustness (tool use over many turns)

- Cap iterations (typical: 10–25 depending on task) and make the cap's behavior explicit: on hitting it, the model gets one final no-tools turn to summarize partial progress, rather than the loop dying mid-thought.
- Detect repetition: same tool + semantically-same arguments twice in a row is a stuck loop; intervene by injecting a user-role note ("that call already failed with X; try a different approach") rather than letting it burn the budget.
- Truncate/summarize old tool results as the transcript grows — but never truncate the *current* turn's results or the system prompt. Giant accumulated tool outputs are the top cause of context-limit failures and degraded late-turn reasoning in agent loops.
- Idempotency: assume any tool call can be issued twice (retries, regenerations). Side-effecting tools need idempotency keys or precondition checks in the tool implementation — do not rely on the model not to repeat itself.

## Failure modes & pitfalls

- **Hallucinated enum values:** model returns `"priority": "urgent"` when the enum is `["low","medium","high"]` — synonyms and plausible siblings, especially in prompt-and-parse mode. Fix: enum list verbatim in the field description (not only the schema), add `"other"`, validate with exact membership, repair-with-error. Never "fix" by fuzzy-matching to the nearest enum silently — that's a misclassification laundering machine.
- **Stringified nested JSON:** `"metadata": "{\"key\": \"value\"}"` — a JSON object encoded as a string, common when examples in the prompt show escaped JSON or when the field description says "JSON string". Fix: describe nested fields as objects, show unescaped examples; detect with `isinstance(v, str) and v.lstrip().startswith(("{","["))` and decode-then-revalidate rather than failing.
- **Unicode/escaping corruption:** raw newlines inside JSON strings (illegal), `\u` sequences mangled, smart quotes breaking naive parsers, trailing commas, and single quotes. Fix: lenient extraction pass (a tolerant parser) followed by strict validation; never regex-“repair” quotes blind — you'll corrupt legitimate apostrophes.
- **Markdown fences despite "JSON only":** always strip ```` ```json ```` fences / extract the balanced block before parsing; treat "the model added prose" as normal, not exceptional.
- **Truncation mistaken for malformation:** output hit `max_tokens` mid-object; the "parse error" is actually a length limit. Check the stop/finish reason *before* diagnosing JSON errors; fix by raising the limit or shrinking the schema (drop echo-fields the model shouldn't repeat).
- **Schema-echo waste:** making the model copy the full input into the output "for traceability" doubles cost and creates corruption opportunities; join on an ID instead.
- **Constrained mode silently downgrading:** some providers fall back or error on unsupported schema features (deep recursion, `anyOf` unions, `patternProperties`); test the exact schema against the exact provider, and pin behavior with a contract test in CI.
- **Trusting `finish_reason == "tool_calls"` implies valid args:** argument strings can still be malformed or violate your semantics; validate tool arguments with the same rigor as final outputs before executing side effects.
- **Executing side-effecting parallel calls concurrently** (two `transfer_funds` calls the model duplicated): dedupe identical calls in one turn and gate irreversible actions behind sequential confirmation.

## Worked micro-example: redesigning a schema that "the model keeps getting wrong"

Failing schema (extraction from support emails, ~12% invalid or wrong-field rate):

```json
{"customer": {"contact": {"email": "string", "phone": "string"}},
 "issue": {"details": {"category": "string", "product_line": "string",
                       "meta": "string  // JSON string with extra fields"}},
 "sentiment": "number 1-10"}
```

Expert diagnosis: 3-level nesting (structural errors + attention dilution), free-text `category` (unbounded values downstream can't branch on), a field *described as* a JSON string (guarantees stringified-JSON output), an unanchored numeric scale (scores cluster 6–8, meaningless), and no way to say "not present in the email."

Redesigned:

```json
{
  "customer_email": "string or null — null if no email address appears in the message",
  "customer_phone": "string or null — digits and '+' only, null if absent",
  "category": "one of: billing | shipping | product_defect | account_access | other — use other when no listed value clearly applies",
  "category_other_note": "string or null — only when category is other: one sentence on why",
  "product_line": "one of: <your 8 SKU families> | unknown",
  "is_refund_requested": "boolean — true only if the customer explicitly asks for money back",
  "sentiment": "one of: angry | frustrated | neutral | satisfied"
}
```

Every change is mechanical application of the rules above: flatten, enumerate, describe, add null/other escape hatches, replace the numeric scale with anchored labels. Typical result of exactly this kind of rewrite: invalid/wrong-field rate drops from ~12% to ~1–2% with no model or prompt change — which is why schema redesign is the first move, not more prompt engineering.

## Verification checklist

- [ ] Every field has a description; enums have an escape value; refusal/uncertainty path exists in the schema.
- [ ] Nesting ≤ 2 levels; nothing downstream-branchable is free text; field order matches generation needs (reasoning first or streamed field first — chosen deliberately).
- [ ] Parser extracts a balanced block (or uses constrained mode); validator checks semantics beyond shape; repair loop bounded with typed failure; repair rate logged.
- [ ] Executor handles N tool calls per turn, returns a result for every call id (including rejections), and serializes side-effecting calls.
- [ ] Tested with: adversarial input designed to tempt a refusal, an input where the right answer is "unknown", an input near max_tokens, and a streaming consumer — before declaring the pipeline reliable.
