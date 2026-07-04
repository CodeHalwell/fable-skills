---
name: ml-production-systems
description: Load for ML-in-production questions — deploying/monitoring models, feature pipelines, training-serving skew, data or concept drift, model versioning and rollback, shadow/canary rollouts, retraining strategy, or diagnosing "the model was fine offline but degrades in production."
---

# ML Production Systems (MLOps)

## Core mental model

- **Training-serving skew is the number-one silent killer.** The most common production ML failure is not drift, not stale models — it's the serving path computing features differently than the training path did. It produces no errors, no alerts, just a model that is quietly 5–30% worse than its offline metrics from day one. Assume skew exists until you have measured that it doesn't.
- **The model is ~5% of the system; the data pipelines are the product.** Almost every production incident traces to data: a schema change upstream, a null-handling difference, a joined table that went stale, a timezone. Debug data first, model second, infrastructure third.
- **Offline metrics are a hypothesis, not a result.** An offline AUC gain means nothing until you've verified that offline improvements correlate with the online metric you actually care about — many teams discover, after a year of "improvements," that their offline metric stopped correlating with revenue/engagement long ago.
- **Every deployed model changes the world it observes.** A recommender only gets feedback on items it showed; a fraud model only gets labels on transactions it let through. Naive retraining on this feedback data poisons the next model. Design the feedback loop before designing the retraining job.
- **You will roll back.** Version everything needed to reproduce and to revert — model artifact, training data snapshot, feature definitions, code — and rehearse the rollback path before you need it.

## Decision framework: serving architecture

| Situation | Choose | Reasoning |
|---|---|---|
| Predictions consumed on a schedule (daily emails, nightly risk scores), entity set enumerable | Batch precompute → key-value store lookup at request time | Simplest reliable architecture; skew surface shrinks (one offline pipeline); latency is a lookup. Choose this whenever freshness allows — teams over-build online serving for tasks batch would serve better. |
| Prediction depends on request-time context (search query, current session, fresh transaction) | Online inference with a feature store for precomputed features + request-time features | Only the request-time features need an online path; precompute everything precomputable. |
| Both fresh signals and heavy features | Hybrid: batch features (daily aggregates) + streaming features (last-hour counters) + request features, joined at serve time | Each feature gets the cheapest freshness tier it actually needs — audit "does this feature need to be fresher than daily?" per feature; most don't. |
| Strict latency budget (<10ms) with a big model | Distill/quantize the model, or precompute; don't try to make the big model fast with caching heroics | Cache hit rates on high-cardinality inputs disappoint; distillation is the honest fix. |
| Model outputs feed another model | Version the interface like an API contract; consumer pins producer's model version | Upgrading the upstream model silently shifts the downstream model's input distribution — a self-inflicted drift incident. |

## Training-serving skew: causes and defenses

Three canonical causes:
1. **Dual implementations** — features computed in SQL/Spark for training, reimplemented in Java/Python for serving. Any divergence (null defaults, rounding, string normalization, timezone, category encoding of unseen values) is skew.
2. **Time travel / label leakage** — training features computed with data that wasn't available at prediction time (using the *current* value of a slowly-changing field for a historical example). Symptom: offline metrics too good to be true, online collapse. Defense: point-in-time-correct joins (feature stores exist mostly to do this correctly).
3. **Distribution mismatch** — training on cleaned/filtered data while serving raw traffic (deduped training set vs. duplicate-heavy traffic; bots filtered offline but present online).

Defenses, in order of effectiveness:
- **Log the features actually served** (with the prediction) and train the next model *from those logs*. This makes skew structurally impossible for logged features and gives you the ideal debugging dataset. This is the single highest-leverage architectural decision in production ML.
- One feature codepath: shared feature-transform library or feature store consumed by both training and serving. If dual implementations are unavoidable, write **parity tests**: replay N thousand production requests through both paths and assert feature-level equality (exact for categoricals, tolerance for floats).
- Continuous skew monitor: sample live traffic, compute features offline-style, diff against served values, alert on divergence rate.

## Drift: data drift vs concept drift — different problems, different responses

| | What changed | Detect via | Correct response |
|---|---|---|---|
| **Data drift** (covariate shift) | Input distribution P(X): new user segment, upstream format change, seasonality | Per-feature distribution monitoring: PSI (>0.2 = investigate), KS test, null-rate and cardinality tracking on inputs; prediction-distribution shift | Often *retraining is wrong first move* — first check if it's a pipeline bug (a broken upstream feed looks exactly like drift). If real, retrain on recent data. |
| **Concept drift** | The relationship P(Y\|X): fraud tactics adapt, user tastes shift, a competitor changes the market | Rising error against *ground-truth labels* as they arrive; proxy: calibration decay (predicted probabilities vs realized rates) | Retrain on recent labeled data; if drift is abrupt (policy change, new fraud scheme), a model trained on mostly-old data still fails — reweight or window the training data. |

Key operational facts:
- Label lag governs what you can detect: if true labels arrive 30 days late (chargebacks, churn), concept drift is invisible for 30 days via accuracy. Bridge the gap with leading indicators: prediction distribution shift, calibration on early-arriving labels, business-metric proxies.
- **Prediction-distribution monitoring beats input monitoring for alert quality**: the model integrates all inputs, so a shifted output histogram (mean score, quantiles, entropy) is a high-signal, low-cardinality alarm. Monitor both, page on outputs.
- Most "drift" alerts are data-quality incidents. Route drift alerts to a pipeline-debugging runbook first, a retraining decision second.

## Versioning and lineage

Minimum viable lineage — for any prediction in production you must be able to answer: which model version, trained by which code commit, on which data snapshot, with which feature definitions, and which hyperparameters?

- Model registry entry = artifact + git SHA of training code + immutable pointer to training data (snapshot/partition IDs or data-version hash, not "the users table") + feature-set version + eval report.
- Feature definitions are code: versioned, reviewed, with owners. A feature "improved" upstream without a version bump silently changes every downstream model.
- Data versioning: for warehouse-scale data, immutable dated partitions + a manifest beat heavyweight data-versioning tools; the requirement is *reproducibility of the exact training set*, not any particular tool.
- Never "latest" tags in serving config. Deployments pin an exact model version; promotion is an explicit config change you can diff and revert.

## Rollout: shadow → canary → progressive

1. **Shadow deployment**: new model receives live traffic, predictions are logged, *nothing acts on them*. Validates: latency under real load, feature availability at serving time, prediction distribution sanity vs incumbent, crash/timeout rate. Catches skew and engineering bugs; **cannot** measure real impact (no user response) — don't skip the canary because shadow "looked fine."
2. **Canary**: 1–5% of traffic gets the new model's decisions. Watch guardrail metrics (latency, error rate, business floor metrics) with automatic rollback triggers. Randomize by *user/entity*, not by request — per-request randomization gives one user inconsistent decisions and contaminates measurement.
3. **Progressive rollout with interleaved holdback**: ramp 5→25→50→100%, but keep a small (1–5%) long-term holdback on the old model (or a random-policy slice where affordable) — it's your only unconfounded baseline for measuring the new model's true effect and for generating unbiased training data.
- Time-based comparisons ("metrics this week vs last week") are confounded by seasonality and mix shift; only concurrent randomized comparison counts.
- Rollback must be a config flip taking effect in minutes, exercised regularly (game-day it), not a redeploy taking hours.

## Feedback loops that poison training data

Every model whose decisions affect what data gets collected creates selection bias in its own future training data:

- **Fraud/moderation**: blocked transactions never get labels. Retraining on "approved + outcome" data teaches the model only about the region it already approves; blind spots become permanent and fraud migrates into them. Fix: approve a small random exploration slice (or use approve-with-review), and weight training data by inverse propensity.
- **Recommenders**: clicks only exist for shown items → position bias + popularity feedback ("rich get richer"). Fix: log propensities (probability each item was shown) and train with inverse-propensity weighting; inject exploration (epsilon or Thompson-style) deliberately.
- **Self-influence**: a pricing/ranking model changes user behavior, then trains on that changed behavior, amplifying its own quirks over successive retrains. Symptom: metrics drift over retrain generations with no external cause. Fix: the long-term holdback slice provides untreated data; compare each new model against it.
- **Label feedback**: using the model's own predictions as labels (or human labelers anchored on model suggestions) causes confidence collapse over generations. Keep a model-blind labeling stream for eval and for a fraction of training data.

Rule: before wiring any automated retraining, write down *how today's model influences tomorrow's training data*. If the answer is "it doesn't," check again — it almost always does.

## Monitoring model outputs (distributions, not uptime)

A model service can be 200-OK, low-latency, and completely broken. Monitor in four layers:
1. **System**: latency, throughput, error rate (necessary, insufficient).
2. **Data**: input feature nulls, out-of-range values, unseen categories, schema changes, feed staleness (feature freshness lag is a top pager).
3. **Model outputs**: prediction histogram vs a reference window (PSI/KS on scores), mean/quantiles, entropy for classifiers, share of predictions at clipping boundaries or exactly at a default value (a spike at the fallback value means a feature is failing and being default-imputed — a classic silent failure).
4. **Outcomes**: realized accuracy/calibration as labels arrive; business metric per model version.
- Slice all of it by segment (country, platform, cohort): aggregate output distributions can look stable while one segment is fully broken.
- Alert design: page on output-distribution shifts and freshness breaches; ticket (don't page) on individual-feature drift — feature-level alerts are too noisy to page on and cause alarm fatigue.

## Retraining cadence economics

- Cadence is a cost-benefit decision, not hygiene. Measure **decay rate**: backtest a model trained at time T on data from T+1w, T+1m, T+3m. If performance decays 0.1% per month, quarterly retraining is fine; if 2% per week (ads, fraud, news), you need weekly-or-faster and the pipeline automation to match.
- Cost of retraining = compute + eval + human review + rollout risk (every new model is a fresh chance to ship a bug). A retrain that recovers less metric than its rollout risk costs is negative-value.
- Prefer **triggered** retraining (drift threshold breached, decay measured, N new labels accumulated) over calendar cadence when decay is irregular; prefer calendar cadence when decay is steady and automation is solid (predictability is worth something).
- Scheduled retrains still need gates: auto-retrain + auto-deploy without an eval gate ships a bad model the first time upstream data breaks. Gate = offline eval vs incumbent on a fixed golden set + canary.

## Offline-online correlation: validate the compass

Before trusting offline metrics to drive decisions:
- Collect history: for each past model change, record (offline delta on your metric, online delta from its A/B test). Plot them. If the sign agrees < ~80% of the time, your offline eval cannot rank candidates and improving it is the highest-ROI project on the team.
- Common causes of decorrelation: offline eval set from a different distribution than current traffic (stale, or filtered differently), label leakage inflating offline numbers, offline metric mismatch (AUC vs. top-k behavior actually shown to users), feedback-loop-biased eval data.
- Never conclude from an online-flat result that offline gains are "noise" without checking power: small canaries are underpowered for small effects; compute the minimum detectable effect before deciding.

## Incident response for model degradation

When the metric dips, check in this order (cheapest and most probable first):
1. Did anything deploy? Model version, serving code, feature pipeline code, upstream schema — correlate the dip's start time against every changelog you can find. Most "model degradation" is a deploy.
2. Is a feature feed stale or failing? Check freshness lag and imputation-rate dashboards per feature.
3. Did the input mix shift? Traffic composition (new campaign, new market, bot wave) changes aggregate metrics without any model change — slice the metric before concluding anything.
4. Only then consider genuine drift, and confirm it by scoring the current model on the freshest labeled window.
Mitigation hierarchy: rollback (if a deploy correlates) → failover to the previous model version → degrade gracefully (rule-based fallback) → hotfix retrain (last resort under time pressure; hasty retrains on incident-window data encode the incident).

## Failure modes & pitfalls

- **Silent default-imputation.** A feature service times out, the client fills 0/mean, the model keeps predicting — degraded, uncomplaining. Correction: count and alert on imputation rate per feature; a step-change in "fraction of predictions using ≥1 fallback value" is a paging alert. Distinguish "feature legitimately 0" from "feature missing" in the encoding.
- **Retraining on post-decision data without noticing.** The training query innocently joins to a table that already reflects the model's decisions (e.g., "transactions" excludes blocked ones). Correction: audit every training-set join against the feedback-loop map; document which slices are exploration/holdback and therefore unbiased.
- **Eval set frozen in 2-years-ago traffic.** The model "improves" on a distribution that no longer exists. Correction: refresh eval data on a schedule; keep the old set too so you can distinguish "model got worse" from "world got harder."
- **Schema drift from upstream teams.** A producer renames a field or changes units (cents→dollars); your pipeline coerces and continues. Correction: schema contracts with explicit versioning on every consumed feed; alert on unseen enum values and unit-scale shifts (a 100x mean shift in one feature is upstream units, not user behavior).
- **Timezone/date-boundary bugs in features.** "Purchases today" computed in UTC at training and local time at serving; features spanning a daylight-saving transition. Shows up as a mild, periodic accuracy dip nobody attributes correctly. Correction: all feature timestamps in UTC end-to-end, tested across a DST boundary.
- **Canary judged on the metric the model optimizes rather than guardrails.** A recommender canary "wins" on clicks while quietly tanking diversity/returns. Correction: pre-register guardrail metrics with thresholds before the rollout starts.
- **Warm-start/cold-start asymmetry in the comparison.** The incumbent has weeks of cache warmth, per-user state, or downstream systems tuned to its score distribution; the challenger's scores hit calibrated thresholds set for the incumbent. Correction: recalibrate thresholds per model version (or calibrate scores to a common scale) before comparing; give shadow mode time to warm caches.
- **Treating the feature store as optional plumbing.** Ad-hoc per-model feature pipelines multiply skew surfaces and make every new model a fresh chance to reimplement `days_since_signup` wrong. Correction: shared, tested, point-in-time-correct feature definitions are the investment that compounds.
- **No ownership of the model in production.** Trained by one team, served by another, monitored by neither. Every production model needs a named owner, a runbook (top 5 failure modes + responses), and a deprecation plan.

## Worked micro-example: diagnosing "offline 0.91 AUC, online performs like 0.78"

Expert debugging order (fastest elimination first):
1. **Check finish/serving config parity** — same model file actually deployed? (hash the artifact); same preprocessing version? Cheap and embarrassingly often the answer.
2. **Feature parity replay**: take 1,000 logged production requests, recompute features through the *training* pipeline, diff against the features served. Any feature mismatching >0.1% of the time is a suspect. This finds dual-implementation skew directly.
3. **Time-travel audit**: for the top-10 most important features (by model attribution), verify each was computed point-in-time-correctly in training. A feature like `user_lifetime_purchases` computed "as of today" instead of "as of the example's timestamp" inflates offline AUC exactly this way.
4. **Population audit**: compare training-set filters vs live traffic (bots, new users with empty features, countries excluded offline).
5. Only if 1–4 pass, consider genuine drift — check whether offline eval on the *most recent* labeled window also shows 0.78 (then the world changed, retrain), vs still 0.91 (then it's skew you haven't found; go back to step 2 with more features).

## Verification checklist before signing off on a production ML design or diagnosis

- [ ] Serving-time feature logging in place (or an explicit parity-test regime); training reads from served-feature logs where possible.
- [ ] Point-in-time correctness asserted for every feature built from historical data.
- [ ] Monitoring covers output distributions and feature freshness, sliced by key segments — not just service health.
- [ ] Lineage: model → code SHA → data snapshot → feature versions all resolvable; rollback is a rehearsed config flip; no "latest" tags.
- [ ] Rollout plan: shadow, entity-randomized canary with automatic guardrails, long-term holdback slice retained.
- [ ] Feedback-loop analysis written down; exploration/holdback data feeding retraining; labels for eval collected model-blind.
- [ ] Retraining cadence justified by measured decay, gated by eval vs incumbent.
- [ ] Offline-online correlation checked historically before treating offline gains as real.
