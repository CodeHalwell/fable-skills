---
name: organic-chemistry-reasoning
description: Load when predicting organic reaction products or mechanisms, choosing between SN1/SN2/E1/E2, drawing electron-pushing arrows, assigning stereochemical outcomes, planning multi-step synthesis or retrosynthesis, selecting protecting groups, or checking protonation states — any mechanistic organic chemistry question.
---

# Organic Chemistry Reasoning

## Core mental model

1. **Every mechanism is "nucleophile attacks electrophile."** Full arrows move electron *pairs*, always from electron-rich (lone pair, π bond, σ bond) to electron-poor (empty orbital, δ+ carbon, H on a strong acid). Before predicting any product: label the best nucleophile and best electrophile in the flask. If a proposed step has an arrow starting at an atom with no available pair, or two arrows colliding into a carbon that would exceed 4 bonds, the mechanism is wrong.
2. **Stability of intermediates decides pathways.** Carbocation stability: benzylic/allylic (resonance) ≥ 3° > 2° >> 1° ≈ methyl ≈ vinyl/aryl (never form the last three in solution). Carbanion/conjugate-base stability is the reverse for alkyl, and is governed by: charge on more electronegative atom > resonance delocalization > induction (distance-attenuated: each intervening CH₂ cuts the effect roughly in half) > hybridization (sp > sp² > sp³, hence terminal alkyne pKa ≈ 25). Radical stability parallels carbocations.
3. **pKa is the universal ruler.** Memorized anchors let you settle nearly any acid–base or "will this deprotonate that" question: H₃O⁺ ≈ −1.7, carboxylic acid ≈ 4–5, ammonium ≈ 9–10, phenol ≈ 10, water 15.7, alcohol 16–18, terminal alkyne 25, H₂ ~36, amine N–H ≈ 38 (LDA territory), alkane ≈ 50; α-C–H of aldehyde/ketone ≈ 17–20, ester ≈ 25, 1,3-dicarbonyl ≈ 9–13. Equilibrium lies toward the weaker acid (higher pKa); a base quantitatively deprotonates when ΔpKa ≥ ~2–3.
4. **Mechanism dictates stereochemistry.** SN2: clean inversion at the electrophilic carbon (Walden). SN1: racemization (often partial, with slight inversion excess from ion pairing). E2: anti-periplanar H and leaving group required — this constraint, applied on a chair or Newman projection, *selects the product*. Syn-additions (H₂/Pd, hydroboration, OsO₄, m-CPBA epoxidation face) vs anti-additions (Br₂, halohydrin, epoxide opening) must be tracked, not decorated afterward.
5. **Retrosynthesis is disconnection at strategic bonds.** Work backward: disconnect C–C bonds α or β to functional groups (that's where reliable forward reactions form them), convert fragments to synthons (idealized cation/anion), then map synthons to real reagents (acyl anion ⇒ umpolung: dithiane, cyanide, or reverse the polarity of the disconnection). Prefer disconnections that split the target near the middle (convergent) and that remove stereocenters via reliable stereospecific steps.

## Decision frameworks

**SN1/SN2/E1/E2 — apply in this order:**

1. **Substrate at the leaving-group carbon:** methyl/1° → SN2 (E2 only with bulky base: t-BuO⁻, LDA); 3° → never SN2; benzylic/allylic can go either. Vinyl/aryl halides: none of the four (no backside approach, terrible cation) — think elimination-addition/benzyne or metal catalysis instead.
2. **Nucleophile/base character:** strong Nu, weak base (I⁻, Br⁻, RS⁻, N₃⁻, CN⁻, PR₃) → substitution. Strong bulky base (t-BuO⁻, DBU, LDA) → E2. Strong small base/Nu (HO⁻, MeO⁻, EtO⁻) → SN2 on 1°, E2 on 3°, mixture on 2° (heat pushes E2). Weak Nu/weak base (H₂O, ROH, RCO₂H as solvent) → SN1/E1 on 3°/2°-benzylic (heat favors E1).
3. **Solvent:** polar aprotic (DMSO, DMF, acetone, MeCN) accelerates SN2 (naked anion); polar protic (H₂O, ROH) stabilizes ions → favors SN1/E1 and *reverses halide nucleophilicity* (protic: I⁻ > Br⁻ > Cl⁻; aprotic: Cl⁻ > Br⁻ > I⁻).
4. **Temperature:** heat favors elimination (ΔS: one molecule → two/three). "Heated with conc. base" on a 2° substrate = E2, full stop.

**2° substrate tiebreak (most-missed case):** 2° + strong small base → E2-major with SN2 minor; 2° + good Nu/weak base in aprotic solvent → SN2; 2° + weak Nu protic solvent + heat → SN1/E1 mixture with rearrangement risk.

**Electrophilic addition regio/stereo cheat table:**

| Reagent on alkene | Regiochemistry | Stereochemistry | Trap |
|---|---|---|---|
| HX | Markovnikov (H to less-subst. C ⇒ cation on more-subst. C) | racemic at new center | Rearrangements! |
| HBr + peroxides (ROOR) | anti-Markovnikov | mixture | Radical chain — HBr only, not HCl/HI |
| H₂O, H⁺ (or Hg(OAc)₂/NaBH₄) | Markovnikov | racemic (oxymercuration: no rearrangement) | Acid-cat. hydration rearranges |
| BH₃·THF then H₂O₂/HO⁻ | anti-Markovnikov | **syn** addition of H and OH | B goes to less hindered C |
| Br₂ (or Br₂/H₂O → bromohydrin) | (OH at more-subst. C for halohydrin) | **anti** via bromonium | Meso vs d/l: cis-alkene + Br₂ → d/l pair; trans → meso (for symmetric alkenes) |
| OsO₄/NMO | — | **syn** diol | Contrast with anti diol from epoxide + H₂O |
| m-CPBA then H₃O⁺ | — | net **anti** diol | Epoxide opens with inversion at attacked C; acid: attack at more-subst. C; base: less-subst. C |

**Carbonyl reactivity ladder (toward nucleophiles):** acyl chloride > anhydride > aldehyde > ketone > ester ≈ carboxylic acid > amide > carboxylate. Nucleophilic acyl substitution runs downhill only (Cl → anything; amide → almost nothing without force). Grignards/organolithiums hit esters *twice* (→ 3° alcohol); to stop at ketone use Weinreb amide or add RLi to carboxylate.

**Protecting-group logic:** protect when a reagent would attack the wrong FG; choose orthogonality (removed by different conditions). Workhorses: ketone/aldehyde → acetal (stable to base, RMgX, LiAlH₄; off with aqueous acid); alcohol → TBS ether (off with F⁻/TBAF, stable to base and mild acid) or benzyl (off with H₂/Pd); amine → Boc (off with TFA) vs Cbz (H₂/Pd) vs Fmoc (piperidine — base!). Grignard formation or use in a molecule containing OH/NH/CO₂H/ketone → protect or the Grignard kills itself as a base. Every protection costs 2 steps; first check whether reordering the synthesis avoids it.

**Retrosynthesis procedure:** (1) Identify target FGs and the C skeleton; (2) look 1,2 / 1,3 / 1,5 relationships: 1,3-dioxygenated ⇒ aldol; 1,5-dicarbonyl ⇒ Michael; α,β-unsaturated carbonyl ⇒ aldol-condensation; 1,2-diol ⇒ alkene + OsO₄; β-hydroxy relation to a carbonyl ⇒ aldol; (3) disconnect C–C adjacent to the FG into acceptor synthon (C=O carbon, Michael β-carbon, R–X) + donor synthon (enolate, Grignard, cuprate, alkynide); (4) unnatural polarity (acyl anion, homoenolate) ⇒ umpolung equivalents (dithiane, CN⁻, nitroalkane); (5) recurse until fragments are ≤ ~5 carbons or purchasable.

## Failure modes & pitfalls

- **Skipping the rearrangement check.** Any mechanism through a carbocation (SN1, E1, HX addition, acid-catalyzed hydration/dehydration, pinacol conditions) must be scanned for a 1,2-hydride or 1,2-alkyl shift that upgrades the cation (2°→3°, or into benzylic/allylic, or ring-expands a strained ring). 3-methyl-1-butene + HCl gives mainly 2-chloro-2-methylbutane, not the 2-chloro-3-methylbutane a naive Markovnikov answer produces. If your product retains a 2° center adjacent to a 3° H after a cationic mechanism, re-check.
- **Misapplying "Markovnikov."** The rule is about *cation stability*, not literally "H to the carbon with more H's" — the literal version fails when substituents (aryl, OR, halogen) stabilize the cation contrary to H-count. State it mechanistically: protonate to give the more stable carbocation. Also: peroxide effect reverses HBr only; anti-Markovnikov alcohol needs hydroboration, not "HBr/ROOR then substitution" hand-waving.
- **E2 without checking anti-periplanar geometry.** On cyclohexanes the leaving group must be *axial*; the H removed must also be axial (trans-diaxial). Menthyl chloride eliminates slowly to only the non-Zaitsev alkene (only one axial-H neighbor available in the reactive conformer); neomenthyl chloride eliminates fast to the Zaitsev product. If the substrate is a substituted cyclohexane, draw the chair with the LG axial before answering; the "Zaitsev by default" reflex fails here. On acyclic substrates with bulky bases, expect Hofmann (less-substituted alkene).
- **Racemization claims where none occur, and vice versa.** SN2 at a carbon that isn't the stereocenter changes nothing at the stereocenter. SN1 racemizes *only the cationic carbon*; other stereocenters are untouched (product is diastereomers-relevant, not fully racemic). Reactions that never touch a stereocenter's bonds retain configuration by default — don't "racemize" out of caution.
- **Wrong protonation state at the stated pH.** At pH 7: carboxylic acids (pKa ~4.5) are carboxylates; aliphatic amines (conj. acid pKa ~10.6) are ammonium; phenol is neutral; amino acids are zwitterions. Henderson–Hasselbalch: pH − pKa = log([A⁻]/[HA]); 2 units past the pKa ⇒ 99% one form. Any mechanism written in aqueous base must not feature H₃O⁺ steps, and vice versa — no strong acid and strong base as spectators in the same flask.
- **Using a strong base where a nucleophile was needed on a 3° center.** "NaOEt + t-BuBr → t-Bu-OEt" is wrong; the product is isobutylene (E2). SN on 3° carbons requires SN1 conditions (weak Nu, protic, often solvolysis) — and then rearrangement/elimination compete.
- **Enolate regiochemistry ignored.** LDA at −78 °C → kinetic enolate (less-substituted α-carbon, deprotonation irreversible); NaOEt/ROH at rt → thermodynamic enolate (more-substituted). Crossed aldols without a plan (one enolizable partner, or preformed enolate, or Mukaiyama) give tar; a crossed-aldol answer must name which partner enolizes and why the other can't (no α-H: ArCHO, HCHO, t-BuCHO).
- **Grignard/organolithium used in the presence of protic H or reactive FG.** Any OH, NH, SH, terminal alkyne C–H, CO₂H in substrate or solvent quenches RMgX first (fast acid–base beats slow addition). Also no Grignard formation with an ester/ketone elsewhere in the same molecule — it attacks itself. Check the whole molecule, not just the target carbonyl.
- **Aromatic substitution direction errors.** Activating o/p-directors: OH, OR, NH₂, NR₂, alkyl; deactivating o/p: halogens (the exception pair: deactivating *and* o/p); deactivating meta: NO₂, CN, C=O, SO₃H, NR₃⁺. In aniline nitration, the strongly acidic medium protonates NH₂ → NH₃⁺ (meta director); acetylate first (acetanilide) to keep o/p direction, then hydrolyze. Friedel–Crafts fails on strongly deactivated rings (nitrobenzene) and on aniline (Lewis acid complexes the N); FC alkylation also polyalkylates and rearranges (n-PrCl/AlCl₃ gives cumene, not n-propylbenzene — acylate then Clemmensen/Wolff–Kishner instead).
- **Oxidation/reduction scope mistakes.** PCC/DMP/Swern: 1° alcohol → aldehyde (stop); CrO₃/H₂SO₄ (Jones), KMnO₄: 1° → carboxylic acid; NaBH₄ reduces aldehydes/ketones but not esters/amides; LiAlH₄ reduces almost all carbonyls (ester → 1° alcohol, amide → amine); DIBAL, 1 equiv, −78 °C: ester → aldehyde. Mixing these scopes is a top-3 synthesis-question error.
- **Forgetting that resonance requires geometry.** An "allylic" cation whose empty p orbital is orthogonal to the π system (bridgeheads, twisted biaryls) gets no resonance stabilization; amide N lone pair is in the π system (planar N, non-basic, non-nucleophilic N) — attack/protonation of amides happens at O.

## Worked micro-examples

**1. Full decision-matrix run.** (2R)-2-bromobutane + NaSMe in DMSO, rt.
Substrate: 2° → SN2 and E2 both possible. Nu: MeS⁻ — excellent Nu (polarizable), weak base (MeSH pKa ≈ 10.3, far below alkoxide) → substitution wins. Solvent: DMSO, polar aprotic → SN2 accelerated. T: rt, no elimination push. Verdict: SN2, backside attack at C2, inversion: product is (2S)-2-(methylthio)butane, single enantiomer. Swap NaSMe for KOt-Bu: bulky strong base → E2, and t-BuO⁻ prefers the less hindered β-H → Hofmann product 1-butene major (with some cis/trans-2-butene).

**2. Rearrangement catch.** 3,3-dimethyl-1-butene + HBr. Protonation gives the 2° cation at C2 (Markovnikov). Adjacent C3 is quaternary-bearing: a 1,2-methyl shift converts 2° → 3° cation at C3. Br⁻ traps the 3° cation: major product 2-bromo-2,3-dimethylbutane, not the "expected" 2-bromo-3,3-dimethylbutane. Rule: after forming any cation, always ask "does a hydride/alkyl shift from the neighbor give a more stable cation?" before drawing the nucleophile's attack.

**3. Retrosynthesis with an umpolung step.** Target: 1-phenyl-2-butanone? No — take 4-hydroxy-4-phenylbutan-2-one (PhCH(OH)CH₂COCH₃). 1,3-relationship of OH and C=O ⇒ aldol disconnect at the C–C between carbinol C and α-C: synthons = PhCHO (acceptor) + ⁻CH₂COCH₃ (donor = acetone enolate). Forward: acetone enolate (excess acetone or preformed) + benzaldehyde (no α-H on PhCHO ⇒ clean crossed aldol), no dehydration (stop at aldol: mild base, low T). Contrast: if the target were PhCOCH₂CH₂CH₃ disconnected as PhC(=O)⁻ + ⁺CH₂CH₂CH₃, the acyl *anion* synthon has unnatural polarity ⇒ real equivalent: 2-phenyl-1,3-dithiane, deprotonate (n-BuLi), alkylate with n-PrBr, hydrolyze (HgCl₂/H₂O).

## Verification / self-check

1. **Arrow audit:** every arrow starts at a pair (lone pair or bond) and ends at an atom/bond; no carbon exceeds 4 bonds at any drawn instant; formal charges recomputed after each step and conserved overall.
2. **Cation formed anywhere? Run the rearrangement scan** (neighboring H or alkyl that upgrades stability) before committing to a product.
3. **Stereochemistry consistency:** name the mechanism, then check the drawn stereocenters obey it (SN2 inverted? syn-addition on the same face? E2 anti-periplanar achievable in a drawn chair/Newman?). If the product is drawn as a single enantiomer, confirm the mechanism can actually deliver enantiopurity (achiral reagents on achiral substrates give racemates).
4. **pKa sanity:** every proton transfer goes from stronger acid to give weaker acid; no species coexists with something that would instantly quench it (RMgX + ROH, LDA + ketone solvent, H₃O⁺ steps under basic conditions).
5. **Mass/atom balance:** count carbons and heteroatoms in target vs starting materials + reagents; leftover or missing atoms mean a forgotten byproduct or a wrong disconnection.
6. **Reagent scope check:** for each named reagent confirm the transformation is inside its scope table (e.g., NaBH₄ won't touch the ester you're hoping it reduces).
