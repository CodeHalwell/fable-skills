# fable-skills

A library of 51 knowledge-transfer skills authored by Claude Fable 5 to distill expert-level judgment for Claude Opus 4.8 (and other agents). Each skill lives at `skills/<name>/SKILL.md` with standard frontmatter (`name`, `description`) so it can be loaded by Claude Code's skill system.

These are not tutorials. Each skill encodes what a senior practitioner knows that a strong generalist gets wrong: core mental models, decision frameworks, specific failure modes with corrections, worked micro-examples, and verification checklists.

## Usage

Copy the `skills/` directory (or individual skills) into a project's `.claude/skills/`, or point a plugin at this repository. The `description` field in each skill's frontmatter tells the agent when to load it.

## Index

### Coding & Software Engineering
| Skill | Focus |
|---|---|
| `debugging-methodology` | Systematic fault isolation: reproduce-first, bisection, hypothesis ranking |
| `code-review-mastery` | Finding real bugs in diffs; reviewing for what's absent |
| `testing-strategy` | Test design that catches bugs per unit effort; flaky-test root causes |
| `refactoring-safely` | Behavior-preserving change: characterization tests, parallel change, strangler fig |
| `error-handling-design` | Error taxonomy, retry semantics, exceptions vs. result types |
| `api-design` | Interfaces people can't misuse; versioning and compatibility |
| `software-architecture` | Decomposition judgment: coupling, boundaries, monolith-first economics |
| `legacy-code-navigation` | Working in unfamiliar codebases; version-control archaeology |
| `performance-optimization` | Measurement-driven optimization; benchmark pitfalls; tail latency |
| `concurrency-and-parallelism` | Races, memory models, deadlock prevention, async pitfalls |

### Systems & Infrastructure
| Skill | Focus |
|---|---|
| `distributed-systems` | Partial failure, idempotency, consensus, backpressure, retry storms |
| `database-engineering` | Index design, EXPLAIN plans, isolation anomalies, safe migrations |
| `caching-strategies` | Invalidation, stampede prevention, cache-key design |
| `systems-programming` | Memory, ownership, undefined behavior, syscalls, profiling tools |
| `observability-engineering` | Logs/metrics/traces selection, cardinality, SLOs, alert design |
| `security-engineering` | Threat modeling, injection family, authn/authz bugs, secrets |

### AI Engineering
| Skill | Focus |
|---|---|
| `prompt-engineering` | Reliable LLM behavior: instruction placement, few-shot, format anchoring |
| `llm-application-engineering` | Context budgeting, model routing, guardrails, prompt versioning |
| `rag-systems` | Chunking, hybrid search, rerankers, retrieval vs. generation evaluation |
| `agent-design` | Tool design, context management, stopping criteria, agent failure modes |
| `llm-evaluation` | Eval sets, LLM-as-judge design, statistical rigor on small samples |
| `fine-tuning-and-adaptation` | Prompting→RAG→fine-tuning decision ladder; LoRA; data quality |
| `structured-outputs-and-tool-use` | Schema design for LLMs, validation-repair loops, tool definitions |
| `ml-production-systems` | Training-serving skew, drift, shadow deployment, feedback loops |

### Deep Learning
| Skill | Focus |
|---|---|
| `neural-network-training` | Training-run debugging: loss pathologies, LR, mixed precision |
| `transformer-architectures` | Attention, KV-cache arithmetic, positional encodings, MoE |
| `deep-learning-fundamentals` | Backprop, gradient pathologies, double descent, regularization |
| `computer-vision` | CNN vs. ViT, augmentation, transfer learning, evaluation traps |
| `nlp-and-sequence-modeling` | Tokenization mysteries, decoding strategies, embedding-space reasoning |
| `reinforcement-learning` | When RL is wrong, reward hacking, PPO practice, RLHF specifics |

### Mathematics & Algorithms
| Skill | Focus |
|---|---|
| `linear-algebra-for-ml` | Shape discipline, SVD, eigen-intuition, solve-don't-invert |
| `probability-and-statistics` | Conditioning errors, distribution selection, testing landmines |
| `optimization-methods` | Convexity recognition, KKT, solver selection, reformulation |
| `numerical-computing` | Floating point, catastrophic cancellation, log-space, tolerances |
| `complexity-analysis` | Honest Big-O, recurrences, NP-hardness recognition, constants |
| `algorithm-design` | Invariant-first design, greedy proof obligations, reductions |
| `data-structures-selection` | Structure choice by operation profile; hash/heap/tree pitfalls |
| `dynamic-programming` | State design, the four-step protocol, DP family recognition |
| `graph-algorithms` | Hidden-graph modeling, traversal selection, flow reductions |

### Computer Science
| Skill | Focus |
|---|---|
| `compilers-and-parsing` | Lexing/parsing ladder, ASTs, type checking, IR optimization |
| `formal-methods-and-invariants` | Invariants, contracts, state-machine modeling, termination |
| `applied-cryptography` | Composing vetted constructions; nonce misuse; timing channels |

### Chemistry
| Skill | Focus |
|---|---|
| `organic-chemistry-reasoning` | Electron pushing, SN1/SN2/E1/E2 matrix, retrosynthesis |
| `computational-chemistry` | Method ladder (FF→DFT→CCSD(T)), frequency checks, error bars |
| `cheminformatics` | SMILES/RDKit practice, fingerprints, dataset hygiene, scaffold splits |
| `physical-chemistry-and-thermodynamics` | ΔG discipline, kinetics vs. thermodynamics, Nernst conventions |
| `drug-discovery` | SAR reasoning, ADMET constraints, PK essentials, assay interpretation |

### Research & Meta-skills
| Skill | Focus |
|---|---|
| `research-methodology` | Question formulation, literature strategy, source credibility |
| `experiment-design-and-statistics` | Power analysis, A/B landmines, causal-graph confounder reasoning |
| `scientific-writing` | Information hierarchy, claims-evidence audit, figure discipline |
| `technical-problem-solving` | Representation change, extreme cases, estimation, verification mode |
