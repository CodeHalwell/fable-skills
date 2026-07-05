---
name: computer-vision
description: Load for computer-vision engineering decisions — choosing CNN vs ViT, designing augmentation pipelines, transfer-learning/finetuning protocols, picking classification vs detection vs segmentation architectures, image resolution/batch/memory tradeoffs, evaluation metrics (mAP, IoU, imbalanced pixels), or judging when classical CV beats deep learning.
---

# Computer Vision Engineering

Assumed baseline (verified expert-grade cold): pretraining beats architecture choice at ≤50k images and linear-probe-first; label-destroying augmentations (flips vs text/chirality, color jitter vs color-defined labels, crops vs small objects); head-first then discriminative-LR finetuning with BN frozen in eval under small batch/domain shift; nearest-neighbor for mask resize; object-px-after-resize arithmetic and tiling for small objects; mAP/pixel-accuracy pitfalls and clDice/boundary metrics for thin structures; group-based splits against near-duplicate leakage; classical CV for geometry/controlled lighting; BGR/normalization bugs; detector+tracker over video models; look-at-the-data-first debugging; 4×-activations/16×-ViT-attention resolution math; EXIF orientation.

## Discipline rules

- Augmentation review = per-transform invariance audit for *this* task's labels, plus confirming masks/boxes/keypoints are co-transformed **in the same call** (`albumentations` with `bbox_params`/mask targets) — desynced RNG between image and target transforms is the classic silent bug.
- Visualize ~20 augmented (image, label) pairs before training; label destruction is visible instantly and invisible in metrics until too late.
- Compute object-size-in-pixels after resize before any architecture discussion; below ~8 px at standard FPN strides, recommend tiling, not a bigger model.
- Baseline order is non-negotiable: linear probe → full finetune → custom architecture. Push back on proposals that skip to step 3.

## Sharpenings the strong baseline lacks

- **Tiling spec for sliced inference**: overlap ≥ **2× the object size** (not a generic 10–20%) plus cross-tile NMS — smaller overlap splits objects across tile boundaries and double-counts or drops them.
- **Normalization constants come from the weights' preprocessing config, not folklore**: CLIP-pretrained backbones have their *own* mean/std — applying ImageNet stats to them is a silent accuracy tax. Likewise, train–val interpolation mismatch (bilinear vs bicubic, differing crop ratio) costs 1–2% and masquerades as a modeling problem.
- **Letterboxing vs stretching mismatch** between training and serving shifts detection boxes systematically — reuse the exact preprocessing function in the serving stack; never reimplement it.
- **Segmentation void labels**: forgetting `ignore_index=255` (or your void value) in `CrossEntropyLoss` trains the model to *predict void* at boundaries.
- **Detection head grafting**: a new class count grafted with an off-by-one against the background class trains with normal-looking loss while one class is silently unlearnable.
- **Label-noise floor check**: relabel a random 100-sample slice yourself; if your agreement with the dataset is <~95% (classification; lower for boxes/masks), annotator disagreement may already explain the metric gap — no model change can beat it.
- **Grad-CAM is not evidence of correct reasoning** — background-shortcut models (grass→cow) produce plausible object-centered maps. Test shortcuts by masking/replacing backgrounds, not by staring at saliency.
- **TTA inflates reported numbers** unless deployment also runs TTA — label which regime every reported number comes from, same for COCO-protocol (0.001 threshold, 100 dets/img) vs your production threshold.
- **Stain normalization, not naive color jitter, for histopathology** — jitter destroys the diagnostic signal that stain variation encodes.
- Hard-example mining beats generic collection: 500 images from observed failure clusters routinely outperform 10k random ones; sample video frames with temporal stratification (consecutive frames waste label budget and leak).
- Hybrid pattern that wins in industry: classical localization/rectification (homography, threshold, morphology — deterministic, 1000+ fps) feeding a small CNN for only the genuinely semantic decision on the rectified crop. Also: morphology (`MORPH_OPEN`/`CLOSE` + component-area filtering) on noisy neural masks routinely recovers 2–5 IoU points and is the correct *first* response to speckled masks — before collecting data.

## Verification / self-check

1. Every transform named with its invariance assumption; co-transformation confirmed in-code.
2. Splits grouped by patient/video/session and deduped before believing any metric.
3. Metric matches deployment: operating threshold, per-class reporting, boundary sensitivity for thin structures.
4. Resolution story survives the resize arithmetic; activation memory recomputed for any proposed resolution change.
5. Preprocessing config (mean/std, interpolation, letterboxing) read from the weights' source, byte-identical between train and serve.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 13 baseline (cut/compressed), 1 partial (sharpened), 0 delta.
- Biggest baseline gaps found: none major — Opus produced the augmentation-invariance pairs, small-object arithmetic, group-split leakage, and classical-CV routing cold.
- Retained value: exact operating details baseline answers omit (2×-object tiling overlap, CLIP-specific normalization, ignore_index/void, letterbox serve-mismatch, 95% label-audit floor, TTA/threshold reporting regimes).
