# fable-skills

A library of 101 knowledge-transfer skills authored by Claude Fable 5 to distill expert-level judgment for Claude Opus 4.8 (and other agents). Each skill lives at `skills/<name>/SKILL.md` with standard frontmatter (`name`, `description`) so it can be loaded by Claude Code's skill system.

These are not tutorials. Each skill encodes what a senior practitioner knows that a strong generalist gets wrong: core mental models, decision frameworks with explicit reasoning chains, "how an expert thinks through this" walkthroughs (including rejected alternatives), specific failure modes with corrections, worked micro-examples, and verification checklists with stopping rules. Round-2 skills (the practice-oriented ones below) were written against live web research, with fast-moving facts verified and marked "as of 2026".

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

### Language Mastery
| Skill | Focus |
|---|---|
| `typescript-mastery` | Type system as proof assistant: narrowing, generics, variance, tsconfig |
| `python-idioms` | Modern typing, asyncio pitfalls, uv-era packaging, free-threaded Python |
| `rust-development` | Ownership as design forcing-function, smart-pointer chains, async Rust |
| `go-development` | Goroutine lifecycle, channel vs mutex, interface design, slice aliasing |
| `nodejs-backend` | Event-loop model, streams/backpressure, memory-leak hunting, ESM |

### Frontend & Web
| Skill | Focus |
|---|---|
| `frontend-architecture` | State location decision chain, RSC boundary, re-render reasoning |
| `css-and-layout` | Layout algorithm selection, container queries, cascade layers, stacking |
| `web-performance` | Core Web Vitals engineering, loading waterfall, JS cost, budgets |
| `web-accessibility` | Semantic HTML first, ARIA judgment, focus management, layered testing |
| `realtime-web` | Transport selection (SSE default), reconnection engineering, CRDTs |

### Development Practice
| Skill | Focus |
|---|---|
| `git-mastery` | Content-addressable mental model, history surgery, bisect, monorepo scale |
| `cli-tool-design` | stdout/stderr discipline, exit codes, config precedence, destructive safety |
| `dependency-management` | Deps as risk, pinning strategy, supply-chain defense, upgrade cadence |
| `technical-debt-management` | Debt portfolio, hotspot prioritization, rewrite economics |
| `code-generation-with-llms` | Spec quality, context curation, verification asymmetry, AI-code review |

### Cloud & Deployment
| Skill | Focus |
|---|---|
| `cloud-architecture-aws` | Compute/storage decision chains, IAM reasoning, multi-region judgment |
| `cloud-architecture-azure` | Compute selection, Entra-centric identity, Cosmos consistency, landing zones |
| `serverless-architectures` | Cost/latency shape fit, cold starts, event-driven composition, DLQs |
| `cloud-cost-optimization` | Unit economics, the optimization ladder, egress traps, commitments |
| `estimation-and-capacity-planning` | Little's Law, peak-to-average, queueing hockey stick, load testing |
| `kubernetes-operations` | Reconciliation model, debugging decision tree, requests/limits, autoscaling |
| `containerization-docker` | Layer caching, multi-stage builds, PID-1, image security |
| `infrastructure-as-code` | State as crown jewel, plan-review discipline, blast-radius design |
| `ci-cd-pipelines` | Fail-fast design, caching, OIDC security, monorepo CI |
| `platform-engineering` | Golden paths, platform-as-product, cognitive load, abstraction leaks |
| `deployment-strategies` | Rolling/blue-green/canary reasoning, expand-migrate-contract, flags |
| `incident-response` | Mitigation-first, what-changed prior, roles/comms, blameless postmortems |
| `networking-fundamentals` | The request's journey, layer-by-layer debugging, LB and CDN models |
| `authentication-and-identity` | OAuth2/OIDC reasoning, session vs JWT, passkeys, implementation holes |
| `privacy-engineering` | Data minimization, deletion as engineering, anonymization honesty |

### AI Engineering (Production)
| Skill | Focus |
|---|---|
| `llm-inference-optimization` | Prefill/decode asymmetry, KV-cache arithmetic, quantization, speculative decoding |
| `embeddings-and-vector-search` | Embedding/reranker selection, ANN index tradeoffs, filtered search |
| `model-serving-infrastructure` | Serving stack selection, GPU utilization economics, autoscaling, streaming |
| `local-and-open-models` | Open-weight landscape, VRAM arithmetic, local runtimes, local-vs-API |
| `llm-cost-engineering` | Token economics, caching, routing, fine-tune-to-shrink arithmetic |
| `mcp-and-tool-protocols` | MCP server design, transport/auth, tool-poisoning defenses |
| `building-coding-agents` | Agent loop anatomy, context engineering, sandboxing, verification design |
| `multimodal-ai` | VLM capability map, image token economics, document AI, voice pipelines |
| `conversational-ai-design` | Dialogue state, memory architecture, grounding, conversation repair |
| `ai-guardrails-and-red-teaming` | Layered defenses, prompt injection, red-team methodology, residual risk |
| `ai-data-engineering` | Curation cascade, dedup/decontamination, synthetic data judgment |
| `llm-observability` | Traces as debugging unit, online eval, drift detection, feedback loops |

### Data Engineering
| Skill | Focus |
|---|---|
| `data-pipelines` | Idempotency, incremental processing, orchestration, ELT reasoning |
| `streaming-systems` | Kafka model, delivery semantics honesty, watermarks, when batch wins |
| `analytics-engineering` | Dimensional modeling, dbt-era layering, warehouse cost levers, fan-out bug |
| `postgres-mastery` | MVCC explains everything, planner conversations, indexes, pooling, locks |
| `data-quality-and-governance` | Data contracts, validation layering, lineage, incident response for data |
| `api-integration-patterns` | Resilient clients, rate limits, webhook engineering, integration failures |
| `wasm-and-edge-computing` | Wasm mental model, edge platform reasoning, data-locality constraint |
| `simulation-and-scientific-computing` | Modeling ladder, solver selection, Monte Carlo, JAX-era, validation |
