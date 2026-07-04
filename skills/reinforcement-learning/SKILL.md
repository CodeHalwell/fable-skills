---
name: reinforcement-learning
description: Load for reinforcement-learning decisions — judging whether RL is even the right tool, choosing policy-gradient vs value-based vs actor-critic, designing rewards against reward hacking, PPO implementation details, exploration strategies, RLHF/RLAIF for language models (KL penalties, reward-model overoptimization), and evaluating RL results without fooling yourself.
---

# Reinforcement Learning That Works

## Core mental model

- **RL is the tool of last resort.** It pays for sequential credit assignment with sample inefficiency, instability, and evaluation noise. If the problem can be reformulated as supervised learning (you have demonstrations), imitation + finetuning, or a contextual bandit (actions don't affect future state), do that instead — those alternatives are 10–1000× more sample-efficient and vastly easier to debug. Most requests that arrive labeled "RL problem" are not.
- **Reward hacking is the default outcome, not an edge case.** The optimizer will find the maximum of the *reward function you wrote*, not the behavior you meant. Design rewards adversarially: before training, spend real effort answering "what is the laziest, most degenerate policy that maxes this reward?" — the algorithm will find it faster than you did.
- **The problem hierarchy is credit assignment × exploration.** Dense reward + short horizon: easy (bandit-adjacent). Sparse reward + long horizon: hard (needs shaping, curricula, or demonstrations). Locate the problem on this grid before choosing an algorithm; algorithm choice matters far less than reward density and horizon.
- **Variance is the central enemy of policy gradients; bias is the central enemy of value methods.** Every major algorithm is a point on the bias-variance tradeoff: REINFORCE (unbiased, huge variance) → actor-critic with GAE (tunable) → Q-learning (biased by bootstrapping + max-overestimation, low variance).
- **RL results are noise until proven otherwise.** Seed variance in deep RL is enormous; a single-seed improvement is not a result.

## Is RL even the right tool?

Decision ladder — take the first rung that applies:
1. **Demonstrations exist** → supervised/imitation learning (behavior cloning) first; RL at most as a finetuning stage on top. BC is the strongest underrated baseline; report it before any RL claim.
2. **Actions don't influence the next state** (recommendations, ad choice, one-shot decisions with feedback) → contextual bandit (LinUCB, Thompson sampling). Using full RL here adds the credit-assignment machinery for a problem that has none, plus its instability.
3. **A simulator/world model exists and the dynamics are known** → planning/MPC/search may beat learned policies outright.
4. **Reward is truly sequential, delayed, and you can afford millions of environment steps (or have a cheap simulator)** → RL is on the table.
Red flags that RL was chosen for fashion: "we'll learn the reward as we go," no simulator + real-world-expensive samples, horizon of 1, or the reward is a differentiable function of the action (then just do gradient descent on it directly).

## Algorithm selection

| Situation | Choose | Reasoning |
|---|---|---|
| Continuous control, simulator available | PPO (robust default) or SAC (better sample efficiency, off-policy) | PPO tolerates hyperparameter slop; SAC when env steps are the bottleneck |
| Discrete actions, cheap simulator, need sample efficiency | DQN-family (double + dueling + prioritized replay) | replay reuse; but brittle — mind overestimation |
| Expensive/limited env interaction | off-policy (SAC/TD3) or offline RL (CQL/IQL-style) from logged data; question whether RL fits at all | on-policy PPO throws away every batch after one update cycle |
| Large-scale, parallel envs, stability over efficiency | PPO | scales embarrassingly with actors; the industry default for a reason |
| LLM finetuning from preferences | PPO-based RLHF or simpler preference methods (DPO-style) — see below | |
| Very sparse reward, long horizon | first fix the problem: reward shaping (potential-based), curriculum, demonstrations (RL-from-demos), hindsight relabeling (HER for goal-conditioned) | no algorithm choice rescues a reward the policy never encounters |

- Policy gradient vs value-based, compressed: PG optimizes the objective directly (handles continuous/stochastic actions naturally, stable-ish, on-policy-expensive); value-based learns Q and acts greedily (sample-efficient via replay, discrete-action-native, prone to divergence via the deadly triad: bootstrapping + function approximation + off-policy). Actor-critic hybrids are the default because the critic slashes PG variance while the actor keeps optimization direct.
- Discount factor γ is a horizon knob, not a constant to leave at 0.99 unexamined: effective horizon ≈ 1/(1−γ). γ=0.99 ≈ 100 steps. If rewards arrive 500 steps out, γ=0.99 has structurally buried them.

## Reward design (adversarial mindset)

- Write the reward, then role-play the exploiter: reward for forward velocity → policy flings itself and falls; reward per collected item → loop respawn farming; negative reward per timestep → policy learns to terminate itself ASAP (suicide beats slow success); reward for "not crashing" → policy refuses to move. All of these are classics that recur.
- **Potential-based shaping is the only shaping that's safe by construction**: F(s,s') = γΦ(s') − Φ(s) provably preserves the optimal policy. Anything else (bonuses for subgoals, distance-to-goal terms added naively) changes the optimum — the policy may loop through the bonus forever. If a shaped agent does something cyclic and weird, check for a positive-reward cycle in the shaping first.
- Prefer terminal/true-objective reward + shaping via Φ over hand-crafted dense proxies. Every proxy term is an attack surface.
- Instrument the *true* objective separately from the training reward and monitor both; divergence between them = hacking in progress. This is the single most valuable dashboard in any RL project.
- Termination conditions are reward design: episode-end on failure implicitly rewards ending vs suffering negative reward; time-limit truncation must be treated as non-terminal in bootstrapping (`bootstrap on truncation`) — the classic bug is passing `done=True` at time limits, which teaches the agent that time running out is death, corrupting values near the horizon.

## PPO: the stability tricks and why it dominates

Why PPO wins in practice: plain policy gradients allow one bad batch to take a destructively large policy step (policy collapse, unrecoverable because future data comes from the broken policy). PPO's clipped surrogate — ratio r = π_new(a|s)/π_old(a|s), objective min(r·A, clip(r, 1−ε, 1+ε)·A), ε≈0.2 — makes the objective flat once the ratio leaves the trust region, bounding per-update movement while permitting multiple epochs of minibatch reuse on each rollout. It's approximate trust-region at first-order cost.
Implementation details that are actually load-bearing (each one, when missing, has sunk real projects):
- **GAE** (λ≈0.95) for advantages — the main variance-bias dial.
- **Advantage normalization** per batch (mean 0, std 1) — without it, LR is effectively coupled to reward scale.
- **Value-function loss clipping / separate value LR**, value loss coefficient ~0.5.
- **Entropy bonus** (~0.01 coefficient) to delay premature determinism; decay it if final policy must be sharp.
- **Reward normalization/scaling** (running std) — unnormalized rewards of scale 1000 silently break everything downstream.
- **Orthogonal init + small final-layer gain** on the policy head; observation normalization for continuous control.
- **Correct `done` handling at truncation** (see above).
Diagnostics: watch approx-KL per update (spikes → LR too high or too many epochs per batch), clip fraction (healthy ~0.1–0.3; ~0 means updates too timid, ~0.5+ means too aggressive), explained variance of the value function (near 0 or negative → critic useless → advantages are noise).

## Exploration

- ε-greedy/Gaussian action noise: sufficient for dense-reward problems; insufficient by construction for sparse ones (random walk never finds the needle).
- Sparse-reward escalation path, in order of cheapness: (1) reward shaping via potential Φ, (2) curriculum (start states near the goal, expand outward), (3) demonstrations mixed into the buffer or BC-pretrained policy, (4) goal relabeling (HER) when goal-conditioned, (5) intrinsic motivation (count-based novelty, RND-style prediction error). Intrinsic bonuses are last because they add their own hacking surface — the noisy-TV problem: an agent staring at stochastic noise is maximally "novel" forever.
- Entropy regularization (PPO bonus, SAC's temperature) is exploration in the policy-space sense — it keeps options alive but doesn't seek out unvisited states; don't confuse the two roles.

## Debugging workflow and starting hyperparameters

Establish the ladder of sanity checks before believing any result, ascending only when the rung below passes:
1. **Random policy baseline**: run 100 episodes with random actions; record mean return. Every future number is reported relative to this. Surprisingly many "learning" curves never beat it.
2. **Cheat policy**: if a hand-coded heuristic gets near-max reward, the env is easy and RL failing means implementation bugs, not hard exploration.
3. **Overfit one configuration**: fixed seed, single initial state — the agent must solve it near-perfectly. Failure here is always a bug (reward wiring, done handling, action scaling), never "needs more samples."
4. **Known-good env cross-check**: run your agent code on CartPole/Pendulum with reference hyperparameters. If it fails there, stop blaming your env.
5. Only now: full env, multiple seeds.

PPO starting points that work across most continuous-control and small discrete problems (deviate deliberately, not by copy-paste accretion): lr 3e-4 (anneal to 0), γ 0.99 (recompute from horizon: γ ≈ 1 − 1/expected_steps_to_reward), GAE λ 0.95, clip ε 0.2, epochs/batch 4–10, minibatches 4–32, entropy coef 0.01 (0.0 for near-deterministic tasks), value coef 0.5, grad-norm clip 0.5, parallel envs 8–64. For RLHF-PPO: much smaller lr (~1e-6 to 1e-5 on the policy), 1 epoch per batch, and KL coefficient tuned to hold KL-per-token in roughly the low single digits of nats per response — but treat KL target as the primary knob, not lr.

## RLHF/RLAIF for language models

- Pipeline: SFT → reward model (RM) trained on preference pairs (Bradley-Terry loss on chosen vs rejected) → PPO on the policy with per-token reward = RM score at sequence end minus **β·KL(π‖π_SFT)**.
- **The KL constraint is load-bearing for two distinct reasons**: (1) the RM is only trustworthy near the SFT distribution it was trained on — wander off-distribution and the RM's scores are fiction; (2) it prevents mode collapse into a narrow band of high-RM sycophantic phrasing. Symptoms of β too low: reward climbs while outputs converge on repetitive, unctuous, list-heavy boilerplate; PPL against the SFT model explodes. β too high: nothing changes. Tune by watching KL-per-token alongside RM score.
- **RM overoptimization is Goodhart's law made concrete**: true quality rises with RM score up to a point, then *falls* while RM score keeps climbing. Assume the relationship is an inverted U. Defenses: hold out human eval (or a stronger judge) evaluated periodically against checkpoints — pick the checkpoint by held-out quality, not final RM score; RM ensembles; early stopping on KL budget rather than reward plateau.
- Best-of-n sampling against the RM is the embarrassing baseline: no RL, often captures much of the gain, and measures RM exploitability cheaply (if best-of-64 outputs look degenerate, your RM will be hacked by PPO too).
- Direct preference methods (DPO-family) skip the RM+PPO machinery by optimizing preferences directly — far simpler, no rollouts; tend to be strong when preference data is on-policy-ish and plentiful; PPO retains the edge when you need online sampling, explicit reward shaping, or non-preference reward terms (safety classifiers, format checks). Recommending "just DPO it" or "you need full PPO" without asking about data provenance and reward composition is malpractice in both directions.
- RLAIF: replace human labels with an AI judge — inherits every judge bias at scale (verbosity preference, position bias in pairwise comparisons — randomize order, self-preference for its own style). Calibrate the judge against a human-labeled slice before trusting it.
- Verifiable rewards (unit tests, exact-match answers) change the picture: hacking surface shrinks dramatically, KL constraint can loosen — but format hacks (printing the expected answer without computing it, tests gamed via hardcoding) still occur; sandbox and audit samples.

## Failure modes & pitfalls (implementation level)

- **Time-limit truncation marked as terminal** (`done=True` at `TimeLimit`) — corrupts bootstrapping; with Gymnasium, distinguish `terminated` from `truncated` and bootstrap V(s') when truncated. The single most common serious bug in custom training loops.
- **Forgetting to stop gradients through the target** in TD losses: `loss = (Q(s,a) - (r + γ * Q_target(s', a').detach()))**2` — omitting `.detach()`/target network makes the loss minimize by moving the target, which "converges" to garbage smoothly.
- **Computing GAE with stale values or across episode boundaries**: advantages must reset at `done`; a vectorized implementation that carries the recursion across env resets blends unrelated episodes.
- **Sampling actions without storing the log-prob used at sample time**: PPO ratios must use π_old evaluated *at collection*; recomputing "old" log-probs after the first gradient step makes every ratio 1 and PPO degenerates to vanilla PG with extra steps.
- **Tanh-squashed Gaussian without the log-prob correction** (SAC-style): `log_prob -= sum(log(1 - tanh(u)^2))`; omitting it biases entropy and breaks temperature tuning.
- **Observation normalization statistics updating during evaluation** or not checkpointed with the policy — the deployed policy sees differently-scaled inputs than it trained on.
- **Replay buffer storing post-normalization observations** while the normalizer keeps adapting — old transitions become inconsistent with current normalization.
- **Q-value overestimation ignored in DQN variants**: max operator + noise → systematic positive bias; symptoms are Q-values climbing far above any achievable return. Double DQN is the cheap fix; if Q ≫ max possible return, it's this.
- **Epsilon/entropy decayed to zero too fast**: policy commits to the first decent strategy found; learning curves plateau early and identically across seeds. Check the exploration schedule against the time the reward was first found.
- **Reward normalization leaking across the true objective**: normalizing by running return std changes the effective discounting of rare large rewards; log raw returns for interpretation even if training on normalized ones.
- **In RLHF: masking bugs where prompt tokens receive policy gradients** or the KL is computed over prompt+response instead of response-only — both silently shift the objective; verify per-token loss masks on a printed example.
- **Vectorized env auto-reset off-by-one**: most vec-env wrappers return the *new* episode's first observation with the *old* episode's final reward/done; storing `(obs, action, reward, next_obs)` naively pairs the last action with the wrong next state. Use the wrapper's documented `final_observation` field.

## Evaluation pitfalls

- **Seed variance dominates most claimed improvements.** Run ≥5 seeds (10+ for publication-grade), report mean ± std or IQM with bootstrapped CIs, and compare distributions, not best-seed curves. A method beating another by 15% on 3 seeds is indistinguishable from noise in many benchmarks.
- **Env overfitting**: policies exploit simulator quirks (physics-engine contact bugs, fixed spawn points, deterministic opponents). Evaluate with domain randomization, held-out level/task variations, and — if sim-to-real matters — the reality gap eats naive results whole. A policy that only works from the training distribution of initial states hasn't learned the task.
- **Evaluating the stochastic training policy instead of the deterministic/mean policy** (or vice versa) — decide which is deployment-relevant and report it consistently; the gap can be large.
- Training reward curves are not results: report the true objective on held-out evaluation episodes with exploration off.
- Cherry-picked video demos are the field's oldest sin; require aggregate metrics over ≥100 eval episodes.
- For RLHF: win-rate against a fixed baseline judged by a fixed judge, plus KL from SFT, plus capability regression suite (RL for helpfulness can silently degrade reasoning/safety behaviors — always run the pre-RL benchmark battery after).

## Worked micro-example: diagnosing a "working" PPO run

Symptom: CartPole-like custom env, reward rises to near-max, deployed policy jitters and fails. Checklist an expert runs, in order: (1) `done` vs truncation — time-limit steps marked terminal? (found: yes → values near horizon corrupted; fix: bootstrap value on truncation). (2) Eval mode — was "success" the stochastic policy's training return? Evaluate deterministic mean action separately. (3) Observation normalization stats frozen at deployment? A running-mean filter still updating at eval time shifts inputs. (4) Reward instrumented vs true objective — the reward included a small alive-bonus; policy learned to survive while ignoring the balance-quality term. Each of these produces the same curve and a broken policy; the curve alone distinguishes none of them.

## Verification / self-check

1. Did you challenge the premise — could supervised learning or a bandit solve this? State the answer explicitly.
2. Reward audit: name at least two degenerate policies that would maximize the proposed reward; confirm shaping is potential-based or justify why not.
3. Truncation handled correctly (bootstrap, not terminal)? This one bug explains a plurality of "RL almost works" reports.
4. Any performance claim: how many seeds, what variance, deterministic-vs-stochastic eval policy, held-out env variations?
5. RLHF: is KL monitored and bounded, is checkpoint selection by held-out judgment rather than RM score, and was best-of-n run as the baseline?
6. Do the PPO diagnostics (clip fraction, approx-KL, explained variance) support "healthy training," or just the reward curve?
