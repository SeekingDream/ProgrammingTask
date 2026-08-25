# Task 5 — Adversarial Probing & Intermediate-State Dumping for DNN Verifiers (`VerifierProbe`)

**Difficulty: 5 / 5**

> Read `README.md` first. You may use any AI coding tool, but you must be able
> to explain every part of your code.

## Problem

A formal DNN verifier certifies that a network has no adversarial example
inside some input region `X` — but that certificate is only as good as the
verifier's internal numeric model. When a verifier reasons over the network in
exact/real-valued arithmetic (or a floating-point model that doesn't exactly
match hardware floating point), there can be a gap between what the verifier
"believes" and what the network actually computes. This task reproduces that
class of finding on a real verifier, and builds tooling to inspect what a
verifier is doing internally on a given `(model, input)`.

## Background material (study before coding)

- **Paper:** "Exploiting Verified Neural Networks via Floating Point Numerical
  Error" (Jia & Rinard) — https://arxiv.org/abs/2003.03021. Understand the
  paper's core mechanism: how floating-point rounding / numerical error
  accumulation in a verifier's internal computation can let a genuinely
  mispredicted input pass as "certified safe."
- Pick **one** verifier to target, from:
  - `MIPVerify.jl` (MILP-based, complete, Julia) —
    https://github.com/vtjeng/MIPVerify.jl. Note: a fork instrumented by the
    paper's author exists at https://github.com/jia-kai/MIPVerify.jl and is
    useful reading, but do not just copy it — build your own instrumentation
    and understand what it does.
  - `α,β-CROWN` (linear relaxation + branch-and-bound, complete, Python,
    repeated VNN-COMP winner) —
    https://github.com/Verified-Intelligence/alpha-beta-CROWN
  - `ERAN` (abstract interpretation / DeepPoly, sound but incomplete, Python +
    C) — https://github.com/eth-sri/eran
- Before coding, be able to explain: what "sound" and "complete" mean for your
  chosen verifier, and what specific internal computation (LP/MILP relaxation
  bound, floating-point interval, or abstract-domain coefficients) you will
  instrument.

## Objective

Build `VerifierProbe`, a toolbox with two capabilities against your chosen
verifier:

1. **Attack** — given a `(model, verifier, X)` where the verifier claims no
   adversarial example exists in `X`, search for a concrete input that the
   verifier still certifies safe but that the model actually mispredicts under
   real floating-point execution. The search must be grounded in the paper's
   mechanism (e.g. probing near branch-and-bound leaf nodes / relaxation
   boundaries where rounding error is most likely to matter) — not blind
   random sampling.
2. **Introspection** — for a given `(model, verifier, input)`, dump the
   verifier's own intermediate computation (e.g. per-layer bound tensors,
   per-node LP/MILP relaxation values, or abstract-domain coefficients) into a
   structured, documented trace.

## Required public API

```python
from verifier_probe import Attacker, Prober

atk = Attacker(model, verifier, config)
adv_x, atk_logs = atk.run(X)

prober = Prober(verifier)
trace = prober.dump(model, x)
```

| Name       | Type                    | Meaning |
|------------|-------------------------|---------|
| `model`    | DNN (small, e.g. MNIST/CIFAR-scale FC or conv net — these verifiers do not scale to LLM-size models) | The network under test. |
| `verifier` | adapter object          | A thin wrapper you write around the chosen verifier exposing `certify(model, X) -> Certificate` plus access to internal state. |
| `config`   | `dict` / JSON           | Search strategy, budget, numerical-tolerance assumptions, seed. |
| `X`        | input region            | e.g. center point + L∞ radius. |
| `adv_x`    | input                   | An input the verifier certifies safe but the model mispredicts — or, if none is found within budget, the closest near-miss. |
| `atk_logs` | dict                    | Search trajectory, the verifier's claim vs. the model's real output on `adv_x`, and whether a true soundness violation was found. |
| `x`        | single input            | One concrete input to introspect. |
| `trace`    | dict / structured, JSON-serializable | Ordered list of intermediate verifier states (per-layer/node id, bound or relaxation values). |

## Detailed requirements

1. **Verifier adapter.** Implement `certify()` plus internal-state access for
   your chosen verifier. This will likely require instrumenting the
   verifier's own source (it wasn't built for introspection) — document
   exactly what you hooked and why it does not change the verifier's
   semantics.
2. **Grounded search strategy.** Justify the attack strategy in terms of the
   paper's mechanism; state what "numerical gap" you are trying to exploit for
   your specific verifier.
3. **Honest negative results are acceptable.** If your chosen verifier (e.g. a
   sound abstract-interpretation tool like ERAN) resists the attack within a
   reasonable compute budget, `atk_logs` must still report the closest
   near-miss and explain why — this is an expected and valid finding, not a
   failed deliverable.
4. **Documented trace schema.** Every field in `trace` must be documented
   (meaning, units/shape) for your chosen verifier.
5. **Config-driven and reproducible** via `seed`.

## Deliverables

- The `verifier_probe` package (installable via `pip install -e .`).
- `examples/quickstart.py` demonstrating both `Attacker` and `Prober` on one
  small model and your chosen verifier.
- Tests: adapter `certify()` correctness on a toy known-safe and a toy
  known-unsafe property, `trace` schema validation, and an end-to-end attack
  run with a small fixed budget.
- `README.md`: install steps, the usage snippet, and a method write-up
  covering which verifier you chose and why, exactly what you instrumented
  for introspection, and an honest report of whether you reproduced a genuine
  soundness violation.

## Acceptance criteria

- The quickstart runs end-to-end, producing `adv_x` + `atk_logs` and a
  documented `trace`.
- The write-up honestly reports success/failure of the attack and explains
  why.
- You can explain what "sound" and "complete" mean for your chosen verifier
  and exactly where your instrumentation was inserted.
