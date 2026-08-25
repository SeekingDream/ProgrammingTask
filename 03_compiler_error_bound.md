# Task 3 — Formal Error Bound Between a Model and Its Compiled Version

**Difficulty: 4 / 5**

> Read `00_general_instructions.md` first. You may use any AI coding tool, but
> you must be able to explain every part of your code.

## Problem

When a model is compiled (optimized, quantized, fused, lowered to a target
backend), the compiled version is not bit-for-bit identical to the original.
Your task is to **formally bound the maximum output difference** between a model
and its compiled version over a given input domain.

## Objective

Implement a function that uses a **formal method** (not just random sampling) to
infer a sound error bound between a model and its compiled counterpart on a
specified input domain `X`.

## Required public API

```python
cl_model = cl_func(model)
bound = MaxBound(model, cl_model, X)
```

Where:

| Name       | Type                | Meaning |
|------------|---------------------|---------|
| `model`    | Hugging Face DNN    | The original model. |
| `cl_func`  | callable            | The compiler function: `cl_model = cl_func(model)`. |
| `cl_model` | model               | The compiled version of `model`. |
| `X`        | input domain        | A specification of the input region (e.g., a box / interval per input dimension, or input + epsilon ball). |
| `bound`    | number              | A **sound** upper bound on the output discrepancy between `model` and `cl_model` over all inputs in `X`. |

## Detailed requirements

1. **Soundness first.** `bound` must be an *upper bound*: for every input in `X`,
   the actual difference `‖model(x) - cl_model(x)‖` must be ≤ `bound`. A bound
   that can be violated by some input in `X` is incorrect. State the norm you
   use.
2. **Formal method, not sampling.** The bound must come from a formal /
   verification technique. Acceptable directions (pick one and justify it):
   - Interval Bound Propagation (IBP) / abstract interpretation over the
     difference network.
   - Linear relaxation (e.g., CROWN-style bounds).
   - An off-the-shelf NN verifier, with the model-vs-compiled difference encoded
     as the property to certify.
   You may use existing verification libraries — but explain what they do.
3. **Define the input domain `X` precisely.** Document its format (e.g., lower
   and upper bound tensors, or a center point plus L∞ radius).
4. **Be explicit about scope and assumptions.** Full formal verification of a
   large LLM is generally intractable. It is acceptable to:
   - target small models / sub-networks, and/or
   - bound a well-defined slice (e.g., a single layer or block, or a reduced
     precision/quantization step),
   as long as you clearly state the assumptions and the bound is sound under
   them.
5. **Validate empirically as a sanity check.** Sampling many inputs from `X`
   should never produce a difference exceeding `bound`. Include such a test (it
   checks correctness but is *not* a substitute for the formal derivation).

## Deliverables

- A small package/module exposing `MaxBound(model, cl_model, X)`.
- `examples/quickstart.py` demonstrating the API on a small model with a concrete
  `cl_func` (e.g., quantization or a simple compile pass) and a concrete `X`.
- A short **method write-up** (in the README) explaining: the formal technique,
  the norm, the input-domain format, the assumptions, and why the bound is sound.
- Tests: the empirical sanity check above, plus a tiny case where the bound can
  be reasoned about by hand.

## Acceptance criteria

- `MaxBound` returns a number that is a sound upper bound under the stated
  assumptions.
- Empirical sampling over `X` never exceeds the returned `bound`.
- You can explain why the method is sound, the role of `X`, and the limitations
  / scope of your approach.
