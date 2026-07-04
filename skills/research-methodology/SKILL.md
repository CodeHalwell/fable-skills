---
name: research-methodology
description: Use for conducting rigorous research — formulating answerable questions before choosing methods, literature-review strategy (citation chasing, reviews as maps, primary-source verification), identifying genuine novelty, reproducibility hygiene, distinguishing correlation-mining from hypothesis testing, evaluating source credibility, judging whether a question is already answered or open, and guarding against confirmation bias. Load when planning a study, doing a lit review, or assessing whether a claim is established.
---

# Research methodology

## Core mental model

1. **A question must be answerable before it is worth asking.** The answerable-question test: can you name, in advance, the observation or result that would answer it and the observation that would refute your expected answer? "Does X help?" fails; "Does X reduce Y by ≥Δ in population Z under condition C, versus control W?" passes. Method selection is downstream of a well-formed question — never pick a method first and hunt for a question it fits.
2. **The literature is a graph, traverse it both directions.** Start from a recent authoritative review (a map of the field), then chase citations *backward* (what this work builds on) and *forward* (who cited it, what superseded it — use citation indices). One well-chosen review plus two hops of citation chasing usually surfaces the real state of the art faster than keyword search.
3. **Never trust an abstract's framing.** Abstracts overstate, omit caveats, and describe the study the authors wish they ran. Verify every load-bearing claim against the actual results — the figure, the table, the effect size, the n, the confidence interval. If you cite it, you read the methods and results, not the abstract.
4. **Novelty is a claim that must be located.** The contribution is the *delta* over prior work, stated precisely: new method, new result, new domain, new refutation. If you cannot articulate what specifically was not known before, you have not found the contribution — and you may be reinventing something already published.
5. **Distinguish generating hypotheses from testing them.** Mining a dataset for whatever correlates ("what predicts Y?") is exploratory and produces hypotheses, not confirmed findings — the same data cannot both generate and confirm. Confirmation requires a pre-specified hypothesis tested on fresh data.

## Decision frameworks

- **Is this question already answered or genuinely open?** Search for it explicitly. If multiple independent groups converge on an answer with consistent effect sizes → settled; build on it, don't re-litigate. If the "consensus" traces to a single primary source everyone cites → treat as open and verify that source. Contradictory results across strong studies → open, and the interesting question is *why they differ* (population, method, moderator).
- **How to weight a source:** venue peer-review rigor is a weak prior, not a guarantee; weight primary methodology over reputation. Rank by: appropriate design for the claim > pre-registration/data availability > sample size and power > independence from conflicts > replication by others. A large pre-registered study in a modest venue beats a small flashy one in a top venue.
- **When to read primary vs review:** use reviews to orient and find primary sources; use primary sources for any claim you will rely on or cite. Meta-analyses beat individual studies for effect-size estimates *if* heterogeneity and publication bias are addressed.
- **Correlation-mining vs hypothesis test:** if the analysis chose which variables/comparisons to report after seeing the data, treat conclusions as tentative and demand out-of-sample confirmation. If the hypothesis, primary outcome, and analysis were fixed in advance (ideally pre-registered), the test is confirmatory.
- **Confirmation-bias guard:** adopt a pre-registration mindset even informally — before looking, write down what you predict, what would change your mind, and how you'll analyze. Seek the strongest disconfirming evidence, not more confirming examples.

## Literature review as a systematic process

- **Build a coverage map, not a pile.** Organize sources by the question they answer and the method they use, so gaps become visible. If every source uses the same method, the literature may be method-bound and the phenomenon under-tested.
- **Read in a triage order:** title → abstract → figures/tables → methods → full text. Most papers you touch stop at the figures (enough to decide relevance and extract the headline result). Reserve full reads for the few you'll build on or cite substantively.
- **Track provenance as you go.** For each claim you'll use, record the primary source, the exact result (number, n, interval), and how strong the design is. This prevents citation laundering later and makes the writeup's related-work section fall out for free.
- **Recency and supersession.** A foundational paper may be canonical but partly superseded; a forward-citation search surfaces the correction or improvement. Always check whether the "known result" still stands.

## Reproducibility and research-debt hygiene

- **Decision log.** Keep a running record of choices (which dataset, which exclusion rule, which parameter and why). Six months on, "why did we drop those samples?" must be answerable. Undocumented decisions are debt that compounds.
- **Runnable pipeline.** Prefer a script/notebook that regenerates every figure from raw data over hand-edited artifacts. If a reviewer or future-you can't rerun it end to end, the result is fragile.
- **Version and seed capture.** Record software versions, random seeds, and data snapshots/hashes. "It worked last month" without these is not reproducible.
- **Lab-notebook discipline** (wet or dry): timestamped, append-only, enough that someone else could continue your work. Retrofitting a notebook after the fact loses exactly the details that mattered.

## Failure modes and pitfalls

- **Citing the abstract's claim, not the finding.** Abstracts routinely say "X improves Y" when the table shows a nonsignificant trend, a subgroup-only effect, or a different endpoint. Always locate the specific number and its uncertainty.
- **Citation laundering / broken telephone.** A claim gets attributed to a paper that merely cited it, and the original said something weaker. Trace claims to the primary source; when a review states a "fact," follow its citation and confirm the original supports it.
- **Mistaking a single source for consensus.** Ten papers repeating one uncontrolled 1990s study is not ten pieces of evidence. Check whether citations are independent replications or echoes.
- **HARKing (hypothesizing after results are known).** Presenting a post-hoc pattern as if it were the a priori hypothesis. This inflates false positives and is undetectable in the writeup unless pre-registration exists. In your own work, log hypotheses before analysis.
- **p-hacking and the garden of forking paths.** Trying many analyses, outcomes, subgroups, or covariate sets and reporting the significant one. Even without intent, flexible analysis manufactures significance. Fix the analysis plan first; report all comparisons attempted.
- **Ignoring publication bias.** The literature is a biased sample — null results are underpublished, so meta-estimates skew high. Funnel-plot asymmetry and "too many just-significant p-values" are warning signs.
- **Underpowered studies believed because they're significant.** A significant result from a tiny study, in a low-prior field, is more likely a false positive than a real effect (low positive predictive value). Small n + surprising claim = distrust.
- **Reproducibility debt.** Not recording seeds, versions, exact commands, data provenance, and the decisions taken along the way. Six months later you cannot reproduce your own result. Keep a decision log and a runnable pipeline from raw data to figure.
- **Confirmation bias in the search itself.** Searching only for support ("evidence that X works") returns support. Symmetrically search for refutation ("X fails / X harms / limitations of X").
- **Conflating statistical significance with importance,** and both with truth. Significance depends on n; importance depends on effect size and context; truth depends on design and replication.
- **Treating "no evidence of effect" as "evidence of no effect."** A null result in an underpowered study establishes nothing. Check the confidence interval — does it exclude effects you'd care about?
- **Novelty inflation.** Framing an incremental result as a paradigm shift, or missing that the "novel" idea was published years earlier under different terminology. Search synonyms and adjacent fields.
- **Base-rate neglect in interpreting findings.** In a field where most hypotheses are false, even a well-powered significant result has a substantial chance of being a false positive. The prior plausibility of the hypothesis matters as much as the p-value. Extraordinary claims need extraordinary (and replicated) evidence.
- **Cherry-picking a favorable subset of the literature.** Citing the three studies that agree with you and ignoring the five that don't. A fair review weighs the full body of evidence, including inconvenient results, and explains discrepancies rather than hiding them.
- **Conflating a preprint or press release with a peer-reviewed, replicated finding.** Different evidentiary weight. Note the stage of vetting when relying on a source.
- **Ecological / level-of-analysis fallacy.** Inferring individual-level relationships from group-level (aggregate) correlations, or vice versa. The relationship at one level need not hold at another.
- **Assuming a method is valid because it's standard.** Widely used ≠ appropriate for your question. Check that the method's assumptions actually hold in your setting.

## Worked micro-examples

**1. Turning a vague question answerable.** Vague: "Is intermittent fasting good for health?" Answerable: "In adults with obesity, does 16:8 time-restricted eating for 12 weeks reduce body weight by a clinically meaningful ≥3 kg versus an isocaloric standard-eating control, in a randomized trial?" Now the design, control, outcome, and refutation criterion are all specified — and you can search whether it's already answered.

**2. Citation chasing in practice.** You need the state of the art on method M. Step 1: find a 2023 review of M's field → extract the 5 foundational primary papers and the current leading methods. Step 2 (backward): read the foundational papers to understand assumptions. Step 3 (forward): in a citation index, list recent papers citing the leading method → find the critique or successor the review predates. Result: you know both the canonical result and what has since challenged it — impossible from keyword search alone.

**3. Detecting HARKing/p-hacking in a paper.** A study reports a significant effect in "women over 50 with high baseline X" — a specific subgroup not mentioned in the aims. No pre-registration. No correction for the many subgroups implicitly tested. Verdict: exploratory at best; treat as a hypothesis to test elsewhere, not a finding.

**4. Weighing conflicting studies.** Study A (n=40, single site, not pre-registered, in a high-prestige venue) finds a large effect; Study B (n=2,000, pre-registered, multi-site, modest venue) finds a null. The naive move is to trust the prestigious A. The disciplined move: B's design is far stronger (power, pre-registration, replication across sites), so weight B and treat A as likely a false positive or overestimate driven by small-sample variance and publication incentives. The open question becomes whether a moderator explains A's context.

**5. Is it answered or open?** "Does vitamin D supplementation prevent respiratory infections?" A search reveals dozens of RCTs and several meta-analyses with heterogeneous results and a small, dose- and baseline-status-dependent effect. Conclusion: not a clean open question and not fully settled — the productive framing is the *moderator* question (in whom, at what dose, at what baseline level), not the blanket yes/no.

## Verification and self-check

- **Every cited claim traced to primary source**, with the actual effect size, n, and interval read from results — not the abstract.
- **Consensus checked for independence** — is it many independent results or one source echoed?
- **Question passes the answerable test** and you've searched whether it's already answered.
- **Analysis classified** as exploratory (hypothesis-generating) or confirmatory (pre-specified) and conclusions caveated accordingly.
- **Disconfirming search done** — you actively looked for refutation, limitations, and failed replications, not just support.
- **Reproducibility artifacts exist** — seeds, versions, data provenance, decision log — such that another person could regenerate the result.
- **Source credibility weighted** by design and power over venue prestige; conflicts of interest noted.
