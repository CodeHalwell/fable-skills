---
name: compilers-and-parsing
description: Load when building anything that processes a language — parsers, interpreters, compilers, linters, formatters, config formats, template engines, query languages, or DSLs. Covers the lexing/parsing decision ladder, expression precedence, AST and visitor design, scoping and type checking, IR optimization, interpreter performance, error reporting, and embedded-vs-external DSL choice.
---

# Compilers and Parsing

Most of this domain is strong-model baseline (Pratt parsing, left-recursion, occurs check, UB-enables-optimization, eval-sandbox escapes, panic-mode recovery). This sheet keeps the architecture rules, the decision ladders, and the design-it-first items that can't be retrofitted.

## Architecture rules (violations cost a rewrite, not a patch)

- Pipeline of lossy summaries: text → tokens → AST → resolved/typed → IR → output; one job per stage. Smearing stages (parsing that evaluates, checking woven into codegen) works for v1 and makes v2 impossible — folding `2+3` inside `parse_expr` dies the day you need short-circuit semantics or error spans.
- **Spans from day one**: tokens carry `(start, end)` offsets; AST nodes carry the union of their tokens'; retrofitting spans onto a span-less AST is a full rewrite. Store offsets, not line/col (bloats, breaks under rewriting; wrong under tabs/UTF-8) — build a sorted line-start array once and `bisect` at print time. Pick one offset unit (Python: str code points) end-to-end; mixing byte and char offsets garbles spans on non-ASCII source.
- The AST is an API: one typed dataclass per construct, `span` mandatory; keep it syntactic — names/types go in separate tables or annotations, not in-place mutation (mutation kills re-analysis and source printing). Rewriting passes return new nodes; in-place stops composing at ~3 passes.
- Semantics before optimization: passes are legal exactly when they preserve observable behavior *for defined programs*. For a DSL, define everything (trap overflow, error on null): you don't need the last 10% of performance and your users need determinism; every UB you copy from C is an optimizer license to delete their safety checks.

## Decision ladders

- **Parsing tech by nesting, not familiarity**: flat tokens → regex/split; ANY nesting → never regex (mathematically non-regular, not merely inelegant); own small grammar → hand-written lexer + recursive descent (the production default: GCC, Clang, Go, Rust, TypeScript are all hand-written RD — error messages and recovery are why); medium grammar with a spec → Lark with `parser="lalr"` — earley mode swallows ambiguity silently, LALR conflicts are bugs surfaced early (prototype in earley, ship on lalr); someone else's real language → their parser (`ast`, `sqlglot`, tree-sitter); writing your own for a real language is a multi-month underestimate.
- **Embedded vs external DSL vs data format**: programmers-as-users + host available → embedded (free tooling, host escape hatch). Non-programmers, serialized artifacts, sandboxing, or static analyzability → external. Genuinely declarative data → JSON/YAML + schema — until it grows `if`/`for`/interpolation, the Helm/Ansible template-soup failure mode; that's the signal it wanted a real parser — budget it then, not three hacks later.
- **Tree-walk vs bytecode**: tree-walk (1–2 hours) is right when evaluation is dominated by the operations themselves; bytecode VM buys ~5–10× on arithmetic-heavy code (flat arrays, slot-indexed locals, no host recursion). Before rewriting: resolve names to (depth, slot) at parse time, compile nodes to closures (kills isinstance dispatch), cache lookups — recovers 3–5×. A tree-walker inherits the host's stack: deep user recursion crashes *you*; catch it or implement your own call stack.
- **Untrusted expressions**: sandboxing Python `eval` with a restricted namespace is a known-lost battle (`().__class__.__mro__`/`__subclasses__` escapes defeat namespace filtering). Your own small interpreter IS the sandbox — a primary reason DSL interpreters exist. Restricted-`eval` is acceptable only for trusted input.

## Mechanics with exact edges

- Precedence climbing: `rhs = parse_expr(prec + 1 if left_assoc else prec)` — that one token is the entire associativity mechanism; flipping it makes `2**3**2` = 64 instead of 512. Unary minus gets its own precedence in `parse_atom`, and `-2**2` is `-(2**2)` in Python — check the target language's spec. Test associativity with three-operand chains explicitly.
- Left recursion: `E → E '+' T` transcribed literally = infinite recursion on the first token; specs *favor* left recursion for left associativity, so every spec grammar needs the loop transform (`parse_T` + while) or Pratt before hand-implementation.
- Lexer: master regex ordered longest-first (`**` before `*`, `>=` before `>`); keywords by post-check on the identifier rule (separate keyword alternatives lex `ifx` as `if`+`x`); an `ERR` catch-all rule as the last alternative — `re.finditer` silently *skips* unmatched characters otherwise; emit an explicit EOF token (parsers without one special-case end-of-input in every function and one case is always wrong).
- Token-consumption contract, enforced globally: `parse_X` consumes exactly X; `expect(tok)` consumes-or-errors; `peek()` never consumes. Every parser bug cluster traces to a function that sometimes consumes lookahead and sometimes doesn't.
- Type checking: poison `Error` type that unifies with everything and never re-reports — the single trick preventing 50-error cascades. Inference = constraints + unification with the occurs check (`T = List[T]` must fail). Subtyping breaks pure unification (equations become directional) → bidirectional checking, not bolted-on subsumption. Calibration: monomorphic + annotations at function boundaries = 90% of the value at 10% of the complexity; add inference for locals only.
- Resolution: resolve once in a dedicated pass, annotate identifiers with (depth, slot); conscious forks — shadowing, hoisting, block vs function scope, capture by reference vs value (the `for i ... lambda: i` trap is by-reference capture of one loop binding; fresh binding per iteration is the fix). Closures need captured locals boxed/heap-allocated. Mutual recursion needs two passes: declare all top-level names, then resolve bodies.
- Optimization passes: folding must not change trap behavior (`1/0` stays) and must use target FP semantics; DCE needs an effects predicate, however crude — a "dead" call may write, throw, or not terminate; CSE needs purity AND no aliasing store between occurrences; passes feed each other — loop `fold; propagate; dce` to fixpoint. `ast.NodeTransformer` returning `None` silently deletes the node; forgetting `generic_visit` silently skips children — the linter that "misses nested cases" has this bug.
- Parser error recovery: report, skip to a synchronization point (statement keywords, `;`, `}`), resume — without it a missing brace on line 3 makes lines 4–500 unparseable and users fix one error per compile. Message bar: span + expected + found + remote cause ("unclosed '(' opened at 4:12").

## Verification / self-check

- Round-trip: parse → pretty-print → parse; ASTs must be equal — catches precedence/associativity mechanically.
- Differential: your evaluator vs host `eval` on 1000 grammar-generated random expressions; optimizer soundness = full test suite with each pass ON and OFF, diffing program outputs — any difference is an unsound pass.
- Adversarial always: empty input, lone operator, unclosed delimiters, 10⁴-deep nesting (graceful error, not segfault), strings containing your delimiters, `-2**2`-class cases, `>=`-vs-`>`+`=`, `ifx`, unicode identifiers.
- Half the test suite must be *rejection* tests asserting both refusal and error position — a parser that accepts everything passes every positive test.
- Scoping audit: shadowing, use-before-decl, closure-over-loop-variable, mutual recursion — four cases catch nearly all resolver bugs. Error-message audit: five representative typos → correct span and actionable expectation; off-by-one line numbers = newline counting at token boundaries (usually the newline-inside-string case).

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 14 claims: 14 baseline (cut/compressed), 0 partial, 0 delta.
- Opus 4.8 nailed everything: Pratt lbp/rbp associativity with -2**2, left-recursion fixes, maximal munch/keyword post-check/finditer silent-skip, poison types and the occurs check, bidirectional typing for subtyping, hand-written RD in GCC/Clang/Go/Rust/TS, 1/0-folding and DCE purity traps, UB with the CVE-2009-1897 null-check deletion, tree-walk vs bytecode with matching speedup numbers, eval-sandbox escape payloads, offsets + line-start bisect, panic-mode recovery, earley-vs-lalr, round-trip and differential testing, the loop-variable capture trap, two-pass resolution, and the Helm/Ansible config-language failure mode.
- No substantive gaps; retained value is the design-it-first items (spans, stage separation, consumption contract), the decision ladders, and rejection-test discipline.
