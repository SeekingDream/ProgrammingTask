# Task 1 — General-Purpose Compiler Backdoor Attack Toolbox (`DLCLAttack`)

**Difficulty: 4 / 5**

> Read `00_general_instructions.md` first. You may use any AI coding tool, but
> you must be able to explain every part of your code.

## Problem

Deep-learning compilers (e.g., TVM, the various Hugging Face / PyTorch
compilation backends) translate a trained model into an optimized executable.
This task is about a **compiler backdoor attack**: an attack that introduces
malicious behavior during/through the compilation stage rather than by
retraining the original model.

## Background material (study before coding)

- **Paper:** "A general compiler backdoor attack" — https://arxiv.org/abs/2509.11173
- **Reference codebase:** https://github.com/SeekingDream/DLCompilerAttack

Spend time understanding: what the threat model is, what exactly the attack
modifies, what a "backdoored model" means in this context, and how the original
repo measures attack success.

## Objective

Refactor the reference codebase into a clean, **general-purpose** attack toolbox,
packaged as a pip-installable Python library called **`DLCLAttack`**, that can
attack **any LLM loaded from Hugging Face**.

## Required public API

The library must expose exactly this interface:

```python
from DLCLAttack import Attacker

attack = Attacker(config)
bd_model, logs = attack.run(model, train_dataset, test_dataset, cl_func)
```

Where:

| Name            | Type                              | Meaning |
|-----------------|-----------------------------------|---------|
| `config`        | `dict` / JSON                     | Attack configuration (hyperparameters, target behavior, budget, seed, device, etc.). |
| `model`         | Hugging Face model                | The clean LLM to be attacked. |
| `train_dataset` | dataset                           | Data used to craft / optimize the backdoor. |
| `test_dataset`  | dataset                           | Data used to evaluate the backdoored model. |
| `cl_func`       | callable                          | The compiler function: `compiled = cl_func(model)`. |
| `bd_model`      | model                             | The resulting backdoored (adversarial) model. |
| `logs`          | dict / structured object          | Generation logs: metrics per step, attack success rate, timing, config echo. |

## Detailed requirements

1. **Model-agnostic.** The toolbox must work for arbitrary Hugging Face models,
   not just the architecture(s) hard-coded in the original repo. Isolate any
   model-specific assumptions behind a small adapter layer.
2. **Config-driven.** All knobs come from `config` (a JSON-serializable dict).
   Validate it and fail with clear error messages on bad input. Document every
   field.
3. **Pluggable compiler.** `cl_func` is supplied by the caller. Do not assume a
   single compiler backend; treat it as an opaque `model -> compiled_model`
   callable.
4. **Reproducibility.** Honor a `seed` in the config; the same config + inputs
   should give the same result.
5. **Logging.** `logs` must let a reviewer reconstruct what happened: attack
   success metric on `test_dataset`, clean-accuracy impact, number of steps,
   wall-clock time, and the effective config.
6. **Evaluation utility.** Provide a helper to measure attack effectiveness
   (e.g., attack success rate of `bd_model` on `test_dataset`) so results are
   reproducible.

## Deliverables

- The `DLCLAttack` package (installable via `pip install -e .`).
- `examples/quickstart.py` running the snippet above on a small real HF model.
- Tests covering: config validation, the end-to-end `run` path (a tiny model is
  fine), and the evaluation helper.
- `README.md`: install steps, the usage snippet, and a description of every
  `config` field.

## Acceptance criteria

- The usage snippet runs as written and returns a `bd_model` plus `logs`.
- Swapping in a different Hugging Face model requires only changing the `model`
  argument (no library code changes).
- You can explain the module layout, where model-specific logic is isolated, and
  how the backdoor is injected and measured.
