---
name: typescript-mastery
description: Loads expert TypeScript judgment for designing types, debugging type errors, configuring tsconfig, and setting runtime validation boundaries. Use when writing non-trivial TypeScript, reviewing type-level code, fixing generics/variance/narrowing issues, untangling ESM/CJS interop, or deciding how strict a codebase should be.
---

# TypeScript Mastery

## Core mental model

1. **The type system is a proof assistant over a language that doesn't care.** Types are erased; nothing is checked at runtime. Every `as`, `any`, and unvalidated `JSON.parse` is an axiom you asserted without proof. Expert TypeScript is mostly about minimizing axioms and pushing them to the edges (I/O boundaries), so the interior of the program is fully proven.
2. **Typing is structural, and that cuts both ways.** Two types with the same shape are the same type. Consequence A: you never need `implements` for compatibility. Consequence B: excess properties survive (`{a, b}` assigns to `{a}`), IDs of different entities interchange freely, and empty-ish types (`{}`, interfaces with only optional members) match almost everything. When identity matters, brand it: `type UserId = string & { readonly __brand: 'UserId' }`.
3. **Narrowing is control-flow analysis, and it has known blind spots.** The checker tracks refinements per variable, per scope. It resets narrowing across function boundaries (callbacks), after `await` for mutable bindings it can't prove stable, and it never narrows through a property access chain stored in a mutable object. When narrowing "mysteriously fails," the answer is almost always: copy to a `const` local first.
4. **Discriminated unions are the primary modeling tool.** Model states, not fields. `{status: 'loading'} | {status: 'ok', data: T} | {status: 'error', error: E}` makes illegal states unrepresentable and gives you exhaustiveness checking for free. If you find yourself writing `data?: T; error?: E; loading: boolean`, stop — you've created 8 states where 3 exist.
5. **Complexity budget: every type-level trick costs future readers.** The escalation ladder is: plain types → generics with constraints → mapped types → conditional types → recursive template-literal types. Only climb a rung when the rung below forces duplication that will actually drift. A conditional type that saves three overload signatures is good; one that saves one is over-engineering.

## Current state (verified July 2026)

- Stable TypeScript is **6.0.x**. TS 6.0 flipped defaults: `strict: true`, `module: "esnext"`, floating `target` (es2025), `types: []` (you must add `"types": ["node"]` explicitly), `noUncheckedSideEffectImports: true`. It hard-deprecates `target: es5`, `moduleResolution: "node"`/`"classic"` (use `nodenext` or `bundler`), `baseUrl`-as-resolution-root, `outFile`, and `esModuleInterop: false`.
- **TypeScript 7** is the native Go-port compiler (~10x faster), in beta; preview via `@typescript/native-preview` on npm. TS 6 is deliberately the migration bridge — avoid deprecated flags now and the 7 upgrade is mechanical.
- TS 5.9 (still common in the wild) added `import defer` and `--module node20`. Don't recommend flags newer than the project's compiler; check `package.json` first.

## Decision frameworks with reasoning chains

**"Should this be a generic?"** Ask in order: (1) Does the output type depend on the input type? No → no generic; a union or `unknown` suffices. (2) Is the type parameter used in at least two positions (or in the return)? A generic used once in a parameter and nowhere else is identical to its constraint — write the constraint directly (`function f(x: HasId)`, not `function f<T extends HasId>(x: T)` unless you return `T`). (3) Will callers get worse inference or worse error messages? Conditional-type return values produce terrible errors at call sites; overloads often read better even though they're "less elegant." What changes the decision: if the duplication the generic removes is between things that must stay in sync (e.g., an event-name-to-payload map), the generic earns its cost via a lookup type `<K extends keyof Events>(name: K, payload: Events[K])`.

**"Interface or type alias?"** Default `type` for unions, tuples, function types, and anything computed. Use `interface` when you specifically want declaration merging (augmentation targets, library extension points) or extending class hierarchies. Performance differences are no longer a deciding factor at normal scale. Do not export an interface as a public extension point unless you intend third parties to merge into it — merging is a feature and a hazard.

**"Where does runtime validation go?"** The rule: validate at trust boundaries, trust the types inside. Boundaries = HTTP bodies, queue messages, env vars, DB rows if the schema isn't code-generated, `localStorage`, third-party SDK responses typed as `any`. Use zod (or valibot for bundle-size-sensitive edges) and derive the static type from the schema — `type User = z.infer<typeof UserSchema>` — never write the type and schema separately, they will drift. Inside the boundary, adding more zod parsing is waste and a smell that you don't trust your own types.

**"`unknown` vs `any` vs assertion?"** `any` is only acceptable as an *input* constraint in generic bounds where it means "anything" variance-safely (e.g., `(...args: any[]) => any` in a generic constraint — using `unknown[]` there breaks matching against concrete functions due to parameter contravariance). Everywhere else: `unknown` + narrowing. A type assertion `as T` is acceptable when you hold a proof the compiler can't express (e.g., just validated with a schema, or an exhaustive `Object.keys` situation) — and each one deserves a comment stating the proof.

**tsconfig flags that actually matter** (beyond `strict`): `noUncheckedIndexedAccess` (the single highest-value non-default flag — makes `arr[i]` return `T | undefined`), `exactOptionalPropertyTypes` (distinguishes "absent" from "present-but-undefined"; enable at project start, painful to retrofit), `verbatimModuleSyntax` (forces `import type`, kills ESM/CJS ambiguity), `noPropertyAccessFromIndexSignature`. Skip `noImplicitReturns` debates; it's cheap, just turn it on.

## How an expert thinks through it: "my narrowing disappeared"

Scenario: `if (config.mode === 'batch') { items.forEach(i => process(config.batchSize, i)) }` — errors inside the callback: `config.batchSize` possibly undefined.

Internal monologue: *The narrowing worked on the `if` line, so the discriminated union is fine. First hypothesis: callback boundary. TypeScript discards narrowing of anything mutable when it crosses into a function that might run later — `forEach` runs synchronously, but the checker doesn't special-case that; any callback resets narrowing for non-`const` references. Wait — `config` is a `const` binding, but the narrowing is on a property access, `config.mode`. Property narrowing survives into callbacks only if the checker can prove the object isn't mutated... which it can't for a parameter. Options: (a) assert inside the callback — no, that's an unproven axiom that breaks when someone reorders code; (b) restructure the union so I destructure once — `const { mode } = config` doesn't help because `batchSize` is on the union too; (c) the right fix: narrow to a local — `if (config.mode === 'batch') { const batchConfig = config; ... }` — assignment to a fresh `const` captures the narrowed type, and locals keep their type inside closures. Take (c); it's one line and self-documenting.* Prior to internalize: **~80% of "narrowing broke" reports are callback/mutation-boundary resets; the fix is a `const` snapshot, not a smarter type.**

## Failure modes and pitfalls

- **Method-syntax variance hole.** `interface Handler { handle(e: Animal): void }` — method shorthand is *bivariant*: a `handle(e: Dog)` implementation is accepted unsoundly. Property syntax `handle: (e: Animal) => void` is properly contravariant under `strictFunctionTypes`. Declare callback members with property syntax in any interface where soundness matters.
- **Function parameter contravariance surprises.** `(x: Dog) => void` is NOT assignable to `(x: Animal) => void`. When someone asks "why can't I pass my specific handler," the answer is variance, and the fix is to make the handler generic or accept the broader type and narrow inside — not to cast.
- **`Object.keys` returns `string[]`, deliberately.** Because of structural typing, an object may have more properties than its type says, so `keys(x).forEach(k => x[k])` would be unsound. If you control the object literal, `for (const k of Object.keys(o) as (keyof typeof o)[])` is a justified assertion; say so in a comment.
- **Enums and `const enum`: avoid both in libraries.** `const enum` breaks under `isolatedModules`/transpile-only pipelines (esbuild, swc) because it requires cross-file type information. Prefer `as const` object + `type Mode = typeof Mode[keyof typeof Mode]` — pure erasable syntax, and required if anyone runs your code with Node's built-in type stripping.
- **Assertion functions and type predicates must be annotated.** `function isFoo(x): x is Foo` and `function assertFoo(x): asserts x is Foo` are not inferred (return-type-position `asserts` requires explicit annotation; predicate inference exists since 5.5 only for trivial single-return cases). Also, an assertion function called via a method or aliased import may not narrow — the checker requires an explicitly-typed identifier reference.
- **`exactOptionalPropertyTypes` retrofit trap.** Enabling it late floods you with errors where code writes `obj.x = undefined` into `x?: T`. The fix is `delete obj.x` or widening to `x?: T | undefined` — budget real time; don't enable it casually mid-project.
- **Declaration merging misuse.** Augmenting third-party types (`declare module 'express' { interface Request { user: User } }`) must live in a `.d.ts` or a file with at least one top-level import/export, and the module name must match the *resolved* specifier. The classic failure: augmentation silently not applying because the file is a script (no imports) or targets `express` while types come from `express-serve-static-core` — augment the package that actually declares the interface.
- **ESM/CJS interop, the persistent pain.** Under `module: nodenext`, a package with `"type": "commonjs"` imported from ESM gives you only the default export as the namespace in some setups; `import { thing } from 'cjs-pkg'` may typecheck (via `esModuleInterop` synthesis) but throw at runtime because named-export detection (cjs-module-lexer) failed on that package. Rules: trust the runtime, not the types, for CJS named imports; use `verbatimModuleSyntax` so `import type` never emits; never write `require()` in a `.mts` file; check `"exports"` maps have both `"types"` and matching `"import"`/`"require"` conditions, with `"types"` first. Note `require(esm)` now works at runtime in all supported Node LTS lines (22+), which relieves dual-package pressure — shipping ESM-only is now viable for libraries targeting Node 22+.
- **Template-literal-type blowup.** `` `${A}-${B}-${C}` `` with unions multiplies (union sizes multiply; the compiler errors around 100k members). Recursive conditional types have a depth limit (~50 for non-tail; ~1000 tail-recursive since 4.5). When you hit "Type instantiation is excessively deep," the answer is usually to stop computing at the type level and validate at runtime instead.
- **Branded types leaking construction.** A brand is worthless if every module casts into it. Give the brand exactly one constructor function that validates, and grep-audit for `as UserId` in review.

## Worked micro-examples

**Exhaustiveness with a well-typed helper** (the `never` trick, done so the error message names the missed case):
```ts
function assertNever(x: never, msg = `Unhandled: ${JSON.stringify(x)}`): never {
  throw new Error(msg);
}
switch (event.kind) {
  case 'created': return onCreate(event);
  case 'deleted': return onDelete(event);
  default: return assertNever(event); // compile error appears HERE when a variant is added
}
```

**Schema-first boundary with zod:**
```ts
import { z } from 'zod';
const Env = z.object({
  PORT: z.coerce.number().int().default(3000),
  DATABASE_URL: z.string().url(),
});
export const env = Env.parse(process.env); // crash at startup, not at first query
export type Env = z.infer<typeof Env>;     // single source of truth
```

**Type-level debugging.** To see what a complex type actually resolved to, force the compiler to expand it: `type Show<T> = { [K in keyof T]: T[K] } & {}` (hover `Show<Mystery>`). To find *why* two types mismatch, assign in both directions with scratch variables and read which property the error names; the deepest `Types of property 'x' are incompatible` line is the real cause — the top line of a TS error is the summary, read errors bottom-up.

## Verification and stopping rule

Before presenting TypeScript advice or code: (1) confirm the project's TS version and module settings — a `nodenext` answer is wrong for a `bundler` project and vice versa; (2) mentally erase the types and check the JavaScript still behaves (types don't exist at runtime — especially for `instanceof` vs discriminant checks and enum tricks); (3) count your axioms: every `as`, `any`, and `!` needs a stated justification; (4) if a type-level construct took more than a few minutes to understand, write the two-line overload/duplicated version and compare — pick whichever a reviewer reads faster. Stop hardening types when remaining `any`s are quarantined at boundaries behind validated parsers; chasing 100% type-purity inside vendored or generated code is waste.
