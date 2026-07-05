---
name: reinforcement-learning
description: Load for reinforcement-learning decisions — judging whether RL is even the right tool, choosing policy-gradient vs value-based vs actor-critic, designing rewards against reward hacking, PPO implementation details, exploration strategies, RLHF/RLAIF for language models (KL penalties, reward-model overoptimization), and evaluating RL results without fooling yourself.
---

# Reinforcement Learning That Works

Assumed baseline (verified expert-grade cold): RL-as-last-resort decision ladder (demonstrations→BC, no state influence→contextual bandit, known dynamics→planning); degenerate-policy prediction and potential-based shaping as the only optimum-preserving scheme; γ horizon ≈ 1/(1−γ); PPO load-bearing details with values (GAE λ0.95, advantage norm, ortho init + small policy head, lr 3e-4 annealed, entropy 0.01, clip 0.2) and diagnostics with healthy ranges (approx-KL 0.005–0.02, clip fraction 0.05–0.25, explained variance →0.8); terminated-vs-truncated bootstrapping and the TimeLimit bug; RLHF KL rationale, overoptimization inverted-U with checkpoint-by-held-out-eval, best-of-n as the mandatory baseline; DPO-vs-PPO by data provenance and reward composition; stale-log-prob PPO degeneration; offline-RL extrapolation error, filtered-BC-first ladder, stitching requirement; ≥5–10 seeds with IQM + bootstrapped CIs (rliable); sparse-reward escalation ending at intrinsic motivation (noisy-TV); non-Markov plateau signature; tanh-squash correction, GAE episode-boundary resets, vec-env `final_observation`.

## Discipline rules

- State explicitly whether supervised/bandit reformulation was rejected and why, before any algorithm talk.
- Reward audit = name ≥2 degenerate policies that maximize the proposed reward; confirm shaping is potential-based or justify.
- Instrument the *true* objective separately from training reward; divergence between them is hacking in progress — the single most valuable dashboard.
- Recompute γ from the horizon (γ ≈ 1 − 1/expected-steps-to-reward) instead of inheriting 0.99.
- No result claims without: seed count + variance, deterministic-vs-stochastic eval policy stated, held-out env variations, and PPO health diagnostics (not just the reward curve).

## Sharpenings the strong baseline lacks

- **RLHF-PPO hyperparameter regime is its own world**: policy lr ~1e-6 to 1e-5 (1000× below control PPO's 3e-4), **1 epoch per batch** (not 4–10), and the KL budget — roughly low-single-digit nats per response — is the primary knob, tuned before lr. Porting control-PPO defaults into RLHF is a common silent failure.
- **RLHF masking bugs**: prompt tokens receiving policy gradients, or KL computed over prompt+response instead of response-only — both silently shift the objective and pass every unit test. Verify per-token loss/KL masks on a printed example.
- **Verifiable rewards shrink but don't remove the hacking surface**: format hacks (echoing the expected answer without computing it, hardcoding around tests) persist — sandbox and audit samples even with unit-test rewards; the KL constraint can loosen, not vanish.
- **RLAIF judge biases compound at scale**: verbosity preference, pairwise position bias (randomize order), self-preference for its own family's style — calibrate the judge against a human-labeled slice before trusting any win-rate.
- **Post-RL regression battery is mandatory**: RL for helpfulness silently degrades reasoning/safety behaviors — rerun the pre-RL benchmark suite, don't just report win-rate + KL.
- **OPE is statistically brutal**: importance-sampling estimators have variance exponential in horizon; treat long-horizon off-policy value estimates as directional and insist on a small online A/B before shipping decisions on them. Coverage question first: are the new policy's actions ever taken in the logs? Propensity ≈ 0 → no estimator rescues you.
- **Simulator speed budgets the method**: RL needs 1e6–1e9 steps; 10 ms/step caps ~8.6M steps/day/process. If the simulator can't vectorize, that constraint (offline RL, model-based, or not-RL) should be decided before any algorithm discussion.
- **Replay-buffer/normalizer staleness**: storing post-normalization observations while the normalizer keeps adapting makes old transitions inconsistent; freeze and checkpoint normalizer stats with the policy (and stop updating them at eval).
- **Reward normalization distorts rare large rewards**: running-std scaling changes their effective weight — log raw returns for interpretation even when training on normalized ones.
- Episode decomposition beats compute: a 10k-step episode splittable into meaningful 200-step segments with local rewards learns orders of magnitude faster — restructure before scaling.

## Verification / self-check

1. Premise challenged (supervised/bandit) in writing.
2. Two degenerate reward-maximizers named; shaping potential-based or justified; truncation bootstrapped.
3. Claims: seeds/variance/eval-policy/held-out variations stated; PPO diagnostics healthy, not just reward.
4. RLHF: KL monitored and bounded, checkpoint chosen by held-out judgment, best-of-n run, masks verified, regression battery rerun.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 13 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Biggest baseline gaps found: none major — truncation handling, PPO diagnostics with numeric ranges, noisy-TV, offline-RL stitching, and IQM statistics all produced cold.
- Retained value: the RLHF-specific operating regime (lr/epochs/KL-budget numbers, masking bugs, post-RL regression battery) and OPE/simulator budget arithmetic — the places where control-RL intuitions transplant badly.
