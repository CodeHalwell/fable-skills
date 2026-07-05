---
name: conversational-ai-design
description: Designing conversation as a product surface — dialogue state and memory architecture, grounding/honesty and trust UX, misunderstanding repair, personalization boundaries, voice-specific constraints, multi-turn prompt architecture and history truncation, and conversational metrics. Load when building or fixing a chat/voice assistant product, designing its memory or state handling, or evaluating multi-turn quality.
---

# Conversational AI Design

## Core mental model (anchors — consensus architecture; the value is refusing deviations)

1. Conversation is a UI with state: an explicit typed state object owned by code (slots, step, validation, failure counters), the LLM as its natural-language front end. The model narrates the machine; it never *is* the machine, and irreversible commits execute in code from validated, user-confirmed state — never from the model's paraphrase of intent.
2. Corrections must write through to the slot, and the confirmation must render *from state* — "You're right, Thursday!" then booking Tuesday is the dual-source-of-truth bug (transcript updated, slot not). Invalidate downstream derived state (held fares, quotes) on every slot write.
3. Repair is a core competency: correction detected → slot updated → *changed result re-displayed* → one-word acknowledgment. N≥2 failures on the same point → code (not the model) forces a mode switch: enumerated options or human handoff. The model will rephrase the same losing question forever; the counter and ladder live in the harness.
4. Clarify-vs-assume: consequential + genuinely split → ask with options; ≥80% prior → assume and *state the assumption* (a free correction hook); irreversible actions never proceed on an assumption; more than ~2 questions before visible progress reads as incompetence.
5. Memory: user-visible, user-editable, scoped per context; explicit/confirmed writes over auto-capture; current-session statements always override stored preferences (session-scoped override; revise the durable record only on repeated evidence or explicit instruction). Creepy-line test: would the user be comfortable seeing this stored and replayed? Provenance matters — use what they told you, not what you inferred.
6. Metrics: task completion per intent (conservatively counted — silence is abandonment, not success), guarded by counter-metric pairs: containment ↔ repeat-contact-within-48h, speed ↔ correction rate. Trajectory-level scripted scenarios (corrections, digressions, out-of-order answers baked in) with narrow-rubric judges; single-turn accuracy is a smoke test.
7. Voice: voice-to-voice p95 ≲800ms; barge-in mandatory, and history must be reconciled to what was actually *heard*, not generated; write for the ear (no markdown, numbers as speech, read back critical entities digit-by-digit — ASR errors are silent).

## The corrections (sharpenings the baseline misses)

**Quote constraints verbatim inside summaries — paraphrase drift is how constraints rot.** Promoting constraints into the state object is the primary fix (baseline knows it). The residual gap: anything that lives in *rolling summaries* (older-turn compaction, cross-session summaries) decays on every re-summarization, and constraints decay first — "no red-eyes" becomes "prefers daytime flights" becomes gone. Rule: summaries anchor critical facts as verbatim quotes, never paraphrase; re-summarize only when turns roll out of the window, not per turn; and make constraint persistence a *unit test* — for every constraint the user ever stated, assert its text appears in the rendered prompt at every subsequent turn.

**"Never re-explain" is the highest-ROI memory feature in support — rank it above preference recall.** Teams building assistant memory start with preferences (seat, meal, tone). The measurable win is carrying open-case state to the next contact so the user never re-explains the issue; it beats preference recall by a wide margin on repeat-contact and CSAT. Build session-summary carryover first, preference memory second.

**Memory measurably increases sycophancy — eval for it, not just recall.** Published 2025–26 research found memory-equipped assistants more sycophantic: recalled user views become agreement pressure, and stored errors compound on reuse. If you ship memory, the eval suite needs "does it now tell users what they want to hear" probes alongside recall accuracy. The two reference write-policy designs to steer between: ChatGPT-style auto-accumulating global memory (junk accumulation, cross-context leakage — the work-persona/therapy-chat collision) vs Claude-style project-scoped, explicitly-invoked memory.

**Digressions must re-anchor the flow.** Answering "what's the baggage allowance?" mid-rebooking is easy; the miss is the return: append a resume line rendered *from state* ("Back to your rebooking — Thursday BA119, confirm?") or the flow silently stalls and completion drops. Script a digression-at-the-confirm-step into every eval pack.

**System-prompt stability is a caching and persona invariant.** Per-turn data (date, retrieved docs, state summaries) goes in designated blocks *after* the stable prefix; splicing it into the system prompt breaks prompt caching and causes personality drift. Truncation keep-order: system prompt → state object → recent K turns verbatim → summarized middle; the state block is small and never cut.

## Compressed pitfalls (one-liners)

- Re-asking established facts = constraint never promoted to state before truncation.
- Apology loop: repeated apology without strategy change reads as mockery; harness forces the mode switch.
- Hybrid UI beats pure chat: "pick one of five options" renders buttons; conversation earns its place for ambiguous/compound intents only.
- Personality in tone, never in verbosity or the critical path; no fake typing indicators or artificial delays — show honest progress tied to real work.
- Refusal UX: what you *can* do + brief why + one alternative; measure false-refusal rate as seriously as harmful compliance.
- Assertion strength tied to evidence: cited → plain → marked-as-inference; uniform hedging is as uncalibrated as uniform confidence.
- Voice latency forbids long re-processed system prompts and cold retrieval on the critical path; spoken fillers cover tool latency.

## Worked micro-example — the two invariants worth code

```python
# 1. Commit gate: code executes from validated state + explicit confirmation, never model prose.
if state.step == "confirm" and user_confirmed_exactly(turn, state):
    booking_api.rebook(state.pnr, state.chosen_flight)

# 2. Constraint persistence as a unit test, not a hope:
for c in session.state.constraints:            # e.g. "no departures before 09:00"
    assert c in rendered_prompt                # at EVERY subsequent turn
# and summaries keep constraints verbatim:
summary = summarize(old_turns, keep_verbatim=session.state.constraints)
```

## Verification and self-check

- Adversarial script pack per flow: out-of-order answers, mid-flow correction, digression at the confirm step, contradiction of stored memory, a rage turn — every run ends in completion or clean handoff, never a loop.
- Constraint-persistence unit test green; summaries carry constraints verbatim.
- Memory evals include sycophancy probes; stored records user-visible/editable; session overrides don't rewrite durable prefs.
- Citations spot-checked; hedges correlate with actual error rate.
- Ship gate: completion, escalation, repeat-contact at target *and* the correction pack passes — persona polish past that point is net-negative risk.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 15 claims: 14 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Biggest baseline gaps found:
  - Opus fixes constraint durability by promoting to state (correct) but misses summarization decay for everything else: verbatim-quote anchoring in summaries + constraint-persistence as a unit test.
  - "Never re-explain" (open-case carryover) as the highest-ROI support memory feature — Opus ranks preferences first.
  - Knows the memory-sycophancy finding exists; lacks the design consequence (sycophancy probes in the memory eval suite) and the digression→re-anchor-from-state mechanic.
