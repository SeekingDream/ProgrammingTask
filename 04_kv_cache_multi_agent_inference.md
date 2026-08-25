# Task 4 — Cross-Model KV Cache Reuse for Multi-Agent Inference Pipelines (`KVBridge`)

**Difficulty: 5 / 5**

> Read `README.md` first. You may use any AI coding tool, but you must be able
> to explain every part of your code.

## Problem

Many LLM applications solve a task through a **multi-step pipeline**
(extract → plan → answer, etc.) where each step's prompt is the *previous*
context plus a small amount of new text. Hugging Face's generation API already
exposes this incremental structure through `past_key_values` / `DynamicCache`,
but most pipelines still re-run a full prefill at every step, and this gets
worse in **multi-agent** setups where different steps are handled by
*different* models — a plain KV cache from model A is not even the right
shape to hand to model B.

This task asks you to build a toolbox, `KVBridge`, that (1) reuses the KV
cache across steps when the same model handles consecutive steps, and (2)
attempts to **transfer/adapt** the KV cache across steps that switch models,
so a heterogeneous pipeline avoids a cold prefill at every hand-off.

## Background material (study before coding)

- Hugging Face docs on `past_key_values` / `Cache` / `DynamicCache` — understand
  exactly what is cached (per-layer, per-head key/value tensors) and how shape
  depends on `num_hidden_layers`, `num_attention_heads` /
  `num_key_value_heads`, and `head_dim`.
- **Prompt Cache: Modular Attention Reuse for Low-Latency Inference**
  (arXiv:2311.04934) — a good reference for *same-model* prefix KV reuse.
- **Part (2) — cross-model KV transfer — is not a solved problem in the
  literature.** There is no standard technique for mapping one model's KV
  cache into a differently-shaped model's cache. Treat this as an open
  research question: propose a concrete method, state its assumptions, and be
  honest in your write-up about where it degrades or fails.

## Concrete task and pipeline

Use **GSM8K** (grade-school math word problems; has a gold numeric answer per
example) decomposed into three sequential steps, where each step's prompt is
the original question plus every prior step's output (strict append-only
growth of the context):

1. **Extract** — list the quantities/variables stated in the problem.
2. **Plan** — derive the arithmetic expression/equations from the extracted
   quantities.
3. **Compute** — evaluate the expression and state the final numeric answer.

Use three architecturally distinct, openly-licensed Hugging Face models so
that cross-model transfer is genuinely non-trivial (different tokenizer,
hidden size, number of layers/heads):

- `Qwen/Qwen2.5-7B-Instruct`
- `mistralai/Mistral-7B-Instruct-v0.3`
- `microsoft/Phi-3.5-mini-instruct`

(You may substitute different models if compute requires it, as long as they
remain architecturally distinct from each other and you document the swap.)

Evaluate on a fixed eval subset (e.g. 200 examples from the GSM8K test split)
under all four settings below:

1. `task → model_1(step1) → model_1(step2) → model_1(step3) → output`
2. `task → model_2(step1) → model_2(step2) → model_2(step3) → output`
3. `task → model_3(step1) → model_3(step2) → model_3(step3) → output`
4. `task → model_1(step1) → model_2(step2) → model_3(step3) → output`

For **every** setting, avoid redundant prefilling wherever the context is a
superset of something already computed. For setting 4 specifically, this
means transferring/adapting the KV cache across the model-1→model-2 and
model-2→model-3 hand-offs rather than re-prefilling from scratch.

## Required public API

```python
from kvbridge import Pipeline, evaluate

pipeline = Pipeline(models, config)
result, logs = pipeline.run(task_input, step_assignment)

report = evaluate(pipeline, dataset, step_assignments, config)
```

| Name              | Type                       | Meaning |
|-------------------|----------------------------|---------|
| `models`          | `dict[str, HF model+tokenizer]` | The loaded candidate models, keyed by name (e.g. `"model_1"`). |
| `config`          | `dict` / JSON              | Cache policy, KV-adapter hyperparameters, degradation threshold, seed, device. |
| `task_input`      | str / structured           | A single pipeline input (e.g. one GSM8K question). |
| `step_assignment` | `list[str]` (len 3)        | Which model handles each of the 3 steps, e.g. `["model_1", "model_2", "model_3"]`. |
| `result`          | str / structured           | The pipeline's final output. |
| `logs`            | dict                       | Per-step latency (prefill vs. decode), cache-hit token count, whether a cross-model adapter fired, whether it fell back to a cold prefill. |
| `dataset`         | iterable                   | Eval examples with gold answers. |
| `step_assignments`| `list[list[str]]`          | The settings to compare (the 4 above, at minimum). |
| `report`          | dict                       | Per-setting accuracy, latency breakdown, and cache-hit rate; includes a no-cache baseline for comparison. |

## Detailed requirements

1. **Same-model reuse.** When consecutive steps share a model, do not
   recompute attention over prefix tokens already cached. `logs` must report a
   non-zero cache-hit token count in this case.
2. **Cross-model transfer.** Implement a `KVAdapter` interface:
   `adapt(source_cache, source_config, target_config) -> target_shaped_cache`.
   Provide at least one concrete implementation (e.g. a per-layer linear
   projection mapping source `(num_layers, num_heads, head_dim)` key/value
   tensors into the target's shape, calibrated on a small held-out sample) and
   **document and justify your layer-count mapping** (source and target models
   have a different number of layers).
3. **Fallback on degradation.** Before trusting an adapted cache, compare the
   continuation (logits/perplexity on a small calibration sample) against a
   cold-prefill baseline. If quality degrades beyond a configurable
   threshold, fall back to a full prefill for that hand-off and log that the
   fallback triggered.
4. **Config-driven and reproducible.** `step_assignment`, adapter choice, and
   the degradation threshold all come from `config`; honor `seed`.
5. **Reporting.** `evaluate` must report, per setting: accuracy (exact match
   vs. GSM8K gold answer) and latency (prefill / decode / adapter overhead),
   plus a no-cache baseline so the caching speedup is visible.

## Deliverables

- The `kvbridge` package (installable via `pip install -e .`).
- `examples/quickstart.py` running all 4 settings end-to-end on the named
  models (or a documented smaller stand-in).
- Tests: adapter output-shape correctness, cache-hit accounting, the
  fallback path (construct a case that forces it), and an end-to-end run on a
  small eval subset.
- `README.md`: install steps, the usage snippet, a method write-up on the KV
  adapter (design, layer-mapping choice, calibration procedure, and honestly
  reported limitations/failure modes), and a results table for the 4
  settings (accuracy, latency, cache-hit rate).

## Acceptance criteria

- The quickstart runs end-to-end for all 4 settings and produces the results
  table described above.
- `logs` show non-zero cache-hit tokens whenever consecutive steps share a
  model, and show the fallback path is exercised at least once in tests.
- You can explain the KV adapter's design, why the layer-mapping choice is
  reasonable, and where/why it breaks down.
