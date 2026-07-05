---
name: cheminformatics
description: Load when working with molecular data — SMILES/InChI/mol files, RDKit code, molecular fingerprints and similarity search, SMARTS substructure queries, property filters (Lipinski/QED), reaction SMILES, chirality handling, or preparing chemical datasets for machine learning (deduplication, salt stripping, splits).
---

# Cheminformatics

## Core mental model

1. **A SMILES string is not a molecule; it's one serialization of a graph under one toolkit's chemistry model.** The same molecule has many valid SMILES (`c1ccccc1O`, `Oc1ccccc1`, `C1=CC=CC=C1O`); string equality means nothing. Identity requires canonicalization *within one toolkit* — canonical SMILES differ *between* toolkits (RDKit vs OpenEye vs OpenBabel) and between versions. For cross-system identity, use InChI/InChIKey (standardized layers, one algorithm) — but know InChI normalizes some tautomers and can merge structures you wanted distinct, and InChIKey collisions, while rare, exist. InChIKey block structure: **first 14 chars = skeleton/connectivity only** (stereo- and isotope-agnostic — enantiomers and diastereomers collide here); **second block (8 chars) = stereo/isotope/protonation layers**. So connectivity-only matching uses the first block; any stereo-sensitive identity needs at least the first two blocks — never dedup a stereo-rich set on the first block alone.
2. **Aromaticity, tautomers, and protonation are *conventions*, not facts in the file.** RDKit's aromaticity model differs from Daylight's and from what ChemDraw drew; a tautomer is drawn one way and exists as an ensemble; the mol file's protonation state is whatever the depositor chose, not what exists at pH 7.4. Every pipeline needs an explicit standardization step that picks conventions and applies them uniformly to *everything* (train + test + query).
3. **Fingerprint similarity is a weak, radius-limited proxy.** Morgan/ECFP fingerprints see local circular environments (radius 2 ≈ ECFP4); Tanimoto on them measures shared substructure, not shared biology. Tanimoto 0.85 pairs *usually* share activity, but activity cliffs — single-atom changes (methyl→chloro, added F, N-position swap in a ring) causing >100× potency shifts — are common precisely among high-similarity pairs. Similarity is a prior, never a verdict.
4. **Dataset hygiene decides whether your ML result is real.** Chemical datasets are pathologically redundant (salts, duplicates under canonicalization, stereoisomer copies, congeneric series). Random splits leak series between train and test and inflate metrics massively; the honest question "does this generalize to new chemistry?" requires scaffold (or time/cluster) splits. Most inflated benchmark numbers in the literature die here.
5. **Silence is the enemy: toolkits fail loud (sanitization errors) or quietly (dropped stereo, stripped charges, "fixed" valences).** Treat every parse, standardization, and file round-trip as a lossy operation until verified. Count molecules, count stereocenters, count charges before and after every pipeline stage.

## Decision frameworks

**Representation choice:**

| Need | Use | Because |
|---|---|---|
| Human-readable, compact storage/exchange | Canonical isomeric SMILES (`Chem.MolToSmiles(mol)` — isomeric by default in RDKit) | Compact; but toolkit-canonical only |
| Cross-database identity/dedup key | InChIKey (full 27 chars) | Toolkit-independent; watch tautomer merging |
| Coordinates, drawing fidelity, sdf metadata | Mol/SDF (V2000; V3000 for >999 atoms, enhanced stereo) | Only format carrying 2D/3D coords and per-record data fields |
| Substructure query | SMARTS | SMILES-as-query silently means something different (see pitfalls) |
| Reactions | Reaction SMILES / SMIRKS, atom-mapped if you need to track atoms | Unmapped reaction SMILES can't answer "which atom went where" |

**RDKit standardization pipeline (in this order, for anything from the wild):**
```python
from rdkit import Chem
from rdkit.Chem.MolStandardize import rdMolStandardize
mol = Chem.MolFromSmiles(smi)                    # None on failure — check!
if mol is None:
    return None                                  # log & quarantine upstream, don't drop silently
mol = rdMolStandardize.Cleanup(mol)              # normalize nitro/N-oxide etc., strip Hs
mol = rdMolStandardize.FragmentParent(mol)       # salt/solvate stripping → largest organic fragment
mol = rdMolStandardize.Uncharger().uncharge(mol) # neutralize where possible (decide if you want this)
# optional, order matters: tautomer canonicalization LAST and only if you accept its choices
mol = rdMolStandardize.TautomerEnumerator().Canonicalize(mol)
key = Chem.MolToSmiles(mol)                      # dedup key (or InChIKey for cross-toolkit)
```
Decide *and document* each optional step: uncharging changes pKa-relevant identity; tautomer canonicalization merges records; `FragmentParent` on a salt where the counterion IS the drug (e.g., a quaternary ammonium chloride) keeps the cation — check charge afterward.

**Sanitization error triage (`MolFromSmiles` returns `None` / `SanitizeMol` raises):**
- "Explicit valence for atom # N greater than permitted" → real valence error (pentavalent C from a bad bond order in the source file, or charged N drawn without `+`). Fix upstream or set the charge; don't blanket-disable sanitization.
- "Can't kekulize mol" → aromatic ring RDKit's model can't assign alternating bonds to; commonly deposited structures with aromatic bonds on non-aromatic (by RDKit's model) rings, or missing H on pyrrole-type N (`c1ccnc1` fails; `c1cc[nH]c1` parses). Fix the H count, don't just switch to `Chem.MolFromSmiles(smi, sanitize=False)`.
- If you *must* recover partially: `MolFromSmiles(smi, sanitize=False)` then `Chem.SanitizeMol(mol, sanitizeOps=Chem.SANITIZE_ALL ^ Chem.SANITIZE_KEKULIZE)` — a deliberate, logged exception, and downstream code must not assume aromaticity flags are meaningful.

**Fingerprint/similarity selection:** Morgan radius 2, 2048 bits (`rdFingerprintGenerator.GetMorganGenerator(radius=2, fpSize=2048)`) is the default for activity-ish similarity; radius 3 for more specificity; RDKit FP/MACCS only for coarse diversity picking. Count vectors (not bit vectors) when comparing molecules of very different size. **Thresholds are fingerprint-specific — calibrate on your own background distribution, do not import a folklore number.** The widely-quoted "0.85 ⇒ shares activity" is a *Daylight/MACCS-era* rule; on ECFP4 the meaningful band is lower and softer (background between random drug-like pairs sits ~0.1–0.2, so ~0.4 is already a real cousin and ~0.85 finds near-duplicates only). Reporting an ECFP4 Tanimoto against the 0.85 heuristic systematically over-reads dissimilarity. For "same series?" questions use Bemis–Murcko scaffolds (`MurckoScaffold.GetScaffoldForMol`), not similarity.

**Split strategy for ML evaluation:** random split answers "can I interpolate within these series?" (rarely the real question); scaffold split (Bemis–Murcko, put whole scaffolds in one side) answers "new chemotype?"; time split answers "prospective performance?"; cluster split (Butina on fingerprints) when scaffolds fragment oddly. Report which question your split answers. If performance drops 15–30 points from random to scaffold split, that's normal — the random number was the lie.

## Failure modes & pitfalls

- **Deduplicating on raw SMILES strings.** `CCO` and `OCC` are the same molecule, different strings; datasets deduped by string equality retain thousands of duplicates → train/test leakage even with "no duplicates." Always dedup on canonical SMILES after full standardization (and decide: do stereoisomers count as duplicates for your task? tautomers?).
- **Using a SMILES as a substructure query.** `Chem.MolFromSmiles('c1ccccc1')` as a query matches benzene rings, but `CC(=O)O` as a query matches acetic acid *substructures with those exact H-counts implied loosely* — SMILES queries ignore your intent about charge, aromaticity, and substitution. Use SMARTS: carboxylic acid = `[CX3](=O)[OX2H1]`, carboxylate = `[CX3](=O)[O-]`, either = `[CX3](=O)[OX1H0-,OX2H1]`. And know the sharp edges: SMARTS `C` = aliphatic carbon only, `c` = aromatic only, `[#6]` = any carbon; uppercase-vs-lowercase silently halves your matches. `[N]` in SMARTS means nitrogen with *default* query constraints unlike bare `N` in SMILES contexts — always write explicit H/charge/degree constraints (`[NX3;H2]`) rather than trusting defaults.
- **Aromaticity mismatch between drawing and toolkit.** A molecule stored with kekulized bonds may match aromatic SMARTS after RDKit sanitization but not before; 2-pyridone drawn aromatic in one tool is non-aromatic in RDKit's default model. If a substructure search "misses obvious hits," print `Chem.MolToSmiles` of both query target and check which atoms RDKit flagged aromatic (`atom.GetIsAromatic()`), before blaming the SMARTS.
- **Implicit hydrogen surprises.** RDKit hides Hs by default: `mol.GetNumAtoms()` counts heavy atoms only; SMARTS `[H]` matches *explicit* H atoms (usually none) — use `[C;H3]`/`[#6&H3]` for H-count queries. 3D operations (docking prep, conformers with H-bonds) need `Chem.AddHs(mol)` *before* `AllChem.EmbedMolecule`; embedding without Hs then adding them gives distorted geometry. Conversely `RemoveHs` on export can delete stereo-defining Hs on N/S centers if flags are set carelessly.
- **Chirality dropped or fabricated in round-trips.** Classic bugs: (a) generating canonical SMILES with `isomericSmiles=False` (some legacy code paths) silently erases all stereo — grep for it; (b) mol-file → SMILES conversion losing stereo because wedge bonds weren't perceived: call `Chem.AssignStereochemistryFrom3D(mol)` for 3D, or ensure `AssignChiralTypesFromBondDirs`/`AssignStereochemistry(mol, cleanIt=True, force=True)` after 2D wedge parsing; (c) a dataset where 40% of molecules have *unspecified* stereocenters (`FindMolChiralCenters(mol, includeUnassigned=True)`) treated as if achiral — unspecified ≠ racemic ≠ single-isomer, and models trained mixing (R), (S), and unspecified copies of the same skeleton learn noise; (d) enhanced stereo (AND/OR groups in V3000) silently flattened by V2000 export.
- **Trusting Tanimoto across an activity cliff.** Nearest-neighbor at Tanimoto 0.9 predicted active; the query differs by an ortho-methyl that abolishes binding. Any similarity-based activity claim must be phrased with cliff awareness ("structurally analogous; potency may still differ sharply — matched-pair check advised"), and matched molecular pair analysis is the tool to find cliffs in a series.
- **Lipinski/QED as gatekeepers instead of priors.** Rule-of-5 was a statement about *oral absorption trends* in a 1990s dataset; plenty of drugs violate it (natural products, beyond-Ro5 space) and passing it confers nothing. Use as soft filters/flags in triage; never report "compound is drug-like, QED 0.72" as if it were a validated property. Same for synthetic-accessibility scores: SA score flags obvious nightmares, doesn't certify makability.
- **Property prediction on unstandardized inputs.** LogP/TPSA/pKa-dependent descriptors computed on the salt form, or on the wrong tautomer/protonation state, shift by whole units (`Descriptors.MolLogP` on lisinopril free base vs its zwitterion representation differs materially). Standardize → then compute → and state which form the number describes.
- **Scaffold-split leakage via trivial scaffolds.** Bemis–Murcko maps all acyclic molecules to an empty scaffold and merges huge bins (e.g., all monosubstituted benzenes); a "scaffold split" where 30% of data share the benzene scaffold still leaks. Inspect scaffold size distribution; consider generic (CSK) scaffolds or cluster splits when bins are degenerate.
- **Reaction SMILES misuse.** Treating `reactants>>products` strings without atom maps as if atoms are tracked; applying an unmapped/badly-mapped SMIRKS with `rdChemReactions` and getting combinatorial garbage products (RunReactants returns *all* matches — a template matching 3 sites yields 3 product sets; dedup and sanitize each product, many will fail sanitization and need filtering). Also: reagents vs reactants placement (`A.B>C>D`) is convention-dependent across datasets (USPTO variants differ) — normalize before training.
- **Counting on `MolFromSmiles` errors to be visible.** In bulk loops, `None` returns vanish into list comprehensions (`[f(m) for m in mols if m]` silently drops 4% of the dataset — possibly non-randomly: the failures cluster in organometallics or macrocycles, biasing the dataset). Always log the failure count and inspect a sample of failures.

## Worked micro-examples

**1. Honest dedup + split pipeline (the 20 lines that save the paper).**
```python
from rdkit import Chem
from rdkit.Chem.MolStandardize import rdMolStandardize
from rdkit.Chem.Scaffolds import MurckoScaffold
from collections import defaultdict

parent = rdMolStandardize.FragmentParent
records, failures = {}, 0
for smi, y in raw_data:                     # raw: 10,000 rows
    m = Chem.MolFromSmiles(smi)
    if m is None: failures += 1; continue
    can = Chem.MolToSmiles(parent(rdMolStandardize.Cleanup(m)))
    records.setdefault(can, []).append(y)   # collect duplicate measurements
print(failures, "parse failures;", len(records), "unique after standardization")
# typical outcome: 10,000 → ~7,800 unique; duplicate y-values disagreeing by >1 log unit
# flag assay noise — median-aggregate, don't keep both.
scaff_bins = defaultdict(list)
for can in records:
    s = MurckoScaffold.MurckoScaffoldSmiles(can) or "acyclic"
    scaff_bins[s].append(can)
bins = sorted(scaff_bins.values(), key=len, reverse=True)  # inspect: is bin[0] 'benzene', 2,000 strong?
test, train = [], []
for b in bins: (train if len(train) <= 0.8*len(records) else test).extend(b)
```
Every printed count is a checkpoint; the disagreeing-duplicates check routinely uncovers that "our model's RMSE of 0.6 log units" is at the assay-reproducibility floor — report that context.

**2. SMARTS debugging session.** Task: find primary aliphatic amines. First attempt `N` → matches amides, anilines, nitriles' N, everything. Second `[NH2]` → misses protonated amines, still hits sulfonamide NH₂. Correct reasoning chain: primary aliphatic amine = N with 2 Hs, 1 heavy neighbor that is sp³ carbon, N not adjacent to carbonyl/sulfonyl, allow protonated form:
`[NX3;H2;!$(NC=O);!$(NS(=O)=O)][CX4]` plus (if desired) `[NX4+;H3][CX4]` for ammonium. Validate on a probe set *before* running on the database: butylamine (hit), aniline (miss — `[CX4]` excludes it), acetamide (miss), glycine (hit — decide if wanted), benzylamine (hit). A SMARTS without a probe-set test is untested code.

**3. Similarity number interpretation.** ECFP4/2048 Tanimoto A=0.91, B=0.42, C=0.18: A is a near-duplicate (check first whether it is literally the same compound differently standardized), B is a real same-series cousin worth examining, C is background (random drug-like pairs sit ~0.1–0.2, so 0.18 = "unrelated," not "18% similar"). The common error is discarding B by linear intuition — 0.42 is *not* "not very similar" on ECFP4.

## Verification / self-check

1. **Conservation counts at every stage:** N_in vs N_parsed vs N_standardized vs N_unique — report the losses and eyeball a sample of each loss category. Stereocenter and formal-charge totals before/after round-trips.
2. **Round-trip test:** `MolToSmiles(MolFromSmiles(s))` stable on second pass (idempotent); mol → SMILES → mol preserves InChIKey for a random sample including stereo-rich and charged molecules.
3. **SMARTS validated on a positive/negative probe set** (5–10 hand-picked molecules each) before deployment; report hit counts against expectation on the full set (a query matching 60% of a drug database is broken).
4. **Split integrity:** assert zero canonical-SMILES (and near-duplicate: Tanimoto > 0.95) overlap between train and test; state which generalization question the split answers.
5. **Toolkit/version pinned and stated** for anything canonical (SMILES, tautomers, aromaticity) — a result that changes under `rdkit` version bump must be caught by re-running the conservation counts.
6. **Numbers contextualized:** every similarity threshold, QED/Ro5 verdict, or model metric is presented with its domain of validity (fingerprint type, split type, assay noise floor) — if that context is missing from your draft answer, add it or soften the claim.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 10 claims: ~8 baseline (compressed), ~2 partial (sharpened). Opus 4.8 cold nailed canonical-SMILES dedup, the scaffold-split rationale (~0.05–0.15 metric drop), the carboxylic-acid SMARTS + why-not-SMILES, the RDKit standardization order, InChIKey two-block structure, activity cliffs, the silent-`None`-filtering label-misalignment trap, and RDKit≠OpenBabel canonicalization.
- Genuine deltas sharpened: (a) **ECFP4 Tanimoto thresholds** — Opus defaulted to the conventional 0.7/"0.85-shares-activity" heuristic, which is a Daylight/MACCS-era number; the sharpened rule is calibrate-to-background (~0.1–0.2) and treat ~0.4 as a real cousin on ECFP4. (b) Cleaned the InChIKey block description (the SMARTS aromatic `C` vs `c` vs `[#6]` sharp edge is the other sub-baseline detail Opus glossed).
- Everything else compressed to anchors.
