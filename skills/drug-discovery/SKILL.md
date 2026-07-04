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

## Worked micro-examples

**1. Cheng-Prusoff.** A competitive inhibitor gives IC50 = 100 nM in an assay run at [S] = 4·Km. True Ki = 100/(1 + 4) = 20 nM. If a second lab runs the same compound at [S] = Km, they'd measure IC50 = Ki·(1+1) = 40 nM — a 2× "discrepancy" that is really perfect agreement. Always normalize to Ki.

**2. LLE guiding a decision.** Analog A: IC50 = 100 nM (pIC50 = 7.0), logD = 4.5 → LLE = 2.5. Analog B: IC50 = 300 nM (pIC50 = 6.5), logD = 2.0 → LLE = 4.5. B is "less potent" but far better positioned: lower lipophilicity means better solubility, lower clearance and hERG risk, and the higher LLE says the binding is driven by specific interactions, not grease. Prefer B as the series lead.

**3. Half-life reasoning.** Two compounds both have t₁/₂ = 12 h. Compound X: CL low, Vd normal → clearance-limited, expect dose-proportional exposure, manageable. Compound Y: CL high, Vd very large → long t₁/₂ from deep tissue distribution; high total dose needed, potential tissue accumulation/tox. Same t₁/₂, different development risk.

**4. Magic methyl SAR.** Adding an ortho-methyl to a biaryl raises potency 20× and improves metabolic stability. Interpretation: the methyl enforces a twisted, bioactive conformation (restricting rotation to the bound geometry) and blocks a metabolic soft spot — a classic conformational + metabolic double win, not just added bulk.

## Verification and self-check

- **State the assay context** behind every potency number: biochemical vs cellular, [S]/Km, readout type. If you can't, don't compare it to another number.
- **Convert to Ki/Kd** before ranking competitive inhibitors; sanity-check IC50 ≥ Ki.
- **Compute LE and LLE**, not just potency, before calling an analog "better." LE = 1.37·pIC50/heavy-atom count (kcal/mol per heavy atom); LLE = pIC50 − logD.
- **Run the artifact checklist** (dose-response shape, Hill slope, detergent sensitivity, PAINS/reactive alerts, SAR presence) before trusting a hit.
- **Reason in free drug and exposure over time**, not single in-vitro numbers, for any in-vivo claim; check that free Cmax/Cmin brackets the target Ki.
- **Confirm selectivity breadth** and note species/translation caveats before asserting a compound is clean.
