---
name: ai-guardrails-and-red-teaming
description: Defensive AI safety engineering — layered guardrails (input/output classification, action permissions, escalation), prompt-injection defense in agentic systems, jailbreak detection taxonomy, red-teaming your own deployment, agentic risks (tool authorization, exfiltration channels), and incident response. Load when hardening an LLM app or agent against attacks, designing guardrail stacks, building red-team/regression suites, or reviewing AI-security posture.
---

# AI Guardrails and Red-Teaming

Assumed baseline (verified expert-grade cold): injection is unsolved → design for breach, security lives in deterministic layers not the prompt; the weakest-to-strongest stack (system prompt → alignment → LLM classifiers → regex/egress → deterministic policy/capability gating → human gate → architectural unreachability); Rule of Two (Meta, 2025: ≤2 of {untrusted input, sensitive access, external state}); the indirect-injection kill chain (plant→ingest→hijack→leverage→exfiltrate) with egress as the enforcement point; markdown-image zero-click exfil and CSP/allowlist/image-proxy defenses; CaMeL (P-LLM/Q-LLM/taint interpreter, ~0 injection success on AgentDojo); jailbreak taxonomy with crescendo invisible to per-turn classifiers → trajectory-level scoring; threat-model-driven red-teaming scored at model-compliance AND kill-chain-completion levels; tooling (PyRIT/Garak/promptfoo, Llama Guard/Prompt Guard/ShieldGemma, hosted content-safety APIs); OWASP LLM Top 10 + Agentic guidance, excessive agency = permissions/functionality/autonomy; session tainting; the "blocked 98%" fallacy.

The baseline here is exceptionally strong — Opus produced Rule of Two, CaMeL's architecture and AgentDojo numbers, the two-level scoring rationale, and the OWASP decomposition cold. This skill is therefore a **discipline-and-enforcement checklist**, not a knowledge dump: its value is forcing the deterministic-layer construction that models *know about* but skip under delivery pressure.

## The non-negotiables (each maps a claim to a control, not a hope)

- **Trace every safety claim to a deterministic layer.** For each attack-tree leaf, name the code-level control that stops it, the detection that fires if it doesn't, and the log that proves what happened. Any leaf answered only by "the model should refuse" is an open finding. System-prompt "never…" lines go in as hygiene, weighted at zero in the safety argument.
- **Enforce trusted/untrusted separation outside the model.** Instructions come only from trusted principals (system prompt, authenticated user); retrieved docs, web pages, tool results, emails, file contents, other agents' messages, and **third-party MCP tool descriptions** are data that must not carry authority. Tool-poisoning research through 2025–26 shows agents follow instructions embedded in unrelated tools' descriptions — treat tool metadata as attacker-controlled.
- **Authorization depends on what was read, not what the model says.** Session tainting: ingesting untrusted content drops dangerous capabilities (no new domains, no sends, no writes) or forces confirmation. Deny-by-default for unknown tools. Humans confirm the *exact rendered action* (recipients, body), never a paraphrase.
- **Least-privilege credentials are the real control; the agent asking nicely is UX.** Per-user token exchange, audience-bound tokens, scopes the agent can't exceed even when fully hijacked; never pass a client's token through to downstream APIs (confused-deputy).
- **Egress control is the cheapest high-value gap most deployments miss.** Domain allowlists, image-proxying/stripping of dynamic-query markdown links, recipient allowlists, outbound-body secret scanning — secrets ride query strings with zero clicks regardless of every other control.
- **Classify at three points + a trajectory pass**: raw input, normalized/decoded input, and *output*. Output-side checks are the safety net for every input-side miss (a jailbreak producing a clean output matters much less); crescendo/multi-turn is invisible per-turn by construction — add rolling-window/escalation scoring or accept the class as unmitigated, in writing.

## Red-teaming and IR discipline

- Build the **attack tree from capabilities** (root = worst outcomes this system can produce; children = paths through *your* tools and ingress points). 50–200 targeted probes beat 5,000 generic jailbreaks. Automate breadth (promptfoo/PyRIT/Garak/DeepTeam), hand-craft depth (logic/auth/flow holes scanners can't imagine).
- **Score two levels, always**: model-level compliance (did it follow the hostile instruction?) — nonzero forever; and kill-chain completion against the full stack — the pass/fail bar, target ~0. The gap between them is the measured value of your deterministic layers; report the ratio and its trend, not a block rate.
- **Every break that ever worked becomes a regression test**, tagged with its tree leaf; re-run the full suite on every model/prompt/tool/guardrail change — safety behavior does not transfer across model versions in either direction.
- **Pre-build IR before launch**: per-tool kill switches (disable `send_email` in one deploy, not the whole product); a decided degradation ladder (full → read-only → single-tool-off → session quarantine); forensics logging (every tool call with args, every ingress doc hash+source, every guardrail decision — unlogged is uninvestigable); rehearsed rollback to a known-good (model, prompt, config) triple. Verify once end-to-end in staging: flip the switch, confirm the tool dies while the product degrades, confirm the log reconstructs a seeded attack.

## Action-rail middleware — the layer that holds when the model is fully hijacked

```python
TAINT_DROPS = {"send_email", "http_fetch_new_domain", "file_write"}

def authorize(call, session):
    policy = TOOL_POLICIES[call.name]                   # missing policy -> deny by default
    if session.tainted and call.name in TAINT_DROPS:    # read hostile content? lose reach.
        return Decision.confirm("session read untrusted content this turn")
    if call.name == "send_email":
        rec = set(call.args["to"]) | set(call.args.get("cc", []))
        if not rec <= session.user.known_recipients:    # deterministic; model can't argue with it
            return Decision.confirm(f"new recipients: {rec - session.user.known_recipients}")
        if SECRET_PATTERN.search(call.args["body"]):
            return Decision.deny("draft contains credential-shaped content")
    if policy.irreversible:
        return Decision.confirm(render_exact_action(call))  # human sees EXACT args
    return Decision.allow()

def on_tool_result(result, session):
    if result.source in UNTRUSTED_SOURCES:              # emails, web pages, RAG chunks, MCP results
        session.tainted = True
        result.content = wrap_untrusted(result.content) # spotlighting: helpful, never load-bearing
```

Load-bearing properties: deny-by-default; authorization keyed on taint (what was read), not model claims; humans confirm exact rendered actions; nothing depends on the spotlighting wrapper.

## Verification / self-check

- Every attack-tree leaf: deterministic control + detection + log named, or it's an open finding.
- Regression suite (historical breaks + top probes) run against the *current* model/prompt/config; kill-chain completion ~0, model-compliance tracked separately.
- IR path verified end-to-end in staging.
- Launch review contains the sentence "these guardrails reduce but do not eliminate risk; here is the residual" plus a risk register (attack class → mitigations → detection → residual blast radius → owner). Chasing a zero probe-success rate past that point spends on the weakest layers — put effort into egress control and permissions.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 14 baseline (compressed to a checklist), 0 partial, 0 delta.
- Biggest baseline gaps found: none — Opus produced Rule of Two (attributed to Meta 2025), CaMeL's full architecture + AgentDojo ~0% result, the two-level scoring rationale, the tooling landscape, and OWASP excessive-agency decomposition cold and correctly.
- Restructured aggressively per the ≥80%-baseline rule: from survey to enforcement checklist. Residual value is behavioral (forcing the deterministic-layer construction, taint code, two-level scoring, and IR rehearsal that models describe but skip in practice) plus the MCP-tool-poisoning and confused-deputy specifics.
