---
name: physical-chemistry-and-thermodynamics
description: Use for thermodynamic and kinetic reasoning — spontaneity vs rate, ΔG/ΔH/ΔS, equilibrium and Le Chatelier via K and Q, rate laws from mechanisms, Arrhenius activation-energy arithmetic, phase equilibria and colligative properties, electrochemistry and the Nernst equation, and units discipline (R, kJ vs J). Load when a problem involves free energy, equilibrium constants, reaction rates, cell potentials, or entropy reasoning.
---

# Physical chemistry and thermodynamics

## Core mental model

1. **Spontaneity is not speed.** ΔG < 0 says a process *can* happen and how far it will go; it says *nothing* about how fast. Diamond → graphite has ΔG < 0 at room T and takes geologic time. This is the single most common error: never infer a rate from a ΔG, and never infer a position of equilibrium from an activation energy. Thermodynamics = destination; kinetics = travel time.
2. **State functions vs path functions.** U, H, S, G, T, P, V are state functions — ΔX depends only on endpoints, so you may choose any convenient path (Hess's law, thermodynamic cycles). q and w are path functions — "the heat of the system" is meaningless; only q for a *specified path* exists. First law: ΔU = q + w (chemistry sign convention: w = −P_ext ΔV, work done *on* system positive).
3. **ΔG = ΔH − TΔS is the master equation**, but only ΔG at constant T and P predicts spontaneity, and only when ΔH and ΔS are themselves approximately T-independent can you extrapolate across temperatures. The four sign regimes: (ΔH<0,ΔS>0) always spontaneous; (ΔH>0,ΔS<0) never; (ΔH<0,ΔS<0) spontaneous at low T; (ΔH>0,ΔS>0) spontaneous at high T. The crossover temperature is T = ΔH/ΔS.
4. **Equilibrium is dynamic and quantitative.** Le Chatelier is a qualitative crutch; the rigorous tool is comparing Q to K. ΔG = ΔG° + RT ln Q. At equilibrium ΔG = 0 and Q = K, so **ΔG° = −RT ln K**. ΔG° tells you K; ΔG (with actual Q) tells you which way *this* mixture moves right now.
5. **Entropy is counting.** S = k_B ln W. Entropy increases because there are overwhelmingly more microstates for dispersed energy/matter. This intuition instantly predicts signs: gas > liquid > solid; more moles of gas = higher S; mixing increases S; dissolving a gas *decreases* S.

## Decision frameworks

- **"Will it react?" vs "how fast?"** If asked about feasibility/yield/direction → thermodynamics (ΔG, K, Q). If asked about rate/time/temperature-sensitivity of speed/catalyst → kinetics (rate law, Ea). Catalysts change *only* kinetics (lower Ea, speed both directions equally); they never shift K or equilibrium position.
- **Which free energy?** Constant T,P (typical lab/bio) → Gibbs G. Constant T,V → Helmholtz A. Almost always G in chemistry.
- **Finding a rate law:** derive from mechanism, don't guess. If there's a clear slow step, rate = rate of RDS (rewriting any intermediate via prior fast equilibria). If no single slow step, apply the steady-state approximation to reactive intermediates (d[I]/dt ≈ 0). Pre-equilibrium is the special case where a fast reversible step precedes the RDS.
- **Le Chatelier, quantified:** compute Q after the disturbance and compare to K. Adding inert gas at constant V changes nothing (partial pressures unchanged). Adding inert gas at constant P shifts toward more moles of gas. Raising T: shifts endothermic direction and *changes K itself* (van't Hoff); pressure/concentration changes shift position but leave K fixed.
- **Cell spontaneity:** E°_cell > 0 ⟺ ΔG° < 0 ⟺ K > 1, via ΔG° = −nFE°. Compute E°_cell = E°_cathode − E°_anode (both as reduction potentials — do NOT flip the sign of the anode's tabulated reduction potential; the subtraction handles it).

## More on kinetics vs thermodynamics

- **Thermodynamic vs kinetic product.** A reaction can yield different major products depending on whether it runs under kinetic control (fast, low-Ea path, low T, irreversible — the *fastest-formed* product dominates) or thermodynamic control (reversible, high T, long time — the *most stable* product dominates). 1,2- vs 1,4-addition to dienes is the classic case. When a question specifies temperature or reversibility, decide which regime governs before naming the product.
- **Catalysis is purely kinetic.** A catalyst provides a lower-Ea path and accelerates forward and reverse equally, so it reaches the *same* equilibrium faster. It never changes ΔG, K, or equilibrium yield. Anyone claiming a catalyst "shifts equilibrium" or "increases yield at equilibrium" is wrong (it can improve yield only by reaching equilibrium within available time, or by selecting one product kinetically).
- **Enzyme/transition-state logic.** Rate depends on ΔG‡ (activation free energy), the gap from reactants to the transition state; equilibrium depends on ΔG, the gap between reactants and products. They are independent — a huge driving force can sit behind a huge barrier.

## Failure modes and pitfalls

- **kJ/J mismatch in ΔG = ΔH − TΔS.** ΔH is usually kJ/mol, ΔS is J/(mol·K). You *must* convert ΔS to kJ or ΔH to J before subtracting. Forgetting this gives answers off by 1000. Always write units on every term.
- **Wrong R.** Use R = 8.314 J/(mol·K) with energies in joules; R = 0.08206 L·atm/(mol·K) for PV = nRT with pressure in atm and volume in L. In ΔG° = −RT ln K, R is 8.314 J/(mol·K) so ΔG° comes out in J/mol — then don't report it as kJ without dividing by 1000.
- **Putting units or concentrations into K/Q for pure solids and liquids.** Pure solids, pure liquids, and solvents have activity 1 — omit them from K and Q. A common error is including [H₂O] in aqueous equilibria.
- **Nernst sign and n errors (the top electrochemistry trap).** E = E° − (RT/nF) ln Q, or at 298 K, E = E° − (0.0592/n) log Q. Q is written products-over-reactants *as the cell reaction is written*; n is electrons transferred in the *balanced* equation. Getting n wrong (e.g., n=1 when the balanced reaction transfers 2) halves your correction. When [products] rises, Q rises, ln Q > 0, E drops — sanity-check that consuming reactants lowers cell potential.
- **Flipping the anode potential twice.** People negate the anode's reduction potential AND subtract it. Do one or the other. Standard method: E°_cell = E°_cathode(reduction) − E°_anode(reduction).
- **Assuming ΔH, ΔS constant across huge T ranges.** Fine over tens of degrees; risky over hundreds. For K vs T use van't Hoff: ln(K₂/K₁) = −(ΔH°/R)(1/T₂ − 1/T₁).
- **Confusing ΔG and ΔG°.** ΔG° uses standard states (1 bar, 1 M, Q would be 1). A reaction with ΔG° > 0 still proceeds forward until Q rises to K if it starts with Q < K. "Nonspontaneous under standard conditions" ≠ "won't happen."
- **Reaction order ≠ stoichiometric coefficient.** Order comes from experiment or mechanism, never from the balanced equation coefficients (unless the reaction is a single elementary step).
- **Sign of activation energy / Arrhenius direction.** Higher T → larger k always (Ea > 0). If your arithmetic gives k decreasing with T, you flipped a sign in 1/T₁ − 1/T₂.
- **Colligative properties depend on particle count, not identity.** Use the van't Hoff factor i: NaCl gives i ≈ 2, CaCl₂ i ≈ 3 (less at real concentrations due to ion pairing). ΔT_f = i·K_f·m uses *molality*, not molarity.
- **Entropy of the surroundings forgotten.** The second law is about the *universe*: ΔS_univ = ΔS_sys + ΔS_surr ≥ 0, with ΔS_surr = −ΔH_sys/T. An endothermic reaction can be spontaneous because ΔS_sys is large enough — that's exactly what ΔG < 0 encodes.
- **Le Chatelier misapplied to inert gas or catalyst.** Neither shifts equilibrium at constant V; a catalyst never changes K.
- **Ideal-gas assumptions applied to real conditions.** PV = nRT and ΔS of mixing assume ideality; at high P or low T, fugacity/activity corrections matter. Flag when a problem pushes into non-ideal territory rather than reporting spurious precision.
- **Confusing q_p and q_v with enthalpy.** ΔH = q_p (heat at constant pressure); ΔU = q_v (heat at constant volume). ΔH = ΔU + Δ(PV) ≈ ΔU + Δn_gas·RT for reactions. Bomb calorimetry measures ΔU; converting to ΔH requires the Δn_gas·RT term — dropping it is a common error.
- **Temperature in K, always.** Every thermodynamic and kinetic formula (ΔG = ΔH − TΔS, Arrhenius, Nernst, van't Hoff, PV = nRT) uses absolute temperature. Plugging °C is a silent, severe error.
- **Standard-state pressure ambiguity.** Modern convention is 1 bar (older tables use 1 atm); the ~1% difference rarely matters but the *reference* matters for ΔG° consistency. Keep p° explicit in ln(p/p°) so K stays dimensionless.
- **Entropy sign of dissolving.** Dissolving a solid usually raises S (more disorder), but dissolving a gas *lowers* S (gas → constrained in solution), and highly charged ions can order water (negative hydration entropy). Don't assume "dissolving increases entropy" universally.

## Worked micro-examples

**1. Crossover temperature.** For CaCO₃(s) → CaO(s) + CO₂(g), ΔH° = +178 kJ/mol, ΔS° = +161 J/(mol·K). Above what T is decomposition spontaneous at standard state?
T = ΔH°/ΔS° = 178,000 J/mol ÷ 161 J/(mol·K) = 1106 K ≈ 833 °C. (Note the J conversion — using 178 without ×1000 gives a nonsensical 1.1 K.) Below this, ΔG° > 0; above, the +TΔS term wins.

**2. Arrhenius from two temperatures.** k doubles from 300 K to 310 K. Find Ea.
ln(k₂/k₁) = −(Ea/R)(1/T₂ − 1/T₁). ln 2 = −(Ea/8.314)(1/310 − 1/300).
1/310 − 1/300 = 0.0032258 − 0.0033333 = −1.075×10⁻⁴ K⁻¹.
0.6931 = −(Ea/8.314)(−1.075×10⁻⁴) → Ea = 0.6931 × 8.314 / 1.075×10⁻⁴ = 5.36×10⁴ J/mol ≈ 53.6 kJ/mol. (Rule of thumb: near room T, doubling per 10 K ⟹ Ea ≈ 50 kJ/mol.)

**3. Steady-state kinetics.** Mechanism: (1) A ⇌ B (k₁, k₋₁), (2) B → P (k₂). Apply steady state to B:
d[B]/dt = k₁[A] − k₋₁[B] − k₂[B] ≈ 0 → [B] = k₁[A]/(k₋₁ + k₂).
Rate = k₂[B] = k₁k₂[A]/(k₋₁ + k₂). If k₂ ≪ k₋₁ (pre-equilibrium limit), rate ≈ (k₁k₂/k₋₁)[A], first order. If k₂ ≫ k₋₁, rate ≈ k₁[A], step 1 rate-limiting.

**4. Nernst.** Daniell cell Zn|Zn²⁺(0.10 M)||Cu²⁺(1.0 M)|Cu. E° = 0.34 − (−0.76) = 1.10 V, n = 2.
Q = [Zn²⁺]/[Cu²⁺] = 0.10/1.0 = 0.10.
E = 1.10 − (0.0592/2) log(0.10) = 1.10 − (0.0296)(−1) = 1.13 V. Lower [Zn²⁺] (less product) raises E — consistent.

**5. ΔG° ↔ K.** A reaction has K = 1×10⁴ at 298 K. ΔG° = −RT ln K = −(8.314)(298) ln(10⁴) = −(8.314)(298)(9.21) = −22,800 J/mol = −22.8 kJ/mol. Rule of thumb check: ΔG° ≈ −5.7 kJ/mol per factor of 10 in K, and 4 decades × −5.7 = −22.8 kJ/mol. Match. Note the answer emerges in J/mol from R in J — divide by 1000 to report kJ.

**6. van't Hoff extrapolation.** An exothermic reaction (ΔH° = −50 kJ/mol) has K = 100 at 298 K. Estimate K at 350 K.
ln(K₂/100) = −(ΔH°/R)(1/350 − 1/298) = −(−50000/8.314)(0.002857 − 0.003356) = −(−6014)(−4.99×10⁻⁴) = −3.00.
K₂ = 100·e^(−3.00) = 100·0.0498 = 5.0. K falls with T for an exothermic reaction — Le Chatelier confirms (heating an exothermic reaction shifts it back), and the sign came out right.

## Verification and self-check

- **Units audit first.** Every energy term same unit (J or kJ); R matches; ΔS in per-K; molality vs molarity correct; pressures in the unit your R expects.
- **Sign sanity:** endothermic + entropy-increasing → spontaneous only at high T; a spontaneous cell has E°_cell > 0 and K > 1; higher T always speeds reactions.
- **Limiting cases:** does K → ∞ give ΔG° → −∞? Does Q = K give ΔG = 0? Does a catalyst leave K unchanged in your answer? Does removing product drive the reaction forward (Q < K)?
- **Order of magnitude:** activation energies for ordinary reactions are tens to low hundreds of kJ/mol; ΔG° of a few ×10 kJ/mol corresponds to K spanning many orders (ΔG° = −RT ln K ≈ −5.7 kJ/mol per factor of 10 in K at 298 K).
- **Cross-check thermodynamics against kinetics separately** — if a question gives both ΔG and Ea, make sure you used ΔG only for yield/direction and Ea only for rate.
