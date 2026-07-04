---
name: compilers-and-parsing
description: Load when building anything that processes a language — parsers, interpreters, compilers, linters, formatters, config formats, template engines, query languages, or DSLs. Covers the lexing/parsing decision ladder, expression precedence, AST and visitor design, scoping and type checking, IR optimization, interpreter performance, error reporting, and embedded-vs-external DSL choice.
---

# Compilers and Parsing

## Core mental model

- **A language processor is a pipeline of lossy summaries:** text → tokens → AST → resolved/typed AST → IR → output. Each stage should do ONE thing and produce a structure the next stage can consume without re-deriving. The most common architectural failure is smearing stages together (parsing that also evaluates, type checking woven into codegen) — it works for v1 and makes v2 impossible.
- **Choose parsing technology by the grammar's nesting, not by familiarity.** Regex handles regular languages only — the moment structures nest (parentheses, blocks, nested tags), regex is *mathematically* insufficient, not merely inelegant. The ladder below is a mechanical decision.
- **Errors are a first-class output, not an afterthought.** Real language tools spend more code on error reporting and recovery than on the happy path. Source locations must flow from the lexer through every stage — retrofitting spans onto an AST that lacks them is a full rewrite. Design the token and AST node to carry `(start, end)` offsets from day one.
- **The AST is an API.** Every later stage, plus tests, plus tooling, consumes it. Prefer many specific node types over stringly-typed generic nodes (`BinOp(op="+")` is fine; `Node(kind="binop", children=[...])` breeds `kind`-switch bugs the type checker can't catch).
- **Semantics before optimization.** Name resolution and type checking define what the program *means*; only then can optimization ask what may be changed without changing meaning. Optimizations are legal exactly when they preserve observable behavior *for defined programs* — which is why undefined behavior exists (see below).

## The lexing/parsing decision ladder

| Input shape | Tool | Notes |
|---|---|---|
| Flat tokens, no nesting (log lines, ISO dates, simple key=value) | Regex / `str.split` | Fine. Anchor patterns; use named groups |
| Nesting of ANY kind (parens, brackets, blocks, HTML/XML) | **Never regex.** | Balanced nesting is non-regular; the "it works on my examples" regex fails on depth or on strings containing delimiters. For HTML use an HTML parser (`html.parser`, lxml); for JSON use `json` |
| Small grammar you control, want great errors (config lang, expression evaluator, DSL) | Hand-written lexer + recursive descent | The professional default: total control over error messages and recovery; every major production compiler (GCC, Clang, Go, Rust, TypeScript, V8) uses hand-written recursive descent |
| Medium/large grammar, spec exists, speed of development matters | Parser generator / PEG library (Python: Lark, or `pyparsing` for small ones) | Lark's earley mode swallows ambiguity silently — prefer `parser="lalr"` to be *forced* to fix ambiguity; grammar conflicts are bugs surfaced early |
| Someone else's real language (Python, SQL, JS) | Use the existing parser (`ast` module, `sqlglot`, tree-sitter) | Writing your own parser for a real language is a multi-month underestimate — string escapes, unicode, and edge-case grammar will eat you |

Lexer specifics that bite: maximal munch (`>=` is one token — order alternatives longest-first, or in a master regex put `>=` before `>`); keywords vs identifiers (lex as identifier, then check a keyword set — don't make keywords separate regex alternatives or `ifx` lexes as `if` + `x`); string escapes handled in the lexer, once; track line/col by counting newlines at token boundaries, or store byte offsets and compute line/col lazily from a line-start index (cheaper and simpler).

## Expressions: ambiguity and precedence climbing

Grammar `E → E + E | E * E | (E) | num` is ambiguous — `1+2*3` has two parse trees. Resolve with precedence and associativity, implemented by **precedence climbing** (a.k.a. Pratt parsing) — one 15-line function replacing a cascade of `parse_addexpr → parse_mulexpr → ...` levels:

```python
PREC = {'+': (1, 'L'), '-': (1, 'L'), '*': (2, 'L'), '/': (2, 'L'), '**': (3, 'R')}

def parse_expr(ts, min_prec=1):
    lhs = parse_atom(ts)                      # num | '(' expr ')' | unary op
    while ts.peek() in PREC and PREC[ts.peek()][0] >= min_prec:
        op = ts.next()
        prec, assoc = PREC[op]
        rhs = parse_expr(ts, prec + 1 if assoc == 'L' else prec)
        lhs = BinOp(op, lhs, rhs, span=(lhs.span[0], rhs.span[1]))
    return lhs
```
The `prec + 1` for left-associative vs `prec` for right-associative is the entire associativity mechanism — flipping it makes `2**3**2` parse as `(2**3)**2` = 64 instead of 2⁹ = 512, and `a-b-c` as `a-(b-c)`. Test associativity explicitly with three-operand chains. Unary minus needs its own (high) precedence in `parse_atom`, and note `-2**2` is `-(2**2)` in Python — check the target language's spec, don't assume.

## AST design and the visitor question

- Node design: one dataclass per construct, fields typed, `span` mandatory. Keep the AST *syntactic* — resolve names and types into separate tables or later annotations rather than mutating parse output in place (mutation destroys the ability to re-run analysis or print original code).
- **Visitor pattern tradeoffs, honestly:** Visitors (double dispatch / `visit_NodeType` methods, like Python's `ast.NodeVisitor`) win when you have many operations over a stable set of node types — add a type checker, formatter, and interpreter without touching node classes. Methods-on-nodes win when node types change often and operations are few. The *expression problem* means you can't have both open. In Python, a third option often beats both: structural `match` statements (3.10+) per operation — `match node: case BinOp('+', l, r): ...` — exhaustive, readable, no framework. Pitfall with `NodeVisitor`: forgetting `generic_visit` means children are silently skipped — a linter that "misses" nested cases has this bug.
- Immutable-ish ASTs + rewriting passes that return new nodes (like `ast.NodeTransformer`) compose far better than in-place mutation once you have 3+ passes.

## Name resolution and scoping

- Core structure: a stack of scope dicts. `resolve(name)` walks outward; declaration inserts into the top. Push on block/function entry, pop on exit. Resolve *once* in a dedicated pass, annotating each identifier node with its declaration (or a (depth, index) slot) — re-resolving by name at every runtime access is both slow and semantically wrong for closures.
- The decisions you must make consciously (each is a language design fork, and defaulting silently causes bugs): shadowing allowed? use-before-declaration (hoisting)? does a block create a scope or only functions (JS `var` vs `let`)? closure capture by reference or by value (the Python `for i ... lambda: i` trap is capture-by-reference of a loop variable)?
- Closures need the resolver to mark captured variables — the interpreter must heap-allocate ("box") captured locals or capture the environment; stack-frame locals that die at return can't be captured. Missing this yields closures that see garbage or the *last* loop value.
- Forward references (mutual recursion between functions/types) require two passes: declare all top-level names first, then resolve bodies. Single-pass resolution rejects legal mutually-recursive programs.

## Type checking as constraint solving

- Simple explicit-types checking is a bottom-up AST walk: compute each expression's type, check against expectations at usage sites. Report errors with BOTH the offending span and the expected-vs-actual types; on error, assign a poison type `Error` that unifies with everything and is never re-reported — this single trick prevents the 50-error cascade from one typo.
- Type *inference* is constraint generation + unification: walk the AST emitting equations (`typeof(f) = typeof(arg) → T_fresh`), then solve by unification (recursively match type constructors, bind type variables, with the **occurs check** — `T = List[T]` must fail or you infer infinite types and loop). This is Hindley-Milner's core; you can implement a useful subset in ~150 lines.
- Practical calibration: for a DSL, monomorphic types + explicit annotations at function boundaries gives 90% of the value at 10% of the complexity of full inference. Add inference only for locals (like modern Java/C++ `var`/`auto`).
- Subtyping breaks pure unification (equations become one-directional constraints); if the DSL needs subtyping, use bidirectional type checking (check-mode vs infer-mode functions) instead of trying to bolt subsumption onto unification.

## IR and the optimization catalog

Optimize on an IR (three-address code, SSA, or even a simplified AST), not on source or final output. The core catalog and what *invalidates* each:

- **Constant folding** (`2*3` → `6`): invalidated by side-effectful or trapping operations (division by zero must still trap — folding `1/0` to a poison constant changes behavior), and by floating-point subtleties (folding must use the target's FP semantics; `0.1+0.2` folded at build time must equal runtime).
- **Dead code elimination:** removing a computation is legal only if it's *pure*; a "dead" call may write, throw, or not terminate. DCE therefore requires an effects analysis, however crude (a `has_side_effects(node)` conservative predicate). Also: code after `return` vs code that's dynamically unreachable — the first is syntactic, the second needs constant-propagation first (the passes feed each other; run to fixpoint).
- **Inlining:** the enabler optimization (exposes constants and DCE opportunities across call boundaries). Invalidated by recursion (bound the depth), and it changes observable behavior if the language exposes call stacks (`traceback`) or relies on function identity. Watch code-size blowup: inline small/hot, not everything.
- **Common subexpression elimination:** requires purity AND that no store between the two occurrences may alias the operands — aliasing analysis is why CSE is hard in languages with pointers and trivial in pure expression DSLs.
- The pass-ordering reality: folding exposes DCE, inlining exposes folding — production compilers run pass pipelines repeatedly. For a small compiler, a loop of `fold; propagate; dce` until no change is simple and effective.

**Why undefined behavior enables optimization:** UB is a *contract*: the compiler may assume UB never happens, so every program state that would trigger it can be assumed unreachable, and facts can be propagated backward from that assumption. Signed-overflow UB lets C compilers treat `i + 1 > i` as always-true and keep loop variables in registers with simple induction analysis; null-deref UB lets a dereference *prove* the pointer non-null and delete subsequent null checks (the famous kernel-bug pattern: `p->x; if (!p) return;` — the check is deleted). Design consequence for YOUR language: every behavior you define costs optimization freedom, every behavior you leave undefined costs user sanity. For a DSL, define everything (trap on overflow, error on null) — you don't need the last 10% of performance, and your users need determinism.

## Interpreters: tree-walk vs bytecode, performance reality

- **Tree-walk** (recursive `eval(node, env)`): 1–2 hours to write, perfect for DSLs, config evaluation, and anything where evaluation time is dominated by the operations themselves (I/O, numpy calls). Overhead: ~10–100× slower than CPython-level bytecode for tight loops, dominated by per-node dispatch and env dict lookups.
- **Bytecode VM** (compile AST → flat instruction list, loop-and-switch dispatch): ~5–10× faster than tree-walk for arithmetic-heavy code; required if user programs contain hot loops. The wins come from: flat arrays instead of pointer-chasing, slot-indexed locals instead of dict lookups (do the slot assignment in the resolver pass), and no Python-level recursion.
- Cheap tree-walk speedups before jumping to bytecode: resolve names to (depth, slot) at parse time; closure-compile nodes to Python closures (`compile_node` returns a lambda — removes the dispatch `isinstance` chain); cache method lookups. These can recover 3–5×.
- Recursion limit: a tree-walk interpreter inherits the host's stack — deep user recursion crashes the *interpreter*. Either implement your own call stack (bytecode VMs get this for free) or set limits and produce a proper "stack overflow" error for the user.
- Don't hand-roll if the host has it: for embedded expressions, compiling to Python `ast` and calling `compile()`/`eval` with a restricted namespace outperforms any interpreter you'll write — but ONLY for trusted input; sandboxing Python `eval` against adversaries is a known-lost battle (dunder escapes like `().__class__.__mro__` defeat namespace filtering). For untrusted input, your own small interpreter IS the sandbox — that's a primary reason DSL interpreters exist.

## Source locations and error messages — design it first

- Tokens carry `(start_offset, end_offset)`; AST nodes carry the union span of their tokens; every semantic error carries the span of the *most specific* relevant node plus, when applicable, a secondary span ("expected int because of the declaration here").
- Store offsets, not line/col; build a sorted array of line-start offsets once and `bisect` to render line/col only when printing an error. Storing line/col per token bloats and breaks under any source rewriting.
- Parser error recovery so users get more than one error per run: on failure, report, then skip tokens to a synchronization point (statement start keywords, `;`, `}`) and resume. Without recovery, a missing brace on line 3 makes lines 4–500 unparseable and users fix errors one compile at a time.
- The quality bar for messages: point at the span, say what was expected AND what was found, and when the cause is remote, show it ("unclosed '(' opened at 4:12"). "Syntax error" alone is a bug report generator.

## DSLs: when embedded beats external

- **Embedded DSL** (a Python API/fluent builder/operator overloading — like SQLAlchemy, pytest fixtures, Keras): free lexer/parser/editor-support/debugger, host escape hatch for anything unanticipated. Choose when users are programmers and the host language is available in their context.
- **External DSL** (own syntax + parser): choose when users are NOT programmers (analysts, designers, ops), when the artifact must be serialized/versioned/exchanged as data, when you need syntax the host can't express, or when execution must be sandboxed/limited (see eval above) or statically analyzable (termination guarantees, cost bounds — impossible for host-language snippets).
- The middle path that wins surprisingly often: define the language as a **data structure** (JSON/YAML/TOML with a schema, or Python dicts) — zero parser, trivially serializable, and validation via `jsonschema`/pydantic. Its ceiling: expressions and abstraction (variables, conditionals in YAML → the Helm/Ansible template-soup failure mode). When your data format grows `if`/`for`/string-interpolation, that's the signal it wanted to be a real DSL with a real parser — budget the recursive descent parser then, not three hacks later.

## Verification / self-check

- Round-trip test: parse → pretty-print → parse again; the two ASTs must be equal. Catches precedence and associativity bugs mechanically.
- Differential test expressions against the host: for arithmetic grammars, compare your evaluator against Python `eval` on 1000 random well-formed expressions (generate from the grammar). Disagreement = precedence/associativity/semantics bug.
- Adversarial inputs, always: empty input, a lone operator, unclosed delimiters, 10⁴-deep nesting (recursion check — parsers should error gracefully, not segfault), strings containing your delimiters and escapes, unicode identifiers if permitted.
- For each optimization pass: run the test suite with the pass ON and OFF and diff program *outputs* — any difference is a soundness bug in the pass (or exposed UB in the test).
- Error-message audit: for five representative typos, does the message include a correct span and an actionable expectation? Line numbers off by one = the lexer counts newlines wrong at token boundaries (usually the newline-inside-string case).
- Scoping audit: test shadowing, use-before-decl, closure-over-loop-variable, and mutual recursion explicitly — these four cases catch nearly all resolver bugs.
