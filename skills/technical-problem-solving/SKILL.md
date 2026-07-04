---
name: technical-problem-solving
description: Use for general expert problem-solving heuristics — restating a problem in multiple representations, working backward from the goal, checking extreme/limiting cases, using dimensional analysis and invariants as free constraints, solving a simpler version first, decomposition/analogy/transformation as master moves, Fermi estimation and order-of-magnitude sanity checks, recognizing and escaping being stuck, and verifying solutions as a separate mode. Load for hard analytical, mathematical, quantitative, or engineering problems.
---

# Technical problem solving

## Core mental model

1. **The representation is usually the difficulty.** Most hard problems become easy in the right representation and stay impossible in the wrong one. Before grinding, restate the problem in two or three forms: algebraic ↔ geometric ↔ graph ↔ probabilistic ↔ physical. Counting problems often collapse under a bijection; optimization under a change of variables; dynamics under the right coordinate frame. If you're stuck, the first move is almost always *re-represent*, not push harder.
2. **Three master moves: decompose, analogize, transform.** Decompose — break into independent subproblems you can solve separately and recombine. Analogize — map to a solved problem in another domain and port the solution. Transform — change variables/coordinates/domain so the problem simplifies (log to turn products into sums, Fourier to turn convolution into multiplication, duality to swap hard for easy). Most breakthroughs are one of these three.
3. **Work backward from the goal.** Assume you have the answer; ask what would immediately produce it, then what produces *that*. Goal-directed search prunes the space far faster than forward flailing, especially for proofs, constructions, and multi-step derivations.
4. **Free constraints are everywhere — harvest them.** Dimensional analysis fixes the form of an answer up to a constant and catches errors instantly. Invariants (conserved quantities, symmetries, parity, monovariants) rule out whole classes of outcomes. Extreme and limiting cases must behave sensibly. These cost nothing and constrain enormously.
5. **Generation and verification are different modes — separate them.** The mindset that produces a candidate answer is not the one that checks it. After solving, switch deliberately into adversarial mode: try to break your own answer with edge cases, limits, and independent methods.

## Decision frameworks

- **When stuck, in order:** (1) re-represent (new coordinates, dual, picture, table); (2) solve a smaller/simpler instance and look for the pattern (n=1,2,3; drop a dimension; relax a constraint); (3) work backward from the goal; (4) find an invariant or symmetry; (5) invert the question ("what would make this impossible / when does it fail?"); (6) timebox and step away — if 20–30 minutes yield no progress on the current representation, change the representation rather than deepen the same dead end.
- **Which master move:** many similar independent parts → decompose. Structurally like a problem you've solved → analogize. A messy operation with a known simplifying map (product→sum, convolution→product, min→max) → transform.
- **Estimation:** decompose the unknown into factors you *can* estimate (Fermi), estimate each to the nearest order of magnitude, multiply, and carry units. Then sanity-check the result's magnitude against anything you know. Every numeric answer gets an order-of-magnitude reasonableness check before you trust it.
- **Sanity checks to run on any answer:** dimensions consistent? limiting cases correct (zero, infinity, symmetry point)? monotonic in the right direction? sign right? order of magnitude plausible? special case matches a known result?
- **When to trust vs re-derive:** if two independent methods agree, trust it. If an answer only comes from one fragile chain of algebra, re-derive it a different way before relying on it.

## The master moves, expanded

- **Decompose** when the problem has separable structure: independent subproblems, a sum you can split, stages you can chain, cases that partition the space. The art is finding a decomposition where the pieces are genuinely easier and recombine cleanly (divide-and-conquer, superposition, casework on a well-chosen variable).
- **Analogize** when the structure matches a solved problem in another domain: a flow network, a Markov chain, a resistor network, a recurrence, a known combinatorial identity. Port the machinery. Danger: check the analogy's assumptions actually hold before trusting the ported result.
- **Transform** when a change of representation trivializes the operation: logarithms turn products into sums; Fourier/Laplace turn convolution and differentiation into multiplication; generating functions turn recurrences into algebra; duality swaps a hard max for an easy min; a coordinate change aligns with the symmetry. The transformed problem, solved, is mapped back.

## Estimation discipline

- **Fermi decomposition:** never guess a large or unfamiliar quantity directly. Factor it into pieces each known to within an order of magnitude, estimate each, multiply, and carry units the whole way. Errors in independent factors partly cancel, so the product is usually within a factor of a few.
- **Anchor with knowns:** tie estimates to numbers you're sure of (population ~8B, seconds/year ~3×10⁷, Avogadro ~6×10²³, a person ~70 kg). 
- **Bracket:** compute a clearly-too-low and clearly-too-high bound; the truth is between and often near the geometric mean. This catches wild errors even when a point estimate is hard.

## Failure modes and pitfalls

- **Committing to the first representation.** Grinding algebra on something that's one picture away from obvious, or brute-forcing a case analysis a symmetry would collapse. Symptom: the work is getting *more* complicated, not less. Cure: stop and re-represent.
- **Solving forward when the goal should drive.** Deriving lots of true-but-irrelevant consequences instead of asking what the target requires. For proofs and constructions, work backward.
- **Skipping the simpler version.** Attacking the general n-dimensional, k-parameter case directly when n=1 would reveal the mechanism. Always solve the smallest nontrivial instance first; the general solution is usually the small one with indices added.
- **No dimensional check.** Presenting a formula where the units don't match (adding a length to an area, an energy to a force). This is a free, instant error detector — a velocity answer must have velocity dimensions. Physicists catch most algebra slips this way.
- **No order-of-magnitude check.** Reporting a numeric answer 1000× off because of a unit slip or a dropped factor, without asking "is this number even plausible?" Every final number gets a Fermi-style reasonableness test. If someone weighs 7000 kg, you made an error.
- **Verifying with the same method that produced the answer.** Re-running identical algebra confirms nothing — it reproduces the same mistake. Verify with an *independent* route: a special case, a limiting behavior, a different derivation, a numerical spot-check.
- **Ignoring boundary and degenerate cases.** The answer works generically but breaks at n=0, empty set, division by a quantity that can vanish, or coincident points. Test extremes explicitly.
- **Confusing "I found a plausible answer" with "I verified it."** Plausibility is generation; verification is a separate, adversarial pass. Don't skip it because the answer "feels right."
- **Not exploiting invariants.** Missing that a quantity is conserved (so many "reachable states" questions are just parity/invariant arguments) or that a monovariant strictly decreases (proving termination). If a problem asks "can you reach X," look for an invariant that X violates.
- **Estimation without decomposition.** Guessing a big number directly instead of factoring it into estimable pieces. Fermi decomposition turns an impossible guess into a product of defensible ones.
- **Timeboxing failure — sunk-cost persistence.** Pouring more time into a stuck approach because you've already invested. Set a limit; when it passes with no traction, deliberately switch representation or invert the question.
- **Losing track of what's known vs assumed.** In long derivations, quietly assuming what you're trying to prove (circularity) or carrying an unjustified step. Keep the ledger of established vs goal explicit.
- **Generalizing from too few cases.** Seeing a pattern hold for n=1,2,3 and asserting it for all n without proof. Small cases *suggest* conjectures; they don't establish them. Look for the mechanism (induction step, bijection) that would force the pattern, and test one case designed to break it.
- **Anchoring on a remembered formula that doesn't apply.** Pattern-matching to a familiar result whose assumptions your problem violates (applying a closed-form that assumes independence, linearity, or a boundary condition you don't have). Verify the preconditions before plugging in.
- **Precision theater.** Reporting an estimate to 4 significant figures when the inputs are order-of-magnitude guesses. Match the precision of the answer to the precision of the inputs; false precision hides the real uncertainty.
- **Over-decomposing.** Splitting into so many pieces that bookkeeping errors and lost interactions between pieces cost more than the decomposition saved. Decompose along the natural seams, not arbitrarily.
- **Solving a harder problem than asked.** Missing that the question wants existence, not construction; a bound, not the exact value; or one example, not all of them. Re-read the actual ask before committing effort.
- **Not noticing the problem is under- or over-determined.** Too few constraints → many solutions (the "answer" isn't unique); too many → possibly none (check consistency). Count degrees of freedom vs constraints early.

## Worked micro-examples

**1. Re-representation collapses a sum.** "Sum 1 + 2 + … + n." Forward grind is fine, but the *representation* insight — pair first and last (1+n), there are n/2 pairs each summing n+1 — gives n(n+1)/2 instantly and generalizes (Gauss). The geometric view (two triangles form an n×(n+1) rectangle) is the same idea, and shows why it's exact for all parities.

**2. Dimensional analysis fixes a form.** Period of a pendulum: guess it depends on length L, gravity g, mass m. [T] = time. Only √(L/g) has dimension of time ([L]/[L/T²])^{1/2} = [T²]^{1/2} = T; mass can't appear (no other mass to cancel [M]). So T = C·√(L/g) with C dimensionless — and indeed C = 2π. Dimensional analysis got the entire functional form from nothing, and correctly predicts mass-independence.

**3. Fermi estimate with a check.** Piano tuners in Chicago: ~3M people ÷ ~3 per household ≈ 1M households; ~1 in 20 owns a piano → 50,000 pianos; tuned ~once/year; a tuner does ~4/day × ~250 days ≈ 1000/year → ~50 tuners. Sanity check: order of magnitude tens-to-hundreds, not thousands — plausible for a service profession. The decomposition, not the final guess, is the skill.

**4. Invariant kills a "reachability" question.** "A 8×8 board with two opposite corners removed — can 31 dominoes tile it?" Color like a checkerboard: each domino covers one black + one white. Opposite corners share a color, so 30 of one color and 32 of the other remain — the invariant (equal counts) is violated. No tiling exists, proven in one line by choosing the right representation (coloring/invariant) instead of enumerating placements.

**5. Work backward on a construction.** "Make 24 from 3, 3, 8, 8 using +−×÷." Forward search is huge. Backward: 24 = 8 × 3, or 24 = 8 ÷ (something small). Try 8/(3 − 8/3): 8/3 = 2.667, 3 − 2.667 = 0.333, 8/0.333 = 24. Goal-directed factoring (what pairs multiply/divide to 24?) prunes the search to almost nothing.

**6. Limiting case catches an error.** You derive that the resistance of two resistors in parallel is R₁ + R₂. Test the limit R₂ → 0 (a short): physically the combination should → 0, but the formula gives R₁. Contradiction — the formula is wrong. The correct R₁R₂/(R₁+R₂) → 0 as R₂ → 0. The limiting case exposed the mistake in seconds without re-checking the algebra.

## Verification and self-check

- **Ran an independent verification** — special case, limiting behavior, alternate derivation, or numerical check — not a re-run of the same algebra.
- **Dimensions consistent** throughout and in the final answer.
- **Limiting/extreme cases correct:** zero, infinity, symmetry points, and degenerate inputs all give sensible results.
- **Order of magnitude sanity-checked** on every numeric answer against something known.
- **Sign and monotonicity** point the right way (answer moves correctly as inputs change).
- **No circularity** — the derivation never assumed the conclusion; known vs goal stayed distinct.
- **If only one fragile derivation exists**, re-derived it a second way before trusting it.
