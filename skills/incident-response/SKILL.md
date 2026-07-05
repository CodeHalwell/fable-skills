---
name: incident-response
description: Load when production is broken or degraded — triaging an outage, deciding rollback vs debug, running or structuring an incident (roles, comms, severity), writing or reviewing a postmortem, designing on-call/alerting to reduce fatigue, or planning game days and chaos experiments.
---

# Incident Response

## The standard playbook, one line each

A strong model already knows this discipline cold; it's listed only so the corrections below have anchors. Mitigate first, diagnose in the postmortem — most mitigations (rollback, failover, kill-switch, degrade, shed load) need no root cause. First question is "what changed?" (deploys, flags, config, certs, crons, *adjacent teams*' changes), and you revert on correlation + reversibility, never on proof. Declare mechanically: symptom alert + plausible user impact → declare; solo-debugging >15 min without mitigation, or blast radius past one customer → declare. IC coordinates and never debugs (the player-coach collapse is the #1 structural failure); responders narrate what they try; comms go out on a fixed clock (30 min for high-sev) always naming the next update time. Severity = breadth × depth × trajectory — a climbing symptom classifies one level above its current impact — and never internal drama or embarrassment. Page only on symptoms (multiwindow error-budget burn: 14.4× fast / 6× slow), single-digit pages per week, every page actionable with a runbook. Postmortems: blameless, contributing factors not a single root cause, ≤5 action items with owner + ticket + date, at least one attacking detection/mitigation speed.

## Corrections to the trained instinct

Where the playbook a strong engineer (or model) produces cold is subtly wrong:

- **Parallel reverts are fine; the "one change at a time" rule is for novel changes only.** The reflex is to serialize everything: revert the flag, wait, then maybe roll back the deploy. Over-general. Two *reverts to known-good* (flag revert + deploy rollback) can go simultaneously — both are safe, and you do not need to know which was guilty before acting; attribution is postmortem work, and waiting between near-free reverts is user pain spent on curiosity. What must be serialized — one at a time, announced, with a stated expected effect — is *novel* changes: config edits, failovers, capacity moves. Blanket serialization slows mitigation; blanket parallelism destroys attribution. Split by revert-vs-novel.
- **The author's veto.** "It can't be my change, it's a one-liner" reliably delays the guilty revert. Authors systematically underrate blast radius and are anchored on having tested it. Timelines and reverts adjudicate, not authors — and the one-line "safe" change is over-represented in triggers precisely because it skipped scrutiny. If the flag revert fixes it, the deploy is exonerated by evidence instead of by its author.
- **Keep a specimen; don't just "capture forensics first."** The standard advice (heap dump, then restart) loses the live state. The stronger move: pull one sick instance out of the LB and leave it *running* as a specimen while restarting the rest. Mitigation and forensics are compatible if you think for ten seconds; a fleet-wide restart both masks the bug for 6 hours and deletes the evidence.
- **Recheck "what changed" twice.** People miss the flag flip on the first pass; the IC re-runs the change sweep with fresh eyes ~15 minutes in. Also: if the graphs say errors started at 14:02 and the suspect deploy landed 14:20, the deploy is innocent regardless of how suspicious it looks — but a deploy 22 minutes *before* onset stays a suspect (gradual rollouts and caches delay onsets).
- **Declaring victory on a masked symptom.** Error rate fell because the cache refilled, traffic dipped at lunch, or retries are absorbing it — not because your fix worked. Before standing down: tie recovery causally to your action (timing matches, mechanism explains it) and confirm on the *user-facing* metric. And if the revert fixed it but you don't know why (5% was fine, 50% wasn't — smells like a capacity cliff), the incident is mitigated but the risk is live: pin the change off, file the investigation *before* standing down; re-rolling without the answer schedules a rerun.
- **The "watch and see" non-decision.** "Give it 10 more minutes" repeated four times is a 40-minute decision to do nothing, never actually made. IC forces the framing: what specifically are we waiting to learn, and what do we do at each outcome? No answer → act now.
- **Handoffs have a physiology.** After ~2–4 hours of high-sev response, effectiveness craters and risk appetite goes weird. The IC schedules relief before that point; handoff means the incoming person reads the timeline and states the situation back — not "you're up, good luck."
- **Sub-severity blips get a tracking issue.** The weird self-recovering blip is a near-miss: same causal structure as a disaster, minus the harm and the politics. Three identical blips are an incident announcing itself in installments. If your incident count is low, near-misses are most of your training data — postmortem them at a discount.
- **Game days: purpose-rank before injecting chaos.** (1) Rehearse *response* — people, roles, rollback under a clock; (2) validate *recovery mechanisms* — does failover fail over, does the kill switch work, do backups restore; (3) discover unknown failure modes. Most teams buy lesson 3 before mastering 1–2; injecting novel chaos into a system whose *known* recovery paths are untested is paying for advanced lessons before the basics. Cheapest high-value version: re-run a recent real incident as a tabletop — it finds runbook rot, stale credentials, and "who can actually approve a region failover?" every single time, and every real incident should end with a "was the runbook right?" line item.
- **Nobody writes anything down during the incident** → the postmortem timeline gets reconstructed from memory three days later and the 14:07–14:20 wrong turn is invisible. The IC's running log (or a bot capturing the channel, with timestamps on graph screenshots *including time ranges*) is the raw data for every lesson; write the minute-by-minute timeline *before* anyone theorizes — it usually falsifies the story people remember.
- **Retry storms invert the capacity fix.** Load-shaped symptoms (queues growing, saturation) get "add capacity" — but capacity added into a retry storm feeds it. Before scaling anything, ask: what is actually saturated, and does added capacity increase load on the saturated thing (more web workers = more DB pressure)? Shed load / reduce retry aggression first.

## The first five minutes (take the first exit that applies)

1. Confirm real user impact (30 s): a cause alert (CPU, disk) with no symptom movement is not yet an incident.
2. Something changed in the window? → revert now, on suspicion. Both candidates revertible → revert both (see above).
3. Impact isolated to a region/AZ/shard? → fail over / drain; autopsy the sick node later.
4. A feature or dependency implicated? → kill-switch or degrade (stale cache beats down).
5. Load-shaped? → shed load + reduce retries, then capacity.
6. None → genuine diagnosis: explicit hypotheses stated with predictions *before* looking; bisecting questions first (one service or all? one region? reads or writes? one customer?); timebox each theory ~15 min and assign a second responder to a *different* hypothesis rather than piling on; re-run this tree every ~15 min.

## Severity ladder (pre-agree per product; the rightmost columns are the point)

| Sev | Definition (user terms) | What it buys/demands |
|---|---|---|
| 1 | Critical path down or data loss, many users | Page IC + responders + exec note; risky mitigations pre-authorized; 30-min comms + status page |
| 2 | Critical path degraded, or secondary feature down | Page on-call, IC assigned, change discipline suspended for reverts; 60-min comms |
| 3 | Minor, workaround exists, not worsening | Ticket, business hours, no page |

Trajectory rule: anything climbing classifies one level up. Upgrades announce immediately; downgrades wait for sustained evidence, not one good minute on a graph.

## Postmortem skeleton (forces the analysis that matters)

```markdown
# INC-2231: Elevated checkout failures — 2026-07-03
Impact: 8–12% of checkouts failed 14:02–14:16 UTC (~3,400 users, ~$41k delayed GMV)
Detection: alert at 14:07 — 5 min after onset; why not sooner? → CF-4
## Timeline (from channel log, not memory)
## Contributing factors (not "root cause")
CF-1 Trigger: new payment path saturates provider conn limit above ~20% traffic
CF-2 The 5% soak could not have caught a >20% capacity cliff (rollout design)
CF-3 Flag changes bypassed the canary analysis deploys get (control gap)
CF-4 Detection lagged: 5m alert window; 1m fast-burn window missing
## What went well / near misses
## Action items (owner, ticket, due) — each a system change, none "be more careful"
```

The quality gate: minute-level timeline from logs; ≥3 contributing factors with at least one detection/mitigation-speed factor; and the test — would these items have prevented *or materially shortened* this incident? If no item shortens MTTR, you analyzed the trigger and ignored the response.

## Verification / self-check

- IC loop every ~15 min: impact still occurring per which metric? Mitigation in flight, owner, expected effect by when? Next-best mitigation if it fails? Next stakeholder update? "What changed" rechecked?
- Before closing: symptom at baseline *and* you can say why (mechanism, timing); reverted-but-unexplained changes pinned off with a follow-up filed.
- Stopping rule: users healthy, regression risk pinned, specimen/forensics captured, follow-ups filed, timeline snapshotted. Root-causing beyond what mitigation requires is postmortem work — go to bed.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 10 baseline (cut/compressed — IC discipline, declaration thresholds, comms cadence, severity inputs, burn-rate alerting incl. exact 14.4×/dual-window constants, action-item hygiene, contributing-factors framing, alert-fatigue rules, 15-responder coordination), 2 partial (sharpened), 0 hard deltas among probed claims.
- Biggest baseline gaps: Opus over-serializes mitigation ("one change at a time" blanket rule) — missing the safe-reverts-in-parallel vs novel-changes-serial split; game-day guidance lacked the response>recovery>discovery purpose ranking; "capture forensics then restart" instead of keeping a live specimen out of the LB.
- This file is deliberately a correction sheet: the playbook itself is baseline knowledge; only the anti-instinct corrections and lookup artifacts (ladder, skeleton) are kept at full weight.
