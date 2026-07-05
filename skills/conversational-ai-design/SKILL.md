---
name: conversational-ai-design
description: Designing conversation as a product surface — dialogue state and memory architecture, grounding/honesty and trust UX, misunderstanding repair, personalization boundaries, voice-specific constraints, multi-turn prompt architecture and history truncation, and conversational metrics. Load when building or fixing a chat/voice assistant product, designing its memory or state handling, or evaluating multi-turn quality.
---

# Conversational AI Design

## Core mental model

1. **Conversation is a UI with state, not a text generator with history.** Every turn, decide explicitly: what must the system *remember* (slots the user filled, decisions made, task progress), what can it *re-derive* (from tools, retrieval, or the visible transcript), and what must it *never* silently forget (a stated constraint like "no flights before 9am"). Products fail when this is left implicit — the model "remembers" by attention over a lossy window, and attention is not a datastore.
2. **Hybrid state is the winning pattern.** Pure free-form chat can't guarantee a transactional flow completes correctly (booking, refund, KYC); a rigid state machine can't handle humans, who answer questions out of order and change their minds. The pattern that works: an **explicit state object** (typed slots, current step, validation status) owned by code, with the LLM as the *natural-language interface* to it — filling slots from free-form input, explaining state, handling digressions — and code deciding transitions and enforcing invariants. The model narrates the machine; it does not *be* the machine.
3. **Trust is a budget you spend and refill.** Every confident-but-wrong answer spends trust; every calibrated "I'm not sure, but…", visible citation, and graceful refusal refills it. Users don't average quality — one confident fabrication in a domain they know poisons everything else. Design the honesty surface deliberately: when to hedge, when to cite, when to refuse-with-alternative, when to escalate to a human. A refusal with a path forward costs little; a stonewall or a fabrication costs the account.
4. **Repair is a core competency, not an error path.** Human conversation runs on repair — "no, I meant X" happens constantly. The assistant must (a) detect correction vs new topic, (b) update state *and act on the update* rather than merely acknowledging it, (c) not over-apologize. The apology loop (apologize → repeat the same mistake → apologize again) reads as mockery; the correction handler that says "Got it — Thursday, not Tuesday" and *shows the corrected result* is the trust-refill moment.
5. **Latency, memory, and personality are product decisions with cost curves.** Voice needs sub-second responses, which constrains model size and retrieval. Persistent memory delights until it's creepy or wrong. Personality delights until it obstructs the task. Default: function first, then warmth; never the inverse.

## Decision frameworks

### Dialogue state: what to store explicitly vs leave in-context

Ask per item: (1) Does a wrong value have real cost (payment, booking, identity)? → typed slot in code, validated, confirmed with the user before commit. (2) Is it needed after the context window rolls or the session ends? → persisted state/memory. (3) Is it cheaply re-derivable (today's date, account status)? → re-derive; stored copies go stale. (4) Is it conversational color (user seemed frustrated)? → leave it in-context; don't build schema for vibes. The failure of storing too much: state drifts from reality and the model trusts the stale store over the user's latest message. The failure of storing too little: turn 14 re-asks what turn 2 established — the single most rage-inducing chatbot behavior.

### Memory architecture (state of practice, mid-2026)

Three layers, choose deliberately:
- **Session memory** = the context window + your truncation strategy. Free but volatile.
- **Cross-session summary memory**: rolling summaries of past sessions. Beware **summarization decay** — each re-summarization loses nuance and can entrench early errors; anchor summaries to verbatim quotes for critical facts, and re-derive from source when stakes are high.
- **Persistent fact/preference memory**: discrete, *user-visible and user-editable* records. Industry converged here through 2025–26 (ChatGPT's auto-accumulating global memory vs Claude's project-scoped, explicitly-invoked memory are the two reference designs). Design judgments that matter: scoping (project/topic-scoped memory prevents cross-context leakage — the work-persona/therapy-chat collision), write policy (auto-captured memories accumulate junk and surprise users; explicit or confirmed writes are safer), and *visibility* (a memory the user can't view, edit, and delete is a liability — and in regulated contexts a compliance problem).
- Evidence note (as of 2026): published research found memory-equipped assistants measurably **more sycophantic** — recalled user views become agreement pressure. If you ship memory, eval for "does it now tell users what they want to hear," not just recall accuracy.

### Grounding and honesty surface

- Tie assertion strength to evidence: retrieved-with-source → cite it inline (tap/click to verify); model-knowledge → plain statement; inference/guess → marked as such ("Based on your last order, probably…").
- Refusal UX: state what you *can* do, why this is out of bounds (briefly, no lecture), and one concrete alternative or escalation. Measure false-refusal rate as seriously as harmful-compliance rate — over-refusal quietly kills retention.
- Never fake certainty to seem competent, and never hedge everything to seem safe — uniform hedging is as uncalibrated as uniform confidence.

### Clarify vs assume — the question-budget rule

Every clarifying question costs a turn of user patience; every wrong assumption costs more. The expert's calculus per ambiguity:

1. **Is the ambiguity consequential?** If all readings lead to roughly the same action, pick the most probable reading and *state it* ("Assuming your upcoming trip on the 14th…") — the statement is a free correction hook.
2. **Is one reading dominant?** ≥80% prior on one interpretation → assume + state. Genuinely split → ask, but ask with *options* ("the March invoice or the April one?"), never open-endedly re-asking what they just said.
3. **Is the action reversible?** Irreversible actions never proceed on an assumption, no matter how probable — that's what the confirm step is for.
4. Budget: more than two clarifying questions before any visible progress reads as incompetence; front-load progress ("I've found your booking — one question before I change it…").

### The metrics problem — measuring conversation without lying to yourself

- North star: **task completion** per intent, defined by observable outcome (rebooked, refunded, answered-with-source), counted conservatively — "user stopped replying" is abandonment until proven otherwise, not success.
- Guard the north star with counter-metrics in pairs, because each headline metric is gameable alone: containment ↔ repeat-contact-within-48h and post-escalation CSAT (containment that bounces back isn't containment); resolution speed ↔ correction rate (fast and wrong loses); engagement/turns ↔ nothing — turn count is not a goal in either direction.
- Diagnostic layer (find *why* the north star moved): correction rate per flow (NLU quality), constraint-violation rate (state handling), false-refusal and harmful-compliance rates (honesty surface), truncation-forgetting incidents (turn where a re-ask of established fact occurred).
- As of 2026 the tooling standard is trajectory-level evaluation: scripted multi-turn scenarios (with corrections, digressions, out-of-order answers baked in) scored on completeness and knowledge retention, plus LLM-judges with *narrow rubrics* ("did the assistant confirm before committing?") — never a global 1–10 "quality" judge, whose scores drift and correlate with verbosity.
- Retention is the only vibe-proof long-run metric for open-ended assistants: cohort return rate says what users actually decided about you. Vibes demos and cherry-picked transcripts predict nothing.

### Voice-specific constraints

- Latency budget: voice-to-voice p95 under ~800ms; past ~1s of silence users assume failure. This forbids long system prompts re-processed per turn, cold retrieval on the critical path, and non-streaming stages. Use spoken fillers to cover unavoidable tool latency.
- **Barge-in** (user interrupts TTS) is mandatory: kill playback instantly, and reconcile history to what was actually *heard*, not what was generated — otherwise the model believes it said things the user never heard.
- Write for the ear: numbers, dates, IDs, and lists that render fine as text are unintelligible as speech ("$1,234.56" → "twelve hundred thirty-four dollars"); no markdown, no long enumerations; confirm alphanumerics digit-by-digit. Prosody-aware formatting (short sentences, commas where pauses belong) is prompt work, not TTS config.

### Multi-turn prompt architecture

- **System prompt stability**: keep it constant across turns (cache-friendly, behavior-stable). Per-turn dynamic content (retrieved docs, state summaries) goes in designated blocks *after* the stable prefix, not spliced into the system prompt — churn there defeats prompt caching and causes personality drift.
- **Truncation that preserves coherence**, in order of what to keep: (1) system prompt, (2) explicit state object/summary, (3) most recent K turns verbatim, (4) summarized middle. Never silently drop the user's stated constraints — promote them into the state object *before* their turns roll out of the window. Drop-oldest-first without a state object is how "no flights before 9am" gets forgotten at turn 30.

## How an expert thinks through it

*Scenario: airline support assistant — rebooking, baggage, refunds; chat now, voice later.*

Start with the flows, not the prompt. Rebooking is transactional: it has legally-meaningful commits and payment. So: state machine in code — `identify_booking → select_new_flight → confirm_fare_difference → commit` — with typed slots (PNR, flight choice, fare delta acknowledged). The LLM fronts it: parses "actually can we do Thursday instead" into a slot update, explains fare rules, handles "wait, what's the baggage allowance on that one?" as a digression *without losing the flow's place*. (Rejected: pure prompt-driven flow — "ALWAYS confirm before rebooking" in a system prompt is a behavior lottery, and a compliance team won't accept a lottery. Rejected: rigid IVR-style form-filling — users answer out of order; forcing sequence multiplies turns and abandonment.)

The commit step: code, not the model, executes the rebook, and only from a validated state object the user explicitly confirmed ("Confirm: BA117 → BA119 Thursday, +£42 — yes?"). The model can never trigger an irreversible action from its own paraphrase of intent.

Memory: store loyalty tier, seat/meal preferences — visible and editable in settings. Do NOT auto-store inferred life facts ("user mentioned a funeral") — creepy on replay, wrong half the time; the line I use: store what the user *would expect an agent to note on their account*; leave the rest in-session. Session summary carries open-case state to the next contact so the user never re-explains — "never re-explain" is the highest-ROI memory feature in support, far ahead of preference recall.

Repair design: corrections are the top intent after task requests. Detection: short user turn contradicting a just-filled slot → treat as correction, update, *re-display the changed result*, one-word acknowledgment, no apology paragraph. Two failed understanding attempts on the same point → don't try a third rephrase; switch mode: present enumerated options or offer a human. (Rejected: unlimited retry loops — the third "I'm sorry, I didn't quite get that" is where users start typing "AGENT. AGENT. AGENT.")

Metrics before launch: define task-completion per flow (rebooking completed without human = containment, *but* count "user gave up" separately from "resolved" — containment inflated by abandonment is the classic vanity metric); escalation rate; correction rate per flow (proxy for NLU quality); repeat-contact-within-48h (proxy for false resolution); CSAT sliced by flow. (Rejected as primary: average conversation length — ambiguous in both directions; and "sounded helpful" LLM-judge vibes — use LLM judges for *specific* rubrics like "did it confirm before committing," not global quality.)

Voice later: this architecture ports because state lives in code — I swap the surface, rewrite output templates for the ear, add barge-in and filler strategies. A prompt-only design would need a rebuild.

## Failure modes and pitfalls

- **Re-asking established facts.** Cause: truncation dropped the turn, and no state object existed. Fix: promote constraints/slots into explicit state at the moment they're stated; state survives truncation by construction.
- **Acknowledge-but-ignore corrections.** Model says "You're right, Thursday!" then books Tuesday, because the correction updated the *transcript* but not the *slot*, and downstream code read the slot. Any correction must write through to state, and the confirmation must render *from state*, never from the model's prose.
- **The apology loop.** Repeated failure + repeated apology with no strategy change. Detect N≥2 failures on the same point in code and *force* a mode switch (options list, human handoff). The model will not break the loop itself; the harness must.
- **Stale-memory poisoning.** Persistent memory says "prefers window seat"; user asked aisle this trip; assistant "helpfully" fights them. Rule: current-session statements always override stored memory, and the override should update or flag the stored record.
- **Cross-context leakage.** Global memory surfaces a sensitive fact from another context ("like you mentioned about your divorce…") inside a work task. Scope memory by project/context; make retrieval respect scope; give users per-scope wipes.
- **Personalization past the creepy line.** Test: would the user be comfortable seeing this *stored and replayed back*? Preferences they stated: yes. Inferences about health/relationships/finances they didn't ask you to keep: no. Delight comes from remembering what they *told* you, not from demonstrating surveillance.
- **Chatbot-for-everything.** Forcing a conversation where a form, button, or table is the right UI. If the interaction is "pick one of five known options," render the options. Conversation earns its place for ambiguous, exploratory, or compound intents — hybrid UIs (chat that emits buttons/cards/forms at decision points) beat pure text in completion rate.
- **Personality over function.** Quirky persona text padding every response adds latency, tokens, and grating repetition at scale. Persona belongs in tone choices, not in extra sentences. Also: fake typing indicators and artificial delays to seem "human" — users uniformly hate discovering the fakery; show honest progress ("checking your booking…") tied to real work instead.
- **System-prompt churn.** Injecting per-turn data into the system prompt (date, user mood, last tool result) breaks prompt caching (cost/latency) and destabilizes persona. Stable prefix; dynamic content in clearly-delimited later blocks.
- **Single-turn evals for a multi-turn product.** Context drift, constraint forgetting, repair failures, and sycophantic agreement only appear across turns. Eval on full scripted conversations (including a correction, a digression, and an out-of-order answer in every scripted flow) plus replayed production transcripts; as of 2026 this multi-turn/trajectory evaluation is standard practice in conversational eval tooling — single-turn accuracy is a smoke test, not an eval.

## Worked micro-example

Hybrid state pattern (TypeScript sketch — code owns transitions, model fronts them):

```typescript
type RebookState = {
  step: "identify" | "select" | "confirm" | "committed";
  pnr?: string;
  chosenFlight?: string;
  fareDeltaGBP?: number;
  fareAcknowledged: boolean;          // must be true before commit — enforced HERE
  constraints: string[];              // e.g. "no departures before 09:00" — survives truncation
};

// Every turn: model extracts updates as a tool call, code validates and advances.
const update = await llm.extract(userTurn, RebookUpdateSchema); // slots, corrections, digression?
applyValidated(state, update);        // rejects impossible values; corrections overwrite slots

if (update.digression) {
  reply = await llm.answer(update.digression, { state });       // answer it...
  reply += renderResumeLine(state);   // ...then re-anchor: "Back to your rebooking — Thursday BA119?"
}

if (state.step === "confirm" && userConfirmedExactly(userTurn, state)) {
  await bookingApi.rebook(state.pnr!, state.chosenFlight!);     // code commits from STATE,
  state.step = "committed";                                     // never from model prose
}
```

The load-bearing choices: constraints live in a struct that never gets truncated; the irreversible call is gated on validated state plus explicit confirmation; digressions are answered *and then the flow re-anchors itself*.

**Per-turn prompt assembly with coherence-preserving truncation:**

```python
def build_messages(session, budget_tokens: int) -> list[Message]:
    fixed = [
        system_prompt(),                      # byte-identical every turn -> prompt cache hit
        state_block(session.state),           # slots, constraints, flow position — small, never cut
    ]
    recent = last_k_turns(session, k=12)      # verbatim tail — repair needs exact recent wording
    remaining = budget_tokens - count(fixed) - count(recent)

    middle = []
    if session.older_turns:
        if session.middle_summary_stale:      # re-summarize only when turns roll out, not per turn
            session.middle_summary = summarize(
                session.older_turns,
                keep_verbatim=session.state.constraints,   # constraints quoted, not paraphrased —
            )                                              # paraphrase drift is how "no red-eyes"
        middle = [session.middle_summary]                  # becomes "prefers daytime flights"

    return fixed + middle + fit(recent, remaining)

# invariant, checked in tests: for every constraint the user ever stated,
# assert constraint_text in rendered_prompt — at EVERY subsequent turn.
```

The testable invariant at the bottom is the point: constraint persistence should be a unit test, not a hope. The other detail that pays rent is quoting constraints verbatim inside summaries — summarization decay hits paraphrased constraints first.

## Verification and self-check

- Before shipping a flow, run the adversarial-user script pack: answers out of order, mid-flow correction, digression at the confirm step, contradiction of stored memory, and a rage turn — every pack member must end in completion or clean handoff, never a loop.
- Check the trust surface: sample 20 answers, verify each citation actually supports the sentence it decorates, and confirm hedges correlate with actual error rate (calibration, not vibes).
- Replay: read 10 production transcripts end-to-end weekly. Metrics find regressions; transcripts find the *reasons*. Look specifically for silent constraint drops and acknowledge-but-ignore.
- Stopping rule: a flow is done when task completion, escalation, and repeat-contact rates are at target *and* the correction-handling pack passes. Persona polish, cleverness, and more memory features past that point are net-negative risk until the fundamentals hold under adversarial replay.
