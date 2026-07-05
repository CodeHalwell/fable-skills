---
name: ml-production-systems
description: Load for ML-in-production questions — deploying/monitoring models, feature pipelines, training-serving skew, data or concept drift, model versioning and rollback, shadow/canary rollouts, retraining strategy, or diagnosing "the model was fine offline but degrades in production."
---

# ML Production Systems (MLOps)

## Core mental model (anchors — orthodox MLOps; the job is refusing exceptions to it)

- Training-serving skew is the #1 silent killer (dual implementations, time-travel/label leakage, cleaned-train vs raw-serve). The single highest-leverage decision: **log the features actually served and train the next model from those logs** — skew becomes structurally impossible for logged features.
- The model is ~5% of the system; debug data first, model second, infra third. Most "drift" alerts are pipeline bugs; most "model degradation" is a deploy.
- Offline metrics are a hypothesis: validate historically that offline deltas predicted online deltas before trusting them to rank candidates.
- Every deployed model shapes its own future training data (blocked transactions get no labels, unshown items get no clicks). Exploration slice + inverse-propensity weighting + a permanent 1–5% holdback are the standing countermeasures, and the holdback is also the only substrate for honest off-policy evaluation (IPS with clipped weights; report effective sample size `(Σw)²/Σw²`, not raw n).
- Rollout: shadow (catches skew/latency, cannot measure impact) → canary randomized by *entity, never request* → progressive ramp with the holdback retained. Rollback is a rehearsed config flip; no "latest" tags anywhere in serving.
- Monitor four layers (system, data, outputs, outcomes), sliced by segment; **page on output-distribution shifts and freshness breaches, ticket on per-feature drift**; a spike of predictions at a default/fallback value = a feature feed is failing and being silently imputed.
- PSI bands: <0.1 stable; 0.1–0.25 investigate; ≥0.25 act. (Older internal guides saying "investigate at 0.2" are miscalibrated against the convention.)

## The corrections (what standard MLOps doctrine underweights)

**Quantify the offline compass before using it.** Everyone agrees offline-online correlation "should be checked"; the operational rule is sharper: collect (offline delta, online delta) pairs for every past launch, and if the *sign* agrees less than ~80% of the time, your offline eval cannot rank candidates — improving the eval is then the team's highest-ROI project, ahead of any model work. And never conclude "offline gains are noise" from a flat online result without computing the canary's minimum detectable effect; small canaries are underpowered for the effect sizes that matter.

**Challenger comparisons are confounded by incumbency.** The incumbent has warm caches, per-user state, and — the invisible one — downstream systems *tuned to its score distribution*: thresholds, bid multipliers, review queues calibrated for its outputs. A better challenger loses the A/B because its scores hit thresholds set for someone else. Recalibrate scores to a common scale (or re-derive thresholds per model) before comparing, and give shadow mode time to warm caches. This is a top cause of "the new model is better offline and online-neutral."

**Model-feeds-model is a self-inflicted drift incident waiting to fire.** When one model's outputs are another's features, version that interface like an API contract and have the consumer pin the producer's model version. An upstream "improvement" deployed unilaterally shifts the downstream input distribution with no schema change, no alert, and a slow unexplained decay. Almost no team treats upstream-model versions as a consumed dependency; the ones that do skip a whole incident class.

**A retrain is a risk purchase, not hygiene.** Cost = compute + eval + review + *rollout risk* (every new model is a fresh chance to ship a bug); a retrain that recovers less metric than its rollout-risk cost is negative-value. Measure the decay rate (backtest a model trained at T on data from T+1w/1m/3m) and let that number pick the cadence; prefer triggered retraining when decay is irregular; keep the eval-vs-incumbent gate even on automated schedules. Under incident pressure, resist the hotfix retrain on incident-window data — it encodes the incident into the model.

## Compressed checklist (one-line reminders)

- Point-in-time-correct joins for every historical feature (feature stores exist mostly for this); parity-replay N thousand requests through both paths when dual implementations are unavoidable.
- Batch precompute + KV lookup whenever freshness allows — teams over-build online serving; per-feature freshness tiers (most features don't need fresher than daily).
- Distinguish "legitimately 0" from "missing-imputed" in encodings; alert on imputation rate per feature.
- Label lag bridged by leading indicators: prediction-distribution shift, calibration on early labels, champion-challenger disagreement rate, vintage curves.
- Lineage: prediction → model ID → code SHA → immutable data snapshot → feature versions → eval report, all resolvable; feature definitions are versioned, owned code.
- Canary guardrail metrics pre-registered with auto-rollback thresholds; time-based before/after comparisons are confounded — only concurrent randomized comparison counts.
- Model-blind labeling stream for eval and a fraction of training; audit every training-set join against the feedback-loop map (a "transactions" table that excludes blocked ones is post-decision data).
- Incident order: deploys/config → feature freshness & imputation dashboards → traffic mix shift (slice first) → only then genuine drift, confirmed by scoring the current model on the freshest labeled window.
- Schema contracts on consumed feeds; alert on unseen enum values and unit-scale shifts (a 100× mean shift is upstream units, not user behavior); all feature timestamps UTC, tested across a DST boundary.
- Every production model has a named owner, a runbook, and a deprecation plan.

## Worked micro-example — "offline 0.91 AUC, online acts like 0.78"

Order of elimination: (1) artifact/config parity — hash the deployed model, verify preprocessing version (embarrassingly often the answer); (2) feature parity replay of 1,000 logged requests through the training pipeline — any feature mismatching >0.1% is a suspect; (3) time-travel audit of the top-10 features by attribution (`user_lifetime_purchases` computed "as of today" inflates offline AUC exactly this way); (4) population filters vs live traffic (bots, empty-feature new users); (5) only then drift — and check whether offline eval on the *freshest* labeled window also reads 0.78 (world changed → retrain) or still 0.91 (unfound skew → back to step 2). Also recompute the online number with the offline scoring code first — metric-definition mismatch (pooled vs per-segment AUC, prevalence effects) resolves some of these without touching anything.

## Verification checklist before signing off

- [ ] Served-feature logging (or parity-test regime) in place; training reads served logs where possible; point-in-time correctness asserted.
- [ ] Output-distribution + freshness monitoring sliced by segment; imputation-rate alerts.
- [ ] Entity-randomized canary with pre-registered guardrails; holdback retained; thresholds recalibrated per model version.
- [ ] Feedback-loop map written; exploration slice feeding retraining; OPE reports effective sample size.
- [ ] Cadence justified by measured decay and gated vs incumbent; offline-online sign-agreement history exists.
- [ ] Upstream model versions pinned as dependencies wherever model outputs feed models.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 15 claims: 14 baseline (cut/compressed), 1 partial (sharpened — PSI bands; the skill's own 0.2 threshold corrected to the 0.1/0.25 convention per Opus), 0 delta.
- Biggest baseline gaps found:
  - Warm-start/incumbency confounds (downstream thresholds tuned to the old model's score distribution) absent from Opus's rollout reasoning.
  - Offline-online validation stated qualitatively; lacks the ~80% sign-agreement decision rule and the MDE check before dismissing offline gains.
  - Model-feeds-model interface versioning and retrain-value-vs-rollout-risk economics not surfaced.
