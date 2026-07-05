---
name: typescript-mastery
description: Loads expert TypeScript judgment for designing types, debugging type errors, configuring tsconfig, and setting runtime validation boundaries. Use when writing non-trivial TypeScript, reviewing type-level code, fixing generics/variance/narrowing issues, untangling ESM/CJS interop, or deciding how strict a codebase should be.
---

# TypeScript Mastery

## Core mental model

1. **The type system is a proof assistant over a language that doesn't care.** Types are erased; nothing is checked at runtime. Every `as`, `any`, and unvalidated `JSON.parse` is an axiom you asserted without proof. Expert TypeScript is mostly about minimizing axioms and pushing them to the edges (I/O boundaries), so the interior of the program is fully proven.
2. **Typing is structural, and that cuts both ways.** Two types with the same shape are the same type. Consequence A: you never need `implements` for compatibility. Consequence B: excess properties survive aliasing (`{a, b}` assigns to `{a}` through a variable), IDs of different entities interchange freely, and empty-ish types (`{}`, interfaces with only optional members) match almost everything. When identity matters, brand it: `type UserId = string & { readonly __brand: 'UserId' }`.
3. **Narrowing is control-flow analysis, and it has known blind spots.** The checker tracks refinements per reference, per scope. It discards narrowing across function boundaries (callbacks), for mutable bindings it can't prove stable, and for property chains on parameters. When narrowing "mysteriously fails," the answer is almost always: snapshot to a `const` local first.
4. **Discriminated unions are the primary modeling tool.** Model states, not fields. `{status: 'loading'} | {status: 'ok', data: T} | {status: 'error', error: E}` makes illegal states unrepresentable and gives exhaustiveness checking for free. If you find yourself writing `data?: T; error?: E; loading: boolean`, stop — you created 8 representable states where 3 exist.
5. **Every type-level trick spends a complexity budget.** The escalation ladder: plain types → generics with constraints → mapped types → conditional types → recursive template-literal types. Climb a rung only when the rung below forces duplication that will actually drift. A conditional type that keeps an event map and its handlers in sync earns its cost; one that saves a single overload does not.
6. **Types are for readers as much as for the checker.** A signature a caller can't predict the result of (deep conditional return types) fails at its main job even if it's sound. Prefer the form that produces the better *error message* at the call site.

## Current state (verified July 2026)

- Stable TypeScript is **6.0.x**. TS 6.0 flipped defaults:
  - `strict: true`, `module: "esnext"`, floating `target` (currently es2025), `types: []` (add `"types": ["node"]` explicitly or Node globals vanish), `noUncheckedSideEffectImports: true`.
  - Hard-deprecated (now errors): `target: es5`, `moduleResolution: "node"` and `"classic"` (use `nodenext` or `bundler`), `baseUrl` as resolution root, `outFile`, `module: amd/umd/system`, `esModuleInterop: false`.
- **TypeScript 7** is the native Go-port compiler (~10x faster), in beta; preview via `@typescript/native-preview` on npm. TS 6 is deliberately the migration bridge — avoid the deprecated flags now and the 7 upgrade is mechanical. `--stableTypeOrdering` (6.0) makes emitted type ordering deterministic for diffing against TS 7 output.
- TS 5.9 (still common in the wild) added `import defer` and `--module node20`. Never recommend flags newer than the project's compiler — check the `typescript` version in `package.json` before answering config questions.

## Decision frameworks with reasoning chains

**"Should this be a generic?"** Ask in order:
1. Does the *output* type depend on the *input* type? No → no generic; a union or `unknown` suffices.
2. Is the type parameter used in at least two positions, or in the return? A generic used once in a parameter and nowhere else is identical to its constraint — write the constraint directly (`function f(x: HasId)`, not `function f<T extends HasId>(x: T)` unless you return `T`).
3. Will callers get worse inference or worse errors? Conditional-type returns produce terrible call-site errors; two or three overloads often read better despite being "less elegant."
4. What would change the answer: if the duplication removed is between things that must stay in sync — an event-name-to-payload map, a table schema and its row type — the generic earns its keep via lookup types: `on<K extends keyof Events>(name: K, fn: (e: Events[K]) => void)`.

**"Interface or type alias?"** Default `type` for unions, tuples, function types, and anything computed. Use `interface` when you specifically want declaration merging (augmentation targets, extension points) or class hierarchies. Performance is no longer a deciding factor at normal scale. Do not export an interface as a public extension point unless you *intend* third parties to merge into it — merging is a feature and a hazard.

**"Where does runtime validation go?"** Validate at trust boundaries; trust the types inside.
- Boundaries: HTTP bodies, queue messages, env vars, DB rows (unless codegen'd), `localStorage`, webhook payloads, anything typed `any` by an SDK.
- Use zod (or valibot where bundle size matters) and derive the static type from the schema — `type User = z.infer<typeof UserSchema>`. Never maintain the type and the schema separately; they will drift and the type will win the argument while the schema loses the data.
- Inside the boundary, additional parsing is waste and a signal you don't trust your own types. One parse per datum per entry.

**"`unknown` vs `any` vs assertion?"**
- `any` is acceptable in exactly one habitat: generic *constraints* meaning "any function/array shape," e.g. `F extends (...args: any[]) => any` — using `unknown[]` there rejects concrete functions because parameters check contravariantly.
- Everywhere else: `unknown` + narrowing (type guards, schema parse).
- `as T` is acceptable when you hold a proof the compiler can't express (just-validated data, `Object.keys` on a literal you own). Each assertion deserves a comment stating the proof; an uncommented `as` in review is a finding.
- `satisfies` is the tool people reach past: `const config = {...} satisfies Config` checks conformance while *preserving* the narrower inferred type. Use it instead of `: Config` annotations when you want both checking and precise literal types downstream.

**tsconfig flags that actually matter** (beyond `strict`):
- `noUncheckedIndexedAccess` — highest-value non-default flag; `arr[i]` becomes `T | undefined`, killing a whole out-of-bounds bug class. Turn on at project start.
- `exactOptionalPropertyTypes` — distinguishes "absent" from "present-but-undefined"; brutal to retrofit (every `obj.x = undefined` write errors), so decide early.
- `verbatimModuleSyntax` — forces `import type`, removes emit ambiguity; essential for transpile-only pipelines (esbuild, swc, Node type stripping).
- `noPropertyAccessFromIndexSignature` — makes index-signature access visibly dynamic (`obj["key"]`).
- Not worth debating: `noImplicitReturns`, `noFallthroughCasesInSwitch` — cheap, enable and move on.

## How an expert thinks through it: "my narrowing disappeared"

Scenario: `if (config.mode === 'batch') { items.forEach(i => process(config.batchSize, i)) }` — error inside the callback: `config.batchSize` is possibly `undefined`.

Internal monologue: *The narrowing worked on the `if` line — the discriminated union is fine — so this is a persistence problem, not a modeling problem. First hypothesis: callback boundary. The checker discards narrowing of anything potentially-mutable when control flow enters a function that could run later; `forEach` runs synchronously but the checker doesn't special-case that. But wait — `config` is a `const` binding, shouldn't it survive? The narrowing is on `config.mode`, a* property *— property narrowing survives into closures only when the checker can prove the object is never mutated, which it can't for a function parameter. Options: (a) non-null assert inside the callback — rejected: unproven axiom, silently wrong when someone reorders code; (b) destructure `const { mode, batchSize } = config` before the check — works only if the union discriminates on destructured locals (it does, since 4.6, for destructured discriminants) but here `batchSize` doesn't exist on all variants so destructuring won't typecheck; (c) snapshot the narrowed whole: `if (config.mode === 'batch') { const batch = config; items.forEach(i => process(batch.batchSize, i)) }` — a fresh `const` captures the narrowed type and `const` locals keep their narrowing inside closures. Take (c): one line, self-documenting, robust to reordering.*

Prior to internalize: **~80% of "narrowing broke" reports are callback or mutation-boundary resets; the fix is a `const` snapshot, not a cleverer type.** The remaining 20%: narrowing on a non-literal discriminant (discriminants must be literal types, not `string`), or `in`/`instanceof` checks on unions of primitives where a custom guard is needed.

## Failure modes and pitfalls

- **Method-syntax variance hole.** `interface Handler { handle(e: Animal): void }` — method shorthand checks *bivariantly*: an implementation with `handle(e: Dog)` is accepted, unsoundly. Property syntax `handle: (e: Animal) => void` is properly contravariant under `strictFunctionTypes`. Declare callback-bearing members with property syntax anywhere soundness matters.
- **Function parameter contravariance surprises.** `(x: Dog) => void` is NOT assignable where `(x: Animal) => void` is expected. When someone asks "why can't I register my specific handler," the answer is variance; the fix is to accept the broad type and narrow inside, or make the registry generic — never a cast.
- **`Object.keys` returns `string[]` on purpose.** Structural typing means a value may have more properties than its type admits, so `keys(x) as (keyof T)[]` is only justified for object literals you constructed. If you own it, assert with a comment; if it arrived from elsewhere, iterate defensively.
- **Enums: avoid `const enum` in anything transpiled file-by-file.** `const enum` requires cross-file type information, so it breaks under `isolatedModules` pipelines (esbuild, swc, Node's type stripping). Prefer the erasable idiom: `const Mode = { Fast: 'fast', Safe: 'safe' } as const; type Mode = typeof Mode[keyof typeof Mode]`.
- **Type predicates and assertion functions.** `x is Foo` and `asserts x is Foo` return positions are not inferred except for trivial single-return guards (5.5+); annotate them. An assertion function narrows only when called through an explicitly-typed identifier — aliased or method-accessed assertion calls silently don't narrow.
- **Distributive conditional types firing on unions.** `type Box<T> = T extends unknown ? T[] : never` applied to `A | B` gives `A[] | B[]`, not `(A|B)[]` — conditional types distribute over naked type parameters. Wrap to suppress: `[T] extends [unknown] ? T[] : never`. Symptom: "my conditional type returns a weird union."
- **`exactOptionalPropertyTypes` retrofit trap.** Enabling it late floods errors where code writes `undefined` into `x?: T`. Fixes are `delete obj.x` or widening to `x?: T | undefined` per site — budget real time or don't flip it mid-project.
- **Declaration merging / module augmentation misfires.** `declare module 'express' { interface Request { user: User } }` silently does nothing when: the file has no top-level import/export (it's a script, so `declare module` means something else), or the interface actually lives in a different package (`express-serve-static-core`) — augment the module that *declares* the interface, in a file the compiler includes.
- **ESM/CJS interop, the persistent pain.** Under `module: nodenext`, `import { thing } from 'cjs-pkg'` can typecheck yet throw at runtime: named-export synthesis for CJS depends on cjs-module-lexer's static analysis, which fails on some packages. Rules: for CJS deps, trust the runtime over the types — test the import shape; keep `verbatimModuleSyntax` on; in `"exports"` maps put `"types"` first with matching `"import"`/`"require"` conditions; never `require()` in `.mts`. Runtime relief: `require(esm)` now works on all supported Node LTS lines (22+), so libraries targeting Node 22+ can ship ESM-only and skip dual-package hazards entirely.
- **Template-literal and recursion blowups.** Unions multiply through `` `${A}-${B}` `` (sizes multiply; the compiler bails near 100k members); recursive conditional types hit depth limits (~50 non-tail, ~1000 tail-recursive). "Type instantiation is excessively deep" usually means: stop computing this at the type level, validate at runtime instead.
- **Branded types with leaky construction.** A brand is worthless if modules cast into it ad hoc. Exactly one validating constructor per brand; grep for `as UserId` in review.
- **`readonly` is shallow and erased.** `readonly x: T[]` prevents reassigning `x`, not mutating the array — that needs `readonly T[]`. And nothing stops JS callers at runtime; `Object.freeze` if it truly matters.

## Worked micro-examples

**Exhaustiveness with an error message that names the missed case:**
```ts
function assertNever(x: never, msg = `Unhandled: ${JSON.stringify(x)}`): never {
  throw new Error(msg);
}
switch (event.kind) {
  case 'created': return onCreate(event);
  case 'deleted': return onDelete(event);
  default: return assertNever(event); // adding a variant makes THIS line error at compile time
}
```

**Schema-first boundary (zod), single source of truth:**
```ts
import { z } from 'zod';
const Env = z.object({
  PORT: z.coerce.number().int().default(3000),
  DATABASE_URL: z.string().url(),
});
export const env = Env.parse(process.env); // crash at startup, not at first query
export type Env = z.infer<typeof Env>;     // type derived FROM schema, never parallel to it
```

**Precise-yet-checked config with `satisfies`:**
```ts
const routes = {
  home: '/',
  user: '/users/:id',
} satisfies Record<string, `/${string}`>;
// typeof routes.user is the literal '/users/:id' (annotation `: Record<...>` would widen to string)
type RouteName = keyof typeof routes; // 'home' | 'user' — derived, not duplicated
```

**Type-level debugging techniques:**
```ts
// 1. Force full expansion to see what a type resolved to (hover Show<Mystery>):
type Show<T> = { [K in keyof T]: T[K] } & {};
// 2. Locate a mismatch: assign in both directions and read the DEEPEST
//    "Types of property 'x' are incompatible" line — TS errors read bottom-up.
declare const a: Actual; const _check: Expected = a;
// 3. Unit-test types in CI:
import { expectTypeOf } from 'expect-type';
expectTypeOf(parseRoute('/users/:id')).toEqualTypeOf<{ id: string }>();
```

## Verification and stopping rule

Before presenting TypeScript advice or code:
1. Confirm the project's TS version and `module`/`moduleResolution` — a `nodenext` answer is wrong in a `bundler` project and vice versa; a TS 6 default doesn't hold on 5.x.
2. Mentally erase the types and check the JavaScript still behaves — especially anywhere behavior *appears* to depend on types (discriminants vs `instanceof`, enum values, decorators).
3. Count the axioms: every `as`, `any`, and `!` must carry a stated justification, and unvalidated external data must pass a parser before gaining a type.
4. If a type-level construct took more than a few minutes to understand, write the boring duplicated version and compare; ship whichever a reviewer parses faster.
5. Check the error-message quality at a representative wrong call site — a sound API with unreadable errors will be `any`-ed around by teammates.

Stopping rule: stop hardening when remaining `any`s are quarantined at boundaries behind validated parsers and `strict` + `noUncheckedIndexedAccess` are clean. Chasing type purity inside generated or vendored code, or golfing conditional types that already work as overloads, is waste.
