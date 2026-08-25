# Task 2 — General-Purpose White-box Efficiency Attack Toolbox (`llm_efficiency_attack`)

> Read `00_general_instructions.md` first. You may use any AI coding tool, but
> you must be able to explain every part of your code.

## Problem

An **efficiency attack** crafts an input that is (near-)imperceptibly different
from a benign one but forces the model to do far more computation — e.g., making
a sequence model generate much longer outputs, inflating latency and cost. This
task generalizes such an attack into a reusable white-box toolbox.

## Background material (study before coding)

- **Paper:** "NMTSloth" (FSE'22) — https://dl.acm.org/doi/10.1145/3540250.3549102
- **Reference codebase:** https://github.com/SeekingDream/FSE22_NMTSloth

Understand: why the attack increases computation (the output-length / decoding
objective), what "white-box" gives you (gradients), and how the original repo
generates and evaluates adversarial inputs.

## Objective

Refactor the reference codebase into a clean, **general-purpose** white-box
efficiency-attack toolbox, packaged as a pip-installable Python library called
**`llm_efficiency_attack`**, that can attack **any LLM loaded from Hugging
Face**.

## Required public API

The library must expose exactly this interface:

```python
from llm_efficiency_attack import Attacker

attack = Attacker(model)
adv_x, logs = attack.run(x, config)
```

Where:

| Name     | Type                     | Meaning |
|----------|--------------------------|---------|
| `model`  | Hugging Face model       | The white-box LLM under attack. |
| `x`      | input (text / tokens)    | A benign input. |
| `config` | `dict` / JSON            | Attack configuration (objective, perturbation budget, max iterations, step size, seed, device, etc.). |
| `adv_x`  | input                    | The generated adversarial example derived from `x`. |
| `logs`   | dict / structured object | Generation logs: per-iteration objective value, output-length / cost metric, perturbation size, timing. |

## Detailed requirements

1. **Model-agnostic.** Work for arbitrary Hugging Face models (seq2seq and
   causal LM at minimum). Isolate model/tokenizer-specific assumptions behind an
   adapter so the core attack loop is generic.
2. **White-box optimization.** Use gradients from `model` to drive the attack
   toward higher computational cost (e.g., longer generation). Keep the attack
   objective swappable.
3. **Config-driven.** All hyperparameters come from `config` (JSON-serializable
   dict). Validate it; document every field.
4. **Cost metric.** Provide a clear, reproducible measure of "efficiency damage"
   (e.g., output length or estimated FLOPs/latency) for benign `x` vs `adv_x`,
   and report it in `logs`.
5. **Reproducibility.** Honor a `seed`; same config + input → same result.
6. **Perturbation budget.** Respect an imperceptibility / budget constraint from
   the config so `adv_x` stays close to `x`.

## Deliverables

- The `llm_efficiency_attack` package (installable via `pip install -e .`).
- `examples/quickstart.py` running the snippet above on a small real HF model.
- Tests covering: config validation, the end-to-end `run` path on a tiny model,
  and the cost-metric helper.
- `README.md`: install steps, the usage snippet, and a description of every
  `config` field.

## Acceptance criteria

- The usage snippet runs as written and returns an `adv_x` plus `logs`.
- `logs` clearly show the efficiency cost increased from `x` to `adv_x`.
- Swapping in a different Hugging Face model requires only changing the `model`
  argument (no library code changes).
- You can explain the attack objective, the optimization loop, and where
  model-specific logic is isolated.

## Suggested difficulty: 4 / 5
Generalizing a white-box gradient attack from a single seq2seq setting to
arbitrary causal-LM and seq2seq Hugging Face models, while keeping the attack
objective and cost metric swappable, requires solid understanding of both the
attack and differing model/tokenizer internals.
