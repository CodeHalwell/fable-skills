---
name: typescript-mastery
description: Loads expert TypeScript judgment for designing types, debugging type errors, configuring tsconfig, and setting runtime validation boundaries. Use when writing non-trivial TypeScript, reviewing type-level code, fixing generics/variance/narrowing issues, untangling ESM/CJS interop, or deciding how strict a codebase should be.
---

# TypeScript Mastery

## Core mental model

1. **Types are erased axioms; minimize and push them to I/O boundaries.** Every `as`, `any`, and unvalidated `JSON.parse` is an unproven axiom. Validate at trust boundaries (zod/valibot, `type X = z.infer<typeof Schema>` — never a parallel hand-written type); trust the interior. One parse per datum per entry.
2. **Structural typing cuts both ways**: no `implements` needed, but IDs interchange and `{}` matches almost everything. Brand identities that matter: `type UserId = string & { readonly __brand: 'UserId' }` — with exactly one validating constructor; grep for `as UserId` in review.
3. **Narrowing resets at callback and mutation boundaries**; the fix is a `const` snapshot of the narrowed value, not a cleverer type (~80% of "narrowing broke" reports).
4. **Model states with discriminated unions**, discriminating on literal properties — never `instanceof` for anything crossing serialization (deserialized objects lose prototypes; `instanceof` returns false at runtime while the types still claim exhaustiveness).
5. **Prefer the form that produces the better error message at the call site** — a sound API with unreadable errors gets `any`-ed around by teammates.

## Current state (verified July 2026) — assume your training is stale here

The reflex answer is "stable TypeScript is 5.x with 5.x defaults." Wrong as of mid-2026:

- Stable TypeScript is **6.0.x**, and 6.0 **flipped defaults**: `strict: true`, `module: "esnext"`, floating `target` (currently es2025), `types: []` (add `"types": ["node"]` explicitly or Node globals vanish), `noUncheckedSideEffectImports: true`.
- **Hard-deprecated (now errors)**: `target: es5`, `moduleResolution: "node"` and `"classic"` (use `nodenext` or `bundler`), `baseUrl` as resolution root, `outFile`, `module: amd/umd/system`, `esModuleInterop: false`. Advice built on these flags is dead advice.
- **TypeScript 7** is the native Go-port compiler (~10x faster), in beta via `@typescript/native-preview`. TS 6 is deliberately the migration bridge; `--stableTypeOrdering` (6.0) makes emitted type ordering deterministic for diffing against TS 7 output.
- TS 5.9 (common in the wild) added `import defer` and `--module node20`. Check the project's `typescript` version in `package.json` before answering config questions — TS 6 defaults don't hold on 5.x and vice versa.
- **`require(esm)` works on all supported Node LTS lines (22+)** — the dual-package hazard era is over; libraries targeting Node 22+ can ship ESM-only.

## Decision anchors

- **Generic?** Only if the output type depends on the input type AND the parameter appears in ≥2 positions or the return. A generic used once in a parameter equals its constraint — write the constraint. The generic that earns its keep pairs things that must stay in sync via lookup types: `on<K extends keyof Events>(name: K, fn: (e: Events[K]) => void)`.
- **Overloads vs conditional return types** — the tiebreaker Opus-class reasoning misses is *error quality*: overload errors list candidate signatures; conditional-type errors dump the unevaluated conditional at the caller. Choose overloads for ≤4 arbitrary shapes or when call-site errors matter; conditional types when the mapping is a uniform rule or callers pass generic `T`. Hybrid: permissive implementation signature + precise public overloads (the implementation signature is invisible to callers and only loosely checked — keep it honest).
- **`any`** is correct only in generic constraints for function/array shapes (`F extends (...args: any[]) => any`; `unknown[]` rejects real functions by contravariance). Everywhere else `unknown` + narrowing. Every `as` carries a comment stating the proof; `satisfies` for checked-but-narrow config.
- **Flags**: `noUncheckedIndexedAccess` on day one. `exactOptionalPropertyTypes` — decide at project start: retrofitting floods errors at every `obj.x = undefined` write site (fixes are per-site `delete` or `| undefined` widening; budget real time or don't flip it mid-project). `verbatimModuleSyntax` for any transpile-only pipeline.
- **Strictness migration order — the common recommendation is backwards.** The reflex (and Opus-cold answer) is "do `strictNullChecks` last because it's the biggest." Do it **first**: it finds the most real bugs per hour of migration work, and the later flags' fixes often touch the same lines — sequencing it last means re-editing files twice. Then `noImplicitAny`, then the rest. Mark existing errors with `@ts-expect-error` (self-cleaning: errors when the underlying error is fixed) and burn the count down in CI.

## The two priors for types-vs-runtime disagreements

1. **Narrowing "mysteriously fails"** → callback/mutation boundary reset; snapshot the narrowed whole into a `const` and close over that. Remaining cases: non-literal discriminant, or primitive unions needing a custom guard.
2. **Typechecks but throws at a module boundary** → believe the runtime: `node -e "import('pkg').then(m => console.log(Object.keys(m)))"`. For CJS deps, Node synthesizes named exports via cjs-module-lexer's *static* analysis; dynamically-built exports (`module.exports = buildApi()`) yield only `default`. Fix: `import pkg from 'legacy-csv'; const { parse } = pkg`. **The reflex fix — "enable `esModuleInterop`" — is wrong here**: interop flags change type-checking and emit, not Node's runtime named-export synthesis; the `.d.ts` was never the problem. Also rejected: `module: bundler` to make it typecheck (hides the mismatch on code that runs on Node directly).

## Pitfalls — checklist (explanations only where the cold answer fails)

Baseline traps, one line each (kept for completeness):
- Method-syntax members check bivariantly; use property syntax (`handle: (e: Animal) => void`) where soundness matters.
- Distributive conditionals: wrap `[T] extends [U]` to suppress.
- `const enum` breaks isolatedModules pipelines; use `as const` object + derived union.
- `readonly x: T[]` stops reassignment, not mutation (`readonly T[]` for that); all erased at runtime.
- `let` widens literals; fix at definition (`as const`/`satisfies`), not at use sites.
- Template-literal unions cap ~100k members; conditional recursion ~50 deep (~1000 tail) — hitting limits means move it to runtime.
- `Object.keys` is `string[]` by design; assert only on literals you constructed.

Delta traps — where cold answers are wrong or silent:
- **Module augmentation targeting.** `declare module 'express' { interface Request { user: User } }` silently no-ops when the interface actually lives in a *different package* — for Express, `Request` is declared in `express-serve-static-core`; augment the declaring package. Also silently no-ops in a file with no top-level import/export (script, not module) or one the compiler doesn't include. Checklist in code:

```ts
// src/types/express.d.ts — included via tsconfig "include"
import 'express';                            // 1. top-level import ⇒ MODULE, not script
declare module 'express-serve-static-core' { // 2. augment the package that DECLARES Request
  interface Request { user?: { id: string; roles: string[] } } // 3. optional: middleware may not have run
}
```

- **Assertion functions narrow only through explicitly-typed identifiers** — an aliased or method-accessed assertion call (`utils.assertFoo(x)` where `utils` isn't explicitly typed) silently doesn't narrow. Type predicates aren't inferred except trivial single-return guards (5.5+); annotate them.
- **`.includes()` on narrow arrays**: widen the array, not the value — `(list as readonly string[]).includes(x)`. The reflex cast on `x` destroys the literal type you wanted.
- **Writes through generic indexed access** (`o[k] = v` in `<T, K extends keyof T>`) are checked pessimistically; the honest fixes are a mapped-type setter or a commented assertion — not loosening `K`.
- **Global augmentation in test helpers leaks project-wide** (`declare global { var testDb: Db }`) — scope test globals to a `tsconfig.test.json` include.
- **`catch (e)` is `unknown` under strict; never `catch (e: Error)`.** Encode *expected* failures in return types (`Promise<Result<T, E>>` discriminated results) — `Promise<T>` says nothing about rejections.

## Worked micro-examples (the non-obvious ones)

**Type-level debugging:**
```ts
type Show<T> = { [K in keyof T]: T[K] } & {}; // hover Show<Mystery> to force full expansion
// Locate a mismatch: assign both directions; read the DEEPEST
// "Types of property 'x' are incompatible" line — TS errors read bottom-up.
declare const a: Actual; const _check: Expected = a;
// Type-test complex utilities in CI (a refactor silently broadening never→any is invisible otherwise):
import { expectTypeOf } from 'expect-type';
expectTypeOf(parseRoute('/users/:id')).toEqualTypeOf<{ id: string }>();
```

**Exhaustiveness that names the missed case:**
```ts
function assertNever(x: never): never { throw new Error(`Unhandled: ${JSON.stringify(x)}`); }
// default: return assertNever(event);  — adding a variant makes THIS line error at compile time
```

## Verification and stopping rule

1. Confirm TS version and `module`/`moduleResolution` first — a `nodenext` answer is wrong in a `bundler` project; a TS 6 default doesn't hold on 5.x.
2. Mentally erase the types: does the JavaScript still behave, especially where behavior *appears* type-dependent (discriminants vs `instanceof`, enums)?
3. Count the axioms: every `as`/`any`/`!` justified; external data passes a parser before gaining a type.
4. Check error-message quality at a representative wrong call site.

Stop when remaining `any`s are quarantined behind validated parsers and `strict` + `noUncheckedIndexedAccess` are clean. Chasing type purity in generated/vendored code, or golfing conditional types that work as overloads, is waste.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)

- Probed 14 claims: 8 baseline (cut/compressed), 4 partial (sharpened), 2 delta (expanded).
- Biggest baseline gaps: believes TS 5.9 is stable — no knowledge of TS 6.0's flipped defaults and hard-deprecated flags; prescribes `esModuleInterop` for CJS named-export runtime failures (types-level fix for a runtime lexer problem); orders `strictNullChecks` last in migrations.
- Opus cold is reliable on: narrowing mechanics, variance, distributive conditionals, `satisfies`, `const enum`, recursion limits — kept only as one-line checklist anchors.
