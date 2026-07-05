---
name: scientific-writing
description: Use for writing papers and technical reports — structuring by information hierarchy (title/abstract/figures carry the story), stating the contribution, making figures self-contained primary artifacts, auditing claims against evidence, meeting reproducibility standards in methods, positioning related work fairly, writing abstracts, revising structure before prose, and responding to reviews. Load when drafting, revising, or reviewing a scientific manuscript or technical report.
---

# Scientific writing

## Core mental model

1. **Most readers stop early — write in layers of decreasing readership.** Title (everyone) → abstract (many) → figures + captions (skimmers) → introduction and conclusion (some) → full methods and results (few). The paper's core message must survive each truncation: someone who reads only the title and figures should still get the main result. Front-load; never bury the finding in the discussion.
2. **The contribution statement is the spine.** You must be able to state, in one or two sentences, what is new and why it matters — the delta over prior work. If you can't, no amount of polish helps. Everything in the paper either supports the contribution or is cut. Write it first; check every section against it.
3. **Figures are the primary artifact, not decoration.** Reviewers and readers reason from figures. Each figure must be self-contained: a caption that states what it shows and the takeaway, axes labeled with units, error bars present with their definition stated (SD? SEM? 95% CI?), no chartjunk. A reader should understand a figure without the body text.
4. **Every claim must trace to evidence.** Overclaiming is the top reviewer complaint. Each assertion in the abstract and discussion must map to a specific result (figure/table/statistic). Claims the data don't support get softened or removed. Calibrate verbs: "demonstrates" vs "suggests" vs "is consistent with" are different evidentiary commitments.
5. **Methods are a reproducibility contract.** Enough detail that a competent peer could rerun the study and get the same result: exact versions, parameters, seeds, data sources, sample sizes, statistical tests and their assumptions. If it affects the result, it goes in methods (or supplement).

## Decision frameworks

- **Abstract formula:** (1) context — the field and why it matters, one sentence; (2) gap — the specific unsolved problem/limitation; (3) approach — what you did; (4) result — the headline finding *with numbers* (effect size, not "significantly improved"); (5) implication — what it changes. Cut everything else. A quantified result sentence is non-negotiable — "improves performance" is worthless; "improves accuracy from 82% to 89%" is a finding.
- **Revision order — structure before prose:** fix the argument skeleton (does each section earn its place? is the claim-evidence chain complete? are figures in the right order telling the story?) *before* line-editing sentences. Polishing prose in a paragraph you'll delete is wasted effort. Passes: (1) contribution + outline, (2) figures + captions tell the story alone, (3) section-level argument, (4) paragraph topic sentences, (5) sentence-level clarity, last.
- **What goes in a figure vs table vs text:** trends, comparisons, distributions → figure. Exact numbers readers will look up → table. Single key statistics → inline text. Never make readers extract a trend from a table or a precise value from a plot.
- **Related work without strawmanning:** describe prior work as its authors would recognize it, then state the specific limitation your work addresses. "Prior methods ignore X" is usually false and invites a hostile review; "prior methods address X under assumption A; we relax A" is defensible and precise.
- **Claim calibration ladder:** reserve "prove/demonstrate" for direct, controlled evidence; use "show/find" for solid empirical results; "suggest/indicate" for correlational or limited evidence; "may/could" for speculation. Match the verb to the strength of the design.

## Structuring the argument

- **The introduction is a funnel with a promise.** Broad importance → the specific gap → what you do → (often) a one-sentence result preview. By the end of the intro the reader must know exactly what problem you solve and what you found. Don't make them wait for the discussion.
- **Each section answers one question.** Intro: why care and what's the gap? Methods: what did you do (reproducibly)? Results: what happened (facts, minimal interpretation)? Discussion: what does it mean, what are the limits, what's next? Keep interpretation out of results and raw results out of discussion.
- **Results present, discussion interprets.** A frequent structural error is editorializing in results ("impressively, the method excels") or re-listing numbers in discussion. Results = observations; discussion = meaning, mechanism, limitations.
- **Topic sentences carry the skim.** A reader should get the argument from the first sentence of each paragraph alone. Write topic sentences that state conclusions, not topics ("Method C outperforms baselines on noisy data" not "We now discuss noise").

## Figures done right

- **One message per figure.** Each figure makes a single point; if it makes three, split it. The caption's first sentence is that point.
- **Encode honestly.** Bar charts start the y-axis at zero; line charts may not need to but must not exaggerate; use position/length (most perceptually accurate) over area/angle/color intensity for quantitative comparisons. Avoid dual y-axes (they manufacture apparent correlation).
- **Accessibility and clarity.** Perceptually uniform, colorblind-safe palettes; redundant encoding (shape + color) so the figure survives grayscale; direct labels over legends where possible; large enough fonts to read at print size.
- **Show the data and its uncertainty.** Prefer showing distributions (points, box/violin) over bare means; always include error bars/intervals and define them and n in the caption.

## Failure modes and pitfalls

- **Burying the contribution.** The novel result appears on page 7 in the discussion. Move it to the abstract and introduction; state it explicitly ("Our contribution is…"). Readers should never have to reverse-engineer what's new.
- **Overclaiming beyond the evidence.** Causal language ("X causes Y") from correlational data; generalizing beyond the tested population/conditions; "state of the art" without the head-to-head comparison. Audit: highlight every claim, draw an arrow to its supporting result; unsupported claims get cut or hedged.
- **Figures that need the text to decode.** Unlabeled axes, cryptic legends, no units, missing error bars, or a caption that only says "Results for method X." Fix: caption states the comparison and the takeaway; every axis has label + unit; error bars present and defined.
- **Chartjunk and misleading visuals.** 3D bars, truncated y-axes that exaggerate differences, dual axes implying spurious correlation, rainbow colormaps that distort. Use honest baselines (usually y from zero for bar charts), perceptually uniform colormaps, and the simplest encoding that shows the effect.
- **Error bars missing or undefined.** A bar chart of means with no error bars is uninterpretable. Always show variability and *state in the caption what the bars represent* (SD, SEM, or CI — they differ by large factors) and n.
- **The abstract with no number.** "We propose a novel method that significantly improves results." Empty. State the headline metric and its value.
- **Methods too thin to reproduce.** Omitting hyperparameters, versions, preprocessing, exclusion criteria, or the exact statistical test. If a reader can't rerun it, it's not science yet — put the details in methods or a clearly referenced supplement.
- **Passive, hedged, nominalized prose that hides who did what.** "It was observed that an improvement was obtained" → "Method X improved accuracy by 7 points." Prefer active voice, concrete subjects, and verbs over nominalizations ("we measured" not "measurement was performed").
- **Related work as an annotated bibliography.** A flat list of "A did X. B did Y." with no synthesis and no positioning. Instead, group by approach, identify the axis your work advances, and place yourself on it.
- **Line-editing before structural editing.** Perfecting sentences in a section that survives structural review poorly. Structure first.
- **Inconsistent terminology.** Calling the same thing three names ("the model / our approach / the system"). Pick one term per concept and use it everywhere; consistency aids comprehension more than variety.
- **Discussion that restates results instead of interpreting them.** The discussion should say what the results *mean*, their limitations, and what they don't show — not replay the numbers.
- **Ignoring the limitations section, or making it perfunctory.** Honest limitations preempt reviewer objections and build credibility. State the real ones and their scope.
- **"Significant" used ambiguously.** In a paper "significant" reads as *statistically* significant. Don't use it to mean "large" or "important" — say "substantial" or "meaningful" and give the number. Report effect sizes and intervals, not just p-values.
- **Undefined notation and acronyms.** Every symbol defined at first use; acronyms spelled out once. A reader hitting an undefined term stops trusting the paper's care.
- **Figures and text disagreeing.** The number in the abstract differs from the table; the caption says "5 seeds" but methods say 3. These inconsistencies are what reviewers pounce on. Do a numbers-reconciliation pass.
- **Over-long, unfocused abstract.** Trying to summarize every result. The abstract sells one contribution and one headline number; details belong in the body.
- **Weak verbs and throat-clearing.** "It is important to note that", "In order to", "Due to the fact that" — cut them. Every sentence should advance the argument.

## Responding to reviews

- **Answer every point, in order, quoting the reviewer.** A point-by-point response letter that restates each comment then gives the change (with the exact new location/result) is far more persuasive than prose.
- **Concede fair points and fix them; push back only with evidence.** "The reviewer is wrong" loses; "We initially thought so too, but experiment X (new Fig. 5) shows Y" wins. Disagree respectfully and with data.
- **Distinguish changes made from changes declined,** and justify declines by scope or evidence, not by dismissing the reviewer. Reviewers are also readers — if one misread something, others will too, so clarify the text rather than just defending it.

## Worked micro-examples

The transformations here (weak abstract → context/gap/approach/quantified-result/implication; bare caption → takeaway + defined error bars + n; "robust to noise" → bounded "within 2 points up to 20% label noise, untested beyond"; defensive review reply → concede + show the exact new result) are ones a strong model produces cold. Retained only as the concrete *bar* to hit, not new knowledge: the non-negotiable is that the abstract's result sentence and every figure caption carry a **number and a comparison**, and every claim names the specific figure/table it traces to.

## Verification and self-check

- **Title + abstract + figures alone convey the full story** — hand them to someone and check they get the contribution and the headline result.
- **Contribution statement exists in one/two sentences** and the abstract and intro state it explicitly.
- **Claim-evidence audit done:** every abstract/discussion claim maps to a specific result; verbs match evidence strength; no causal language from correlational data.
- **Every figure is self-contained:** labeled axes with units, defined error bars with n, informative caption stating the takeaway, no chartjunk, honest axes.
- **Abstract contains a quantified result**, not just "improves."
- **Methods pass the reproduce test:** versions, parameters, seeds, sample sizes, statistical tests and assumptions all present.
- **Related work positions fairly** without strawmanning; terminology is consistent throughout.
- **Revision proceeded structure-first**, prose last.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 10 claims: ~10 baseline. Opus 4.8 cold produced the full abstract formula, self-contained-figure requirements, structure-before-prose revision ordering, the demonstrates/suggests/consistent-with calibration ladder, SD-vs-SEM-vs-CI caption rules, figure/table/text allocation, the point-by-point review-response format, and a thorough reviewer-pitfalls list.
- No knowledge gap surfaced; the skill functions as a discipline checklist. The one lever worth keeping sharp is the *audit action* — highlight every abstract/discussion claim and draw an arrow to the supporting figure/number — which is a procedure to run, not a fact to know.
- Worked examples compressed; the before/after rewrites were reproducible.
