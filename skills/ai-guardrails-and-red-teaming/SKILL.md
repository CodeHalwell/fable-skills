---
name: ai-guardrails-and-red-teaming
description: Defensive AI safety engineering — layered guardrails (input/output classification, action permissions, escalation), prompt-injection defense in agentic systems, jailbreak detection taxonomy, red-teaming your own deployment, agentic risks (tool authorization, exfiltration channels), and incident response. Load when hardening an LLM app or agent against attacks, designing guardrail stacks, building red-team/regression suites, or reviewing AI-security posture.
---

# AI Guardrails and Red-Teaming

Assumed baseline (verified expert-grade cold): injection is unsolved — design for blast radius, not prevention; the layer ranking with security living in deterministic action permissions + human gates, prompts/classifiers as noise reduction only; Meta's Rule of Two (≤2 of {untrusted input, sensitive access, external side effects}); the indirect-injection kill chain with enforcement at egress (markdown-image/link exfil → CSP/image proxy/link rewrite/domain allowlists); CaMeL's privileged-planner + quarantined-reader + provenance-tracking interpreter (~0% ASR on protected AgentDojo flows); jailbreak taxonomy with crescendo/multi-turn invisible to per-turn classifiers by construction → trajectory-level scoring; attack-tree-driven red-teaming scored at both model-compliance and kill-chain-completion levels (the gap = what layers 4–5 are worth); PyRIT/garak/promptfoo for breadth + manual depth, every successful probe becoming a CI regression test rerun on every model bump; taint-based capability gating; pre-built kill switches at per-tool granularity, degradation ladders, forensic logging, rehearsed rollback; excessive agency = excessive functionality/permissions/autonomy; "blocked 98%" is not a security metric — report kill-chain completions, per-layer catches, and residual blast radius.

## Discipline rules

- Trace every safety claim to a layer: per attack-tree leaf, name the deterministic control (never the prompt line), the detection if it fails, and the log that proves what happened. A leaf answered by "the model should refuse" is an open finding.
- Launch bar: every leaf structurally blocked, human-gated, or accepted-and-logged with a named owner — not "probes stopped succeeding" (they won't). The honest residual-risk sentence appears in the launch review.
- Adding one state-changing tool to a read-only assistant reopens the whole analysis — that's the moment chat-with-docs becomes an agent.
- Humans confirm the **exact rendered action** (full recipients + body), never a paraphrase; deny-by-default for tools without a policy entry.

## Sharpenings the strong baseline lacks

- **Standards naming (as of 2026)**: OWASP's **Top 10 for Agentic Applications (released Dec 2025)** now sits alongside the LLM Top 10 as the agent checklist — cite it, not just the 2025 LLM list. Classifier options: Llama Guard family (Llama Guard 4 is multimodal, MLCommons hazard taxonomy), NVIDIA NemoGuard, Lakera inline, provider moderation endpoints; budget ~10–50ms and real FP/FN rates.
- **MCP/tool-description poisoning is ingress**: third-party tool *metadata* is attacker-controlled — 2025–26 tool-poisoning research showed most agents follow instructions embedded in unrelated tools' descriptions. Scan and taint tool descriptions and tool results, not just documents.
- **Credential design beats intent design**: per-user token exchange, audience-bound tokens, least-privilege scopes — the agent asking nicely is UX; the credential not having the permission is security. Spec-level rule: never pass a client's token through to downstream APIs. A fully hijacked agent bounded by the user's own rights is a contained failure.
- **Taint policy that keeps the product usable**: classifier hits on inbound content don't block (newsletter imperatives would make FP rates intolerable) — they taint the session, dropping dangerous capabilities (no new recipients/domains/writes) and adding a visible banner. Blocking-on-classifier and prompt-only defenses are the two designs to reject explicitly.
- **Probe economics**: 50–200 probes derived from *your* attack tree (payload styles × ingress points, including split-across-ingress and side channels like .ics attachment bodies) beat 5,000 public jailbreaks — public corpora test the base model's language, not your tools. Tag each regression test with the tree leaf it exploits.
- **Classify at three points plus a periodic whole-conversation pass** (raw input, normalized/decoded input, output) — encoding attacks beat plain-text input classifiers by construction; output-side checks are the net for every input-side miss.
- Spotlighting/delimiting raises attack cost and is worth doing — but weight it at zero in the safety argument; adaptive attackers defeat it.

## Reference artifact: