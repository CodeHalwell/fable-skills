---
name: drug-discovery
description: Use for medicinal chemistry and pharmacology reasoning — target validation, SAR and matched molecular pairs, balancing potency/selectivity/ADMET, pharmacokinetics (bioavailability, clearance, half-life, first-pass), lead-optimization tradeoffs and ligand efficiency, structural alerts (PAINS, reactive groups), interpreting IC50/Ki/Kd and Cheng-Prusoff, and reading dose-response curves. Load when evaluating compounds, binding data, drug-likeness, or discovery-pipeline decisions.
---

# Drug discovery

## Core mental model

1. **The target matters more than the molecule.** Most clinical failures are efficacy failures — the target was wrong, not the compound weak. A validated target (genetic + pharmacological + disease-relevant tissue evidence) is worth more than a 10× potency gain against an unvalidated one. When asked to "improve a hit," first ask whether the target is validated and whether the assay reflects the disease biology.
2. **Optimization is multi-objective and simultaneous.** Potency, selectivity, and ADMET (absorption, distribution, metabolism, excretion, toxicity) are coupled constraints, not a sequence. Cranking potency by adding lipophilic bulk almost always degrades solubility, metabolic stability, and promiscuity. The best compound is rarely the most potent; it's the one that clears every constraint at once.
3. **SAR is a difference engine.** You learn from *pairs*, not absolutes. A matched molecular pair (one well-defined structural change) tells you what a specific interaction is worth. A methyl that adds 10× potency ("magic methyl") usually fills a hydrophobic pocket or enforces a bioactive conformation; a methyl that does nothing tells you that vector is empty.
4. **Free drug drives effect.** Only unbound drug at the target site acts. High plasma protein binding is not automatically bad, but exposure, dose, and PK/PD must be reasoned in terms of *free* concentration relative to the target's Ki/IC50 over time.
5. **In vitro potency is necessary, not sufficient.** A nanomolar biochemical IC50 means nothing if the compound can't cross membranes, is effluxed, is metabolized in minutes, or hits the target only because it aggregates or reacts covalently by accident.

## Decision frameworks

- **Is this a real hit?** Before optimizing, rule out artifacts: (a) does activity have a sensible dose-response (clean sigmoid, full curve, Hill slope near 1)? Hill slope ≫ 1 or steep/incomplete curves suggest aggregation or nonspecific effects. (b) Does adding detergent (e.g., 0.01% Triton) abolish activity? → colloidal aggregation. (c) Does the scaffold contain PAINS/reactive motifs? (d) Is there SAR at all, or is every analog equipotent (a red flag for nonspecific mechanism)?
- **Potency metric to demand:** for enzymes and competitive inhibitors, Ki (assay-independent) beats IC50 (depends on [S] and Km). Convert with Cheng-Prusoff. For receptor binding, Kd/Ki from saturation or competition. For functional cellular readouts, EC50 with efficacy (Emax). Never compare IC50s across assays with different substrate concentrations without correcting.
- **Which change to make in lead-op:** to raise potency without wrecking properties, prefer changes that add *specific polar contacts* (H-bond, salt bridge) over bulk lipophilic additions. Track ligand efficiency (LE) and lipophilic ligand efficiency (LLE) so you don't fool yourself with size- or grease-driven gains.
- **ADMET triage priorities:** if oral, check solubility and permeability (BCS class) and metabolic stability (microsomal/hepatocyte clearance) early; check hERG (cardiac), CYP inhibition (DDI), and reactive-metabolite alerts before committing a series.
- **Rule-of-5 as a soft prior, not law:** MW ≤ 500, logP ≤ 5, HBD ≤ 5, HBA ≤ 10 predict oral absorption liability, but many oral drugs violate one; beyond-Ro5 chemistry (macrocycles, PROTACs) is real. Use it to flag risk, not to reject.

## SAR reasoning in detail

- **What a matched pair probes:** each single change interrogates a specific hypothesis about the binding site. A **methyl** tests a small hydrophobic pocket or a conformational lock (rotation restriction). A **halogen** (F, Cl) tests a small lipophilic/electronic pocket; **fluorine** also blocks metabolism at that position and tunes pKa without adding much size. A **ring nitrogen swap** (benzene → pyridine) tests an H-bond acceptor need and lowers logP/improves solubility. **Removing an H-bond donor/acceptor** tests whether a specific polar contact is real (big loss = real contact; no change = solvent-exposed).
- **Interpret potency *cliffs*.** A tiny change causing a large potency swing ("activity cliff") signals a specific, geometry-sensitive interaction — high-value information about the pharmacophore. Flat SAR (nothing you do changes potency) is a warning of a nonspecific mechanism (aggregation, denaturation), not a robust scaffold.
- **Bioisosteres** replace a group with one of similar properties but better ADMET (e.g., carboxylic acid → tetrazole/acylsulfonamide to keep acidity while changing permeability/metabolism; amide → oxadiazole to block hydrolysis). Use them to fix a liability while preserving the key interaction.

## Failure modes and pitfalls

- **Treating IC50 as an intrinsic constant.** IC50 for a competitive inhibitor shifts with substrate concentration: Ki = IC50/(1 + [S]/Km). Two labs reporting different IC50s for the same compound may fully agree on Ki. Always ask what [S]/Km the IC50 was measured at.
- **Comparing potencies across modalities.** IC50 (inhibition), EC50 (functional response), Kd (binding affinity), and Ki (inhibition constant) are not interchangeable. A tight binder (low Kd) can be a weak functional antagonist and vice versa.
- **Chasing potency with lipophilicity.** Adding a phenyl or long alkyl often boosts biochemical potency by burying hydrophobic surface — but tanks solubility, raises metabolic clearance and hERG risk, and inflates promiscuity. LLE = pIC50 − logP (or logD) catches this: if potency rises only because logP rose, LLE is flat and you've made no real progress. Aim to *increase* LLE (good drugs often LLE > 5).
- **Ignoring aggregation and assay interference.** A large fraction of screening hits are colloidal aggregators that nonspecifically inhibit; flat SAR, steep Hill slopes, and detergent-sensitivity are the tells. PAINS (e.g., rhodanines, catechols, quinones, toxoflavins) hit many targets via reactivity, redox cycling, or fluorescence — not by real binding.
- **Overlooking reactive/covalent liabilities.** Michael acceptors, epoxides, aldehydes, anilines/nitroaromatics (→ reactive metabolites), and thiophenes/furans (bioactivation) flag idiosyncratic toxicity. Covalent inhibitors can be excellent drugs *by design*, but accidental reactivity is a liability.
- **Confusing half-life determinants.** t₁/₂ = 0.693·Vd/CL. A long half-life can come from low clearance OR high volume of distribution — they have opposite implications for dosing and tissue exposure. Don't attribute a long t₁/₂ solely to "metabolic stability."
- **Forgetting first-pass metabolism.** Oral bioavailability F = F_abs × F_gut × F_hepatic. A compound can be fully absorbed yet have low F because the liver extracts most of it on first pass (high hepatic extraction ratio). IV potency can't be extrapolated to oral dosing without PK.
- **Reading an incomplete dose-response.** An IC50 quoted from a curve that never reaches a plateau (no defined top/bottom) is an extrapolation, not a measurement. Demand a full curve spanning the inflection with defined asymptotes; report the Hill slope.
- **Selectivity measured against too few off-targets.** "Selective" needs a counter-screen panel (related kinases/receptors, hERG, CYPs, safety panel). Selectivity vs one paralog is weak evidence.
- **Species differences.** Rodent PK/metabolism/potency need not translate to human; CYP isoform differences and target sequence differences bite in translation.
- **Static thinking about exposure.** Efficacy needs free drug above the target Ki for the relevant fraction of the dosing interval (PK/PD), not just a low in-vitro IC50. A potent, rapidly cleared compound fails in vivo.
- **Misreading a steep or shallow Hill slope.** A dose-response Hill slope near 1 is expected for simple 1:1 binding. A very steep slope (≫1) suggests cooperativity or, more often in screening, an artifact (aggregation, precipitation). A shallow slope may indicate multiple binding sites or nonspecific effects. Don't just extract IC50; read the slope.
- **Ki vs Kd conflation.** Kd is the equilibrium dissociation constant for a binding interaction (thermodynamic affinity). Ki is the inhibition constant from a functional/competition assay. For a simple competitive inhibitor they coincide, but for allosteric or slow-off-rate compounds they diverge — and residence time (koff) can matter more for in-vivo efficacy than equilibrium affinity.
- **Ignoring the therapeutic index.** Potency at the target is meaningless without the margin to toxicity. TI = toxic dose / effective dose. A modestly potent compound with a wide window beats a super-potent one with a narrow window.
- **Confusing efflux/permeability with intrinsic potency.** A cellular EC50 far worse than the biochemical IC50 usually means a permeability or efflux problem (e.g., P-gp substrate), not weak target engagement. The fix is chemistry on physicochemical properties, not more potency.
- **Over-relying on a single docking/virtual-screen score.** Docking scores are weakly correlated with real affinity and cannot rank close analogs reliably. Treat in-silico hits as hypotheses to test, expecting most to fail on assay interference, aggregation, or poor properties.

## Pharmacokinetics essentials

- **The core relationships:** CL (clearance, volume cleared per time), Vd (apparent volume of distribution), t₁/₂ = 0.693·Vd/CL, and at steady state Css = (F·Dose/τ)/CL. Exposure (AUC) = F·Dose/CL. These four let you reason about dosing without a simulator.
- **Bioavailability F** = fraction of oral dose reaching systemic circulation intact = F_absorption × (1 − gut extraction) × (1 − hepatic extraction). A high hepatic extraction ratio caps oral F regardless of perfect absorption.
- **Clearance mechanism sets DDI/variability risk:** renal clearance is generally cleaner; hepatic CYP-mediated clearance brings drug-drug interactions and pharmacogenomic variability (CYP2D6, CYP3A4). Know which route dominates.

## Worked micro-examples

A strong model runs all of these cold — Cheng-Prusoff (IC50 100 nM at [S]=4Km → Ki 20 nM); LLE picking the "less potent" low-logD analog as the better lead; same t½ = 12 h splitting into low-CL vs high-Vd risk profiles; magic-methyl as conformational-lock + metabolic-block; oral F = 0.9 × (1−0.7) = 0.27; and the steep-Hill/flat-SAR/detergent-sensitive aggregator verdict. The retained value is the *reflex to run the artifact checklist before optimizing* and to *convert every potency to Ki and pair it with LE/LLE before calling an analog "better"* — the discipline, since the arithmetic and interpretations are not the gap.

## Verification and self-check

- **State the assay context** behind every potency number: biochemical vs cellular, [S]/Km, readout type. If you can't, don't compare it to another number.
- **Convert to Ki/Kd** before ranking competitive inhibitors; sanity-check IC50 ≥ Ki.
- **Compute LE and LLE**, not just potency, before calling an analog "better." LE = 1.37·pIC50/heavy-atom count (kcal/mol per heavy atom); LLE = pIC50 − logD.
- **Run the artifact checklist** (dose-response shape, Hill slope, detergent sensitivity, PAINS/reactive alerts, SAR presence) before trusting a hit.
- **Reason in free drug and exposure over time**, not single in-vitro numbers, for any in-vivo claim; check that free Cmax/Cmin brackets the target Ki.
- **Confirm selectivity breadth** and note species/translation caveats before asserting a compound is clean.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 10 claims: ~10 baseline. Opus 4.8 cold produced Cheng-Prusoff, the aggregator triage (steep Hill + flat SAR + detergent sensitivity), LE/LLE formulas and the prefer-higher-LLE logic, t½ = 0.693·Vd/CL decomposition, F = fabs·(1−EH), magic-methyl mechanisms, the IC50/EC50/Kd/Ki distinctions with receptor-reserve nuance, Ro5-as-soft-filter, and a broad bioisostere list (it exceeded the skill on carboxylic-acid replacements).
- No knowledge delta. The skill functions as a triage discipline: run the artifact checklist before starting chemistry, normalize to Ki with assay context, and judge on LE/LLE plus free-drug exposure rather than raw potency.
- Worked examples compressed to the disciplines; the calculations were reproducible.
