---
name: fine-tuning-and-adaptation
description: Load when deciding whether to fine-tune an LLM, designing SFT/LoRA/QLoRA/DPO training runs, curating fine-tuning data, choosing hyperparameters, or debugging a fine-tuned model that got worse (forgetting, format lock-in, overfitting). Also for "should we fine-tune or just prompt/RAG?" questions.
---

# Fine-Tuning and Model Adaptation

## Core mental model

- **Fine-tuning teaches form and behavior, not facts.** SFT reliably changes *how* a model responds (format, style, tone, task procedure, output schema, persona) but is a poor and unreliable way to inject *knowledge*. New facts belong in the context window (RAG). Teams that fine-tune "to teach the model our product docs" get a model that confidently hallucinates in their house style.
- **Data quality dominates everything.** 1,000 excellent, consistent, correct examples beat 100,000 mediocre ones — noisy data doesn't average out, it teaches the noise. Every hour spent on hyperparameter search before the data is clean is wasted. The model becomes the median of your dataset: if 10% of your examples have a subtle format error, the model will produce it far more than 10% of the time, because it also learns your inconsistency as "format is optional."
- **Fine-tuning is a last resort with a maintenance tax.** Every base-model upgrade forces re-tuning and re-evaluation; the fine-tuned checkpoint freezes you to a model generation. Exhaust cheaper rungs first, and keep the eval set that justified the decision — it's the regression suite forever after.
- **You can only fine-tune what you can already evaluate.** No eval set → no way to know if the tune helped, hurt, or silently broke adjacent capabilities. Build the eval first (see llm-evaluation skill).

## The decision ladder — and the evidence that justifies each step

Climb only when the current rung has demonstrably failed *on a fixed eval set*:

1. **Prompting (zero-shot, better instructions).** Exhausted only when you've genuinely iterated: explicit rubrics, decomposed steps, counterexamples in the instructions. Most "we need fine-tuning" conclusions die here.
2. **Few-shot examples in context.** Evidence to move up: adding 5–10 well-chosen examples plateaus below target. If few-shot examples help a lot, that's evidence fine-tuning *will* work well (the task is learnable from examples) — and often 20–50 in-context examples with prompt caching is cheaper than a tune.
3. **RAG.** Required (not optional) when failures are missing/stale/private *knowledge*. Evidence to move past it: retrieval is verifiably returning the right context and the model still fails to use it correctly.
4. **Fine-tuning.** Justified when: (a) failures are behavioral (format, style, procedure) and persist with correct context and good prompts; (b) you need a small/fast/cheap model to match a big model on ONE narrow task (distillation — the strongest economic case: generate targets with the big model, filter for correctness, SFT the small one); (c) prompt length for instructions+examples is a dominant cost at high volume; (d) latency budget can't fit the mega-prompt.

**Anti-patterns for jumping to fine-tuning:** "the model doesn't know our data" (→ RAG), "the model reasons badly" (SFT rarely improves general reasoning and often hurts it), "we have data lying around" (data availability is not a use case).

## SFT data curation rules

- Target the *inference-time* distribution: same prompt template, same context format (including retrieved passages if you use RAG at inference), same input lengths. Distribution mismatch between tuning data and serving traffic is the top cause of "trained great, serves badly."
- Deduplicate near-duplicates (embedding similarity or MinHash). 50k examples that are 5k patterns × 10 paraphrases teach 5k things while telling your metrics you taught 50k, and overweight those patterns.
- Audit manually: read 100 random examples yourself. Every incorrect label you find at rate r% is being *learned* at rate r%. Cheap filter that works: score every example with a strong LLM against a correctness rubric, drop the bottom 10–20%, spot-check the drops.
- Include refusal/edge examples: if the model should say "I don't have enough information" sometimes, that behavior must be in the data at a realistic rate, or the tune will erase it.
- Mask the loss on the prompt tokens (`labels = -100` on the input portion in HF `transformers`) — compute loss on completion tokens only. Training loss on prompts wastes capacity memorizing your inputs and drags metrics.

## LoRA / QLoRA vs full fine-tuning

| Choice | Use when | Notes |
|---|---|---|
| LoRA | Default for task adaptation on 7B+ models | Matches full FT on most narrow tasks; adapters are swappable per-task on one base model; far less catastrophic forgetting because 99%+ of weights are frozen. |
| QLoRA (4-bit base + LoRA) | GPU memory is the constraint | Quality within noise of LoRA for most tasks; ~2–3× slower training than LoRA on the same hardware; use `bnb_4bit_compute_dtype=torch.bfloat16` and NF4 quantization. |
| Full fine-tuning | Large behavioral shifts (new language, heavy domain shift, pretraining-style continued training), or LoRA at high rank has verifiably plateaued below target | Needs much more data and compute; much higher forgetting risk; only after LoRA has been tried and measured. |

LoRA parameter guidance (defaults that work; tune only with eval evidence):
- `r=16, lora_alpha=32` (keep alpha ≈ 2×r; alpha/r is effectively a scaling on the adapter — raising alpha at fixed r behaves like raising LR). Raise r (32–64) for harder/broader tasks; r>64 rarely helps and mostly overfits.
- **Target all linear layers** (`q,k,v,o,gate,up,down` projections), not just `q_proj,v_proj`. Attention-only LoRA is a legacy default from the original paper and measurably underperforms; targeting MLP layers matters more than raising r.
- `lora_dropout=0.05` for datasets under ~10k examples; 0 for large ones.
- LR for LoRA is ~10× full-FT LR: start 1e-4–2e-4 (vs 1e-5–2e-5 for full FT), cosine decay, warmup 3–10% of steps.

## Catastrophic forgetting — detection and mitigation

Fine-tuning on a narrow distribution degrades everything outside it: general knowledge, instruction following, safety behavior, other languages, and *format flexibility* (the model starts answering everything in your training format — a tune on JSON extraction will start emitting JSON when asked casual questions).

- **Detect:** run a general-capability probe suite (a few hundred items spanning chat, reasoning, safety refusals, and format variety) before and after the tune. If you only eval the target task, forgetting is invisible until production.
- **Mitigate, in order of effectiveness:** (1) train less — fewer epochs, lower LR, LoRA instead of full FT; (2) **mix in 10–30% general instruction data** (generic chat/instruct examples) with your task data — the single most effective cheap fix; (3) lower LoRA rank; (4) early-stop on the *general* suite, not just task loss.
- Safety-behavior regression deserves explicit checking: even benign-task SFT measurably weakens refusal behavior. Include refusal probes in the before/after suite.

## Preference optimization (DPO/RLHF) vs SFT

- **SFT** imitates demonstrations: it can teach the model *what a good answer looks like*. It cannot teach *which of two plausible answers is better*, and it can't push down behaviors (hedging, sycophancy, verbosity) that appear in no training example but emerge anyway — SFT only ever adds positive examples.
- **DPO/RLHF** optimizes a preference signal: right tool when you have (or can generate) *pairs* — "this answer over that one" — especially for style, harmlessness, conciseness, and stamping out a specific recurring bad behavior (use the bad behavior as the rejected response).
- **Order matters: SFT first, then DPO.** DPO on a model that can't produce the desired format/behavior at all just reweights garbage. DPO assumes both chosen and rejected are in-distribution outputs.
- DPO cannot inject knowledge or new skills either — it reshuffles probability mass among behaviors the model already has.
- DPO practicals: `beta=0.1` default (lower = stronger drift from reference, more reward hacking of the preference data; higher = weaker effect); LR ~10× *lower* than your SFT LR (e.g. 5e-7–5e-6 full FT); rejected responses should be *plausible near-misses* (ideally sampled from the SFT model itself), not strawmen — pairs like "good answer vs. gibberish" teach nothing. Watch for length hacking: if preference data even slightly favors longer answers, DPO amplifies it and outputs balloon.

## Hyperparameters that actually matter (small-data SFT)

Priority order: **data quality ≫ LR > epochs > everything else.** Batch size, scheduler shape, and optimizer choice are second-order at this scale.

- **LR:** the one knob that ruins runs. Too high → loss spike then a permanently degraded model (or NaN); slightly too high → model trains but comes out subtly dumber. When unsure, go lower and train slightly longer.
- **Epochs:** 1–3 for datasets ≥ ~10k; 3–5 only for very small (≤1k) sets, with heavy eval monitoring. More epochs on small data = memorization: the model reproduces training completions verbatim on near-match inputs and degrades on everything else.
- **Overfit detection on small data:** hold out 10% *before* training, no exceptions. The signal is eval loss rising while train loss falls — but for generation tasks eval *loss* can look fine while generation quality collapses, so run actual generation evals (task metric on held-out prompts) at each epoch boundary and checkpoint each epoch. Pick the checkpoint by task metric, not by train loss. A telltale: sample the tuned model on a paraphrase of a training input — verbatim regurgitation of a training completion means it memorized.
- Effective batch: 16–64 sequences via gradient accumulation; below ~8 gradients get noisy, above ~128 with small data you get too few optimizer steps per epoch to learn.

## Data formatting / template traps (a leading cause of "fine-tune made it worse")

- **Chat-template mismatch is the #1 silent killer.** Training with template A (or raw concatenated text) and serving with template B — different special tokens, role markers, whitespace — makes the model measurably worse than baseline. Always format training data with the *exact* serving template: use `tokenizer.apply_chat_template(...)` for both, never hand-rolled f-strings.
- EOS handling: the training collator must include the EOS token at the end of each completion with loss computed on it, or the tuned model never learns to stop and generates until max_tokens.
- System prompt consistency: if training examples have no system prompt but serving does (or a different one), you've created train/serve skew. Bake the serving system prompt into training data, or vary it deliberately across examples to teach robustness.
- Invisible characters: mixed `\r\n` vs `\n`, non-breaking spaces, or a stray leading space before completions each tokenize differently and the model learns them. Diff a fully-rendered training string against a fully-rendered serving prompt **at the token-ID level** before any run.

## Worked micro-example: QLoRA SFT that avoids the classic traps

```python
from transformers import AutoModelForCausalLM, AutoTokenizer, BitsAndBytesConfig
from peft import LoraConfig
from trl import SFTTrainer, SFTConfig
import torch

tok = AutoTokenizer.from_pretrained(BASE)
model = AutoModelForCausalLM.from_pretrained(
    BASE,
    quantization_config=BitsAndBytesConfig(
        load_in_4bit=True, bnb_4bit_quant_type="nf4",
        bnb_4bit_compute_dtype=torch.bfloat16),
)
peft_cfg = LoraConfig(
    r=16, lora_alpha=32, lora_dropout=0.05, task_type="CAUSAL_LM",
    target_modules=["q_proj","k_proj","v_proj","o_proj",
                    "gate_proj","up_proj","down_proj"],  # all linear, not just q/v
)
# dataset rows: {"messages":[{"role":"system",...},{"role":"user",...},{"role":"assistant",...}]}
# -> SFTTrainer applies tok.apply_chat_template; verify one rendered sample token-by-token
args = SFTConfig(
    num_train_epochs=2, learning_rate=2e-4, lr_scheduler_type="cosine",
    warmup_ratio=0.05, per_device_train_batch_size=4,
    gradient_accumulation_steps=8,            # effective batch 32
    bf16=True, eval_strategy="epoch", save_strategy="epoch",
    assistant_only_loss=True,                 # loss on completions only
)
trainer = SFTTrainer(model=model, args=args, peft_config=peft_cfg,
                     train_dataset=train_ds, eval_dataset=val_ds)
print(tok.decode(trainer.train_dataset[0]["input_ids"]))  # EYEBALL the rendered template + EOS
trainer.train()
```

Checkpoint selection afterward: run the task eval **and** the general-capability probe on each epoch checkpoint; take the best task score whose general score hasn't dropped more than your tolerance.

## Verification checklist before recommending or shipping a fine-tune

- [ ] Ladder evidence: prompting and few-shot demonstrably plateaued on a fixed eval set; knowledge gaps routed to RAG, not the tune.
- [ ] 100 training examples read by a human; near-duplicates removed; label error rate estimated.
- [ ] Rendered training prompt == rendered serving prompt at the token level (template, system prompt, EOS).
- [ ] Loss masked to completion tokens.
- [ ] Held-out split created before training; checkpoint chosen by held-out *task metric*, not train loss.
- [ ] Before/after comparison on: task eval, general-capability probe, refusal/safety probe, and output-format flexibility (ask it a casual question — does it answer in training format?).
- [ ] Tuned model beats the best prompted baseline **on the same eval**, by a margin exceeding the eval's noise floor — otherwise recommend shipping the prompt.
