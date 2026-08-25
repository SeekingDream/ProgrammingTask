# Task 6 — Static + Dynamic Escape Finder for Python Execution Sandboxes (`SandboxEscapeFinder`)

> Read `README.md` first. You may use any AI coding tool, but you must be able
> to explain every part of your code.
>
> This is a defensive-security research task: you are evaluating the
> robustness of an existing open-source sandbox, not attacking a live system.
> All dynamic tests must run against your own local instance of the target
> sandbox, isolated so an actual escape cannot affect the host running your
> test suite.

## Problem

Systems that execute untrusted or LLM-generated Python code (code-interpreter
tools, agent frameworks, online judges, notebook services) typically run it
inside a restricted execution environment. These sandboxes are frequently
broken by well-known classes of Python introspection tricks (e.g. reaching
`os` via `().__class__.__bases__[0].__subclasses__()`, or recovering
`__builtins__` through a function's `__globals__`). This task builds a small
program-analysis toolbox that (1) statically flags source code that looks
like it is attempting a known escape technique, and (2) dynamically fuzzes a
real sandbox instance with escape-attempt payloads to see which ones actually
break out.

## Background material (study before coding)

- **Target sandbox:** `RestrictedPython` —
  https://github.com/zopefoundation/RestrictedPython. Read its docs closely,
  including what it explicitly states it does and does not protect against.
- **Known escape-technique catalogs (for building your payload corpus):**
  - https://github.com/jailctf/pyjailbreaker
  - https://github.com/moshekaplan/python_sandbox_escapes
- Before coding, be able to name and explain at least 5 distinct escape
  technique classes from the catalogs above (e.g. `__subclasses__` traversal,
  `__globals__`/`__closure__` access, `__builtins__` restoration, generator /
  frame introspection, format-string attribute access).

## Objective

Build `SandboxEscapeFinder`: a static analyzer that scans arbitrary Python
source for suspected escape-technique usage, and a dynamic prober that runs a
payload corpus against a live `RestrictedPython` sandbox and empirically
determines which payloads actually escape.

## Required public API

```python
from sandbox_escape_finder import StaticAnalyzer, DynamicProber

analyzer = StaticAnalyzer(config)
findings = analyzer.scan(source_code)

prober = DynamicProber(sandbox_exec, oracle, config)
report = prober.run(payload_corpus)
```

| Name             | Type       | Meaning |
|------------------|------------|---------|
| `config`         | `dict` / JSON | Enabled technique detectors/payload categories, thresholds, seed. |
| `source_code`    | str        | Arbitrary Python source to statically vet. |
| `findings`       | `list[dict]` | One entry per match: technique name, matched AST node + source location, confidence. |
| `sandbox_exec`   | callable   | `code: str -> result`; a thin wrapper around `RestrictedPython`'s documented restricted-compile-and-exec setup. |
| `oracle`         | callable   | `(pre_state, exec_result, post_state) -> bool`; an automated check for whether a run achieved an out-of-sandbox effect (e.g. a canary file outside the allowed directory was written, a forbidden module got used, a marker value leaked out). |
| `payload_corpus` | iterable of str | Candidate payloads, seeded from the technique catalogs above, each attributed to its source technique. |
| `report`         | `list[dict]` | Per payload: static-analyzer verdict, whether it executed, whether the oracle confirmed an escape, timing. |

## Detailed requirements

1. **AST-based static analysis**, not regex-only matching. Detect at least 5
   distinct named technique classes; each finding must cite the technique
   name and the offending AST node's source location.
2. **Isolated dynamic execution.** Run payloads against a real, unmodified
   `RestrictedPython` instance, inside a subprocess or other isolated context,
   with any "escape target" (canary file, marker env var) pointed at a
   throwaway location — never a real system path — so a genuine escape cannot
   damage the host running your tests.
3. **Automated oracle, not manual inspection.** The oracle must be a callable
   check (e.g. "did this canary file outside the allow-listed directory get
   created/modified") that a test suite can assert on.
4. **Attributed payload corpus.** Seed at least 10 payloads from the
   technique catalogs, each attributed to its source technique; allow
   additional custom payloads via `config`.
5. **Required false-positive/negative analysis.** Report payloads the static
   analyzer flagged that did *not* escape in practice, and any payloads that
   escaped but were *not* statically flagged. This comparison is part of the
   deliverable, not optional.
6. **Config-driven and reproducible** via `seed`.

## Deliverables

- The `sandbox_escape_finder` package (installable via `pip install -e .`).
- `examples/quickstart.py` running both the analyzer and the prober against
  `RestrictedPython` with the seeded payload corpus.
- Tests: AST-detector unit tests with known positive/negative snippets, an
  oracle-correctness test using an intentionally-broken toy sandbox fixture
  that must be caught, and an end-to-end run of the full corpus against
  `RestrictedPython`.
- `README.md`: install steps, the usage snippet, a description of each
  detected technique class and the oracle design, and a results table
  (payload × technique × static-flag × dynamic-escape).

## Acceptance criteria

- The quickstart runs end-to-end and produces both `findings` and `report`.
- At least 5 technique classes are implemented with real positive/negative
  test cases.
- The dynamic prober runs the full corpus against `RestrictedPython` safely
  (no host side effects) and the report distinguishes statically-flagged vs.
  actually-escaping payloads.
- You can explain, for each technique, why `RestrictedPython`'s specific
  restriction is (or isn't) sufficient against it.

## Suggested difficulty: 3 / 5
Well-scoped against a single documented target with existing payload
catalogs to build from, but still requires real AST analysis, safe process
isolation, and a rigorous static-vs-dynamic comparison.
