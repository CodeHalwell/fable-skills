---
name: computer-vision
description: Load for computer-vision engineering decisions — choosing CNN vs ViT, designing augmentation pipelines, transfer-learning/finetuning protocols, picking classification vs detection vs segmentation architectures, image resolution/batch/memory tradeoffs, evaluation metrics (mAP, IoU, imbalanced pixels), or judging when classical CV beats deep learning.
---

# Computer Vision Engineering

## Core mental model

- **Vision is priors + data on a budget.** CNNs bake in locality, translation equivariance, and hierarchy — priors that substitute for data. ViTs learn those priors from data — which they need a lot of (or heavy augmentation/distillation) to acquire. Every CNN-vs-ViT decision is "how much data/pretraining do I have to buy back the priors?"
- **Augmentation is the highest-ROI lever in most applied CV projects.** A day spent designing task-correct augmentation routinely beats a month of architecture search. But every augmentation encodes an invariance assumption, and a wrong assumption *destroys labels silently*.
- **Pretrained-then-finetune is the default; training from scratch is the exception** requiring justification (huge dataset, exotic modality like multispectral with incompatible channel counts, or licensing).
- **Task structure dictates architecture family**, not fashion: per-image label → classification; instances with locations → detection; per-pixel labels → segmentation; instance masks → instance segmentation. Most "which model" questions are settled by the label format, then by latency budget.
- **Deep learning is not the default for structured problems.** Known geometry, controlled lighting, rigid objects, calibrated cameras → classical CV (homography, filtering, morphology, template matching) is faster, more accurate, explainable, and needs zero training data. Ask "is the variability actually semantic?" before reaching for a network.

## CNN vs ViT decision rules

- < ~50k task images and no strong pretrained ViT available for your domain → CNN (ConvNeXt/ResNet/EfficientNet class) finetuned from ImageNet-scale weights. The inductive bias wins in low data.
- Strong pretrained ViT exists (CLIP/DINOv2-style self-supervised weights) → ViT features often win even with small task data, *frozen or lightly tuned* — the pretraining bought the priors. Try a linear probe on frozen features first; it's a 30-minute experiment that sets the baseline.
- Dense prediction at high resolution: vanilla ViT cost grows quadratically in tokens (a 1024² image at patch 16 → 4096 tokens); prefer hierarchical/windowed backbones or CNNs unless you need ViT's pretraining.
- Deployment on edge/CPU → CNNs quantize and prune more predictably; depthwise-separable families exist for this.
- Don't say "ViTs need more data" unconditionally — that's the from-scratch story. With modern self-supervised pretraining the practical rule is: **ViT if you can inherit great pretraining; CNN if you're closer to scratch or resolution/latency-bound.**

## Augmentation: design by invariance audit

Procedure: list the transforms; for each, ask "does the label survive?" for *this* task — not in general.

| Augmentation | Safe for | **Destroys labels for** |
|---|---|---|
| Horizontal flip | natural objects, scenes | text/digits (b↔d, ↔ mirrored characters), chirality tasks (left vs right anatomy — a flipped chest X-ray has the heart on the wrong side), traffic signs, any left/right label |
| Vertical flip | aerial/satellite, microscopy | almost all natural photos (gravity prior), documents |
| 90° rotations | aerial, pathology slides | scenes with canonical orientation, digits (6↔9) |
| Small rotation/affine | most tasks | precise keypoint/measurement tasks unless keypoints are co-transformed |
| Color jitter / grayscale | object category tasks | color-defined labels: ripeness, dermatology lesion color, stained histopathology (use stain normalization instead of naive jitter), wire-color identification |
| Aggressive crop (RandomResizedCrop low scale) | classification of large objects | small-object detection (object cropped out but box remains), counting tasks, global-context labels |
| Cutout / random erasing | classification | can delete the entire small object; for detection require overlap checks |
| MixUp / CutMix | classification with soft-label-capable loss | detection/segmentation (labels don't mix cleanly), calibration-critical tasks; also incompatible with hard-label metric learning |
| JPEG/noise/blur | robustness goals | fine-texture labels (defect detection where the defect *is* high-frequency texture) |

Rules:
- Geometric augs must be applied identically to masks/boxes/keypoints (use `albumentations` with `bbox_params`/`mask` targets, not two independent transform calls — a desynced random seed between image and mask transforms is a classic silent bug).
- Match test-time distribution: if deployment images are always upright and well-lit, heavy rotation/color augs waste capacity on invariances you don't need (mild versions still help as regularization).
- Start from a known-good recipe (e.g., RandAugment for classification) and *subtract* label-destroying ops for your task; don't build up from nothing.
- Normalize with the pretrained backbone's mean/std, and keep train/val preprocessing identical except augmentation. A train-val resize-interpolation mismatch (bilinear vs bicubic, or different crop ratio) costs 1–2% accuracy and looks like a modeling problem.

## Transfer learning protocol (default recipe)

1. **Linear probe first**: freeze backbone, train head, ~few epochs. Establishes the floor and tests your pipeline cheaply. If linear probe is near random, suspect data/label bugs before architecture.
2. **Then finetune**: unfreeze all, with discriminative LRs — head ~1e-3, backbone ~1e-4 to 1e-5 (10–100× lower). Rationale: pretrained features are near a good minimum; large LR destroys them before the head learns to use them ("catastrophic forgetting in the first 100 steps").
3. Warmup matters more when unfreezing: the random head sends garbage gradients into the backbone initially; either warm the head up frozen-backbone first (step 1 doubles as this) or use LR warmup.
4. Progressive unfreezing (top blocks first) helps mainly when task data is tiny (<~5k images); otherwise full finetune with discriminative LR is simpler and as good.
5. BatchNorm handling: with small finetune batches or domain shift, keep BN layers in `eval()` (frozen running stats) while training everything else — updating BN stats on 8-image batches from a new domain is a top cause of "finetune worked, deployment doesn't."
6. Input resolution: finetune at (or step up to) the deployment resolution. Evaluating at a different resolution than training costs accuracy; if you must infer at higher res, do a short high-res finetune at the end (cheap, effective).

## Task → architecture family

- **Classification**: any modern backbone + pooled head. Only decisions: backbone size vs latency, resolution.
- **Detection**: one-stage (YOLO/RetinaNet-family: anchor or anchor-free, fastest, default for real-time), two-stage (Faster R-CNN family: better on small/crowded objects, slower), DETR-family (no NMS/anchor tuning, clean pipeline, strong with good pretraining but slower to converge; small objects historically its weak spot — check the variant). Choose by latency budget and small-object share.
- **Semantic segmentation**: encoder-decoder (U-Net family — the default, especially medical/industrial), or DeepLab/ASPP-style for large-context scenes. U-Net + pretrained encoder (`segmentation_models_pytorch`) covers 90% of applied cases.
- **Instance segmentation**: Mask R-CNN family or query-based (Mask2Former-style). Prompted/interactive mask generation (SAM-style) is a labeling accelerator and zero-shot tool, not a replacement for a trained closed-vocabulary model when classes are fixed.
- Don't use detection when classification suffices ("is there any defect in the image?" is classification; "where are the defects?" is detection) — detection needs box labels, which cost 5–10× per image.
- Segmentation labels are the most expensive; consider weak supervision (boxes → masks via SAM-style tools, then human-verify) before commissioning pixel annotation.

## Resolution / batch / memory arithmetic

- Activation memory scales ∝ H×W×C summed over layers; doubling resolution ≈ 4× activation memory and ≈ 4× FLOPs for CNNs (ViT: 4× tokens → 16× attention FLOPs term, 4× the rest). Params are unchanged — memory blowups at high res are activations, so gradient checkpointing helps, weight quantization doesn't.
- Batch-size math: memory ≈ fixed(weights+optimizer) + batch × per-sample activations. If batch 16 at 512² fits, batch 16 at 1024² needs ~4× activation memory → drop to ~4, then compensate with gradient accumulation (fine for most CV losses; NOT equivalent for within-batch losses — BN stats, contrastive negatives).
- Small objects are a resolution problem before a model problem: an object 15 px wide at native res becomes ~7 px after resize-to-512 and dies in the downsampling stack. Compute object-size-in-pixels *after* your resize before choosing architecture; consider tiling (SAHI-style sliced inference) over resolution increase for large images with small objects.
- Latency: FLOPs ≠ speed. Depthwise convs are FLOPs-cheap but bandwidth-heavy; measure on target hardware with the real input size, batch 1 if that's deployment reality.

## Evaluation traps

- **mAP subtleties**: COCO mAP averages over IoU 0.5:0.95 — a model can gain mAP by better box tightness with zero new detections found, or vice versa. Report AP50 and AP75 separately, plus per-class AP and small/medium/large breakdown; a single mAP number hides "great on cars, useless on pedestrians." Confidence-threshold-free metrics (mAP) don't tell you deployment behavior — you ship at one threshold; report precision/recall at the operating point too.
- **NMS/threshold mismatch between eval and deployment**: eval at score-threshold 0.001 with 100 detections/image (COCO protocol) vastly overstates what your production 0.5-threshold pipeline sees.
- **Pixel accuracy on imbalanced segmentation is meaningless**: 98% background → all-background predicts 98% accuracy. Use per-class IoU and mean IoU; for tiny structures (vessels, cracks, tumors) also Dice and boundary metrics — a 2-px systematic boundary erosion barely dents IoU of big regions but is fatal for thin structures.
- **Class-imbalanced classification**: accuracy lies; use per-class recall/precision, balanced accuracy, PR-AUC (not ROC-AUC, which flatters heavy imbalance).
- **Data leakage patterns specific to CV**: near-duplicate frames from the same video split across train/val; multiple crops of the same patient/scene/device in both splits; augmented copies leaking into val. Split by *group* (patient, video, capture session), never by image. This is the #1 cause of "95% in validation, 70% in production."
- Test-time augmentation (TTA) inflates reported numbers vs deployment unless deployment also runs TTA — label which you're reporting.

## Classical CV: when it wins

- **Homography/geometry**: planar object pose, document rectification, camera-to-world mapping with a calibrated rig, image stitching → `cv2.findHomography` with RANSAC on ORB/SIFT matches. Deterministic, sub-pixel accurate, no training data. A network estimating what a homography computes exactly is malpractice.
- **Filtering/thresholding**: controlled-lighting industrial inspection — background subtraction, adaptive threshold (`cv2.adaptiveThreshold`), blob analysis (`cv2.connectedComponentsWithStats`) solve many "detect the part / measure the gap" problems at 1000+ fps on CPU.
- **Morphology**: `cv2.morphologyEx` open/close to denoise masks, skeletonize for thin structures, watershed for touching-object separation — also the right *post-processing* for neural segmentation outputs (learn the mask with a net, clean it with morphology).
- Hybrid pattern that wins in industry: classical pipeline for localization/rectification (fast, reliable) → small CNN for the genuinely semantic decision on the rectified crop.
- Choose classical when: variability is geometric/photometric and modelable, tolerances are measurable in pixels, data collection is expensive, or certification demands explainability. Choose learned when variability is semantic (breed, defect *type*, occlusion, clutter).

## Verification / self-check

1. Augmentation review: for each transform, name the invariance it assumes and confirm the label survives for this task; confirm masks/boxes are co-transformed in the same call.
2. Pipeline probe: visualize ~20 augmented (image, label) pairs before training — most label-destruction bugs are visible instantly and invisible in metrics until too late.
3. Splits: confirm grouping (patient/video/session) and dedup before believing any metric.
4. Metrics: does the metric match deployment (operating threshold, class balance, boundary sensitivity)? Report per-class, not just the mean.
5. Arithmetic: recompute activation memory and object-size-in-pixels for the proposed resolution; confirm the small-object story survives the resize.
6. Baseline order: linear probe → finetuned backbone → (only then) custom architecture. If a proposal skips to step 3, push back.
