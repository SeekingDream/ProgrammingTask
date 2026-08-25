# Task 7 — LLM Call Recorder & Backend Swap for the Codex Agent Harness (`CodexProbe`)

**Difficulty: 4 / 5**

> Read `README.md` first. You may use any AI coding tool, but you must be able
> to explain every part of your code.

## Problem

Agent harnesses like OpenAI's **Codex CLI** (a terminal coding agent,
open-sourced on GitHub — not Hugging Face; correcting a common mix-up, since
Codex is a Rust CLI application, not a model or dataset artifact) drive a
multi-turn loop of LLM calls (system/developer instructions, running
conversation + tool history, tool definitions in; assistant message and/or
tool calls out) that is normally opaque to the user — you see the agent's
final actions but not the exact requests/responses that produced them. This
task builds a transparent, model-agnostic "hook" that records every LLM call
Codex makes, and uses that same hook point to swap Codex's backend model for
a self-hosted open-source LLM.

## Background material (study before coding)

- **Reference codebase:** Codex CLI — https://github.com/openai/codex
  (Apache-2.0, Rust). Read `docs/config.md` and `docs/getting-started.md`.
- **Extension point:** Codex supports pointing at an arbitrary
  OpenAI-compatible backend via a `[model_providers.<name>]` block in
  `config.toml` (`base_url`, `wire_api`, `env_key`), including a documented
  local-OSS example:
  ```toml
  [model_providers.local_ollama]
  name = "Ollama"
  base_url = "http://localhost:11434/v1"
  wire_api = "responses"
  ```
  Understand the difference between the two `wire_api` values Codex supports
  (the Responses API vs. the Chat Completions API) — local OSS servers such
  as vLLM and Ollama speak Chat Completions (`/v1/chat/completions`).
- Codex has a `log_dir` config option and OpenTelemetry tracing support, but
  **does not, out of the box, dump the full verbatim request/response body of
  every LLM call** — that is the gap this task fills, via a proxy sitting at
  the `base_url` extension point rather than by patching Codex's Rust source.

## Objective

Build `CodexProbe`: a small local reverse-proxy + recorder that (1) sits
between Codex and its real LLM backend, transparently forwarding every
request/response while logging the complete input and output of each call,
and (2) — using that exact same proxy — routes Codex to a self-hosted,
open-source model (e.g. a Qwen coder model) instead of the default OpenAI
backend, and demonstrates Codex completing a real task through it.

## Required public API

```python
from codex_probe import ProxyRecorder

recorder = ProxyRecorder(config)
endpoint = recorder.start()   # e.g. "http://127.0.0.1:8135/v1"
# point model_providers.<name>.base_url in Codex's config.toml at `endpoint`,
# then run Codex normally (e.g. `codex exec "<task prompt>"`)
calls = recorder.stop()
```

| Name        | Type                     | Meaning |
|-------------|--------------------------|---------|
| `config`    | `dict` / JSON            | Real backend `base_url` + `wire_api` + auth to forward to, listen host/port, log directory, seed. |
| `endpoint`  | str (URL)                | The local OpenAI-compatible endpoint to put in Codex's `model_providers.<name>.base_url`. |
| `calls`     | `list[dict]`             | One entry per LLM call, in order: full request body, full response body (streaming reassembled), latency, which backend served it. |

## Detailed requirements

1. **Faithful passthrough.** The proxy must forward each request unchanged to
   the configured real backend and relay the real response back to Codex
   unmodified — Codex's behavior must not change because the proxy is in the
   path.
2. **Streaming-aware.** Codex requests are typically streamed (SSE). The
   proxy must relay the stream live to Codex (so the CLI stays responsive)
   while reassembling the full response for the log.
3. **Full, unabridged capture.** Each logged call must include the entire
   request payload (system/developer instructions, full running
   message/tool history, tool schema) and the entire response (message
   content and/or tool calls) — not a truncated or summarized version.
4. **Backend is config-only, not code.** Swapping which real backend the
   proxy forwards to (OpenAI vs. a local OSS server) must be a `config`
   change, not a code change — this is what lets the same recorder capture
   both the "before" and "after" runs.
5. **Backend swap to an open model.** Serve a Qwen coder model locally with
   an OpenAI-compatible Chat Completions API — e.g. `Qwen/Qwen2.5-Coder-7B-Instruct`
   via `vllm serve Qwen/Qwen2.5-Coder-7B-Instruct` (or Ollama's Qwen2.5-Coder
   build). Add a `model_providers` entry pointing at your recorder's
   `endpoint`, set it as Codex's active `model_provider`/`model`, and run a
   real (small) coding task through Codex end-to-end. Confirm from the
   recorded logs that the calls actually reached the local Qwen server, not
   OpenAI.
6. **Session-organized logs.** Group `calls` per Codex invocation (e.g. one
   JSONL file per run) so a reviewer can replay a session turn-by-turn.
7. **Config-driven and reproducible.**

## Deliverables

- The `codex_probe` package (installable via `pip install -e .`).
- `examples/quickstart.py` (or an equivalent documented shell walkthrough)
  showing: starting the recorder, the `config.toml` fragment pointing Codex
  at it, running Codex against (a) the default OpenAI backend and (b) the
  swapped-in local Qwen backend, and inspecting the resulting logs.
- Tests: passthrough correctness against a mock OpenAI-compatible server
  (request/response round-trip unchanged), streaming reassembly correctness,
  and log schema validation.
- `README.md`: install steps, the usage snippet, how to serve the local Qwen
  model (command + approximate hardware needed), the exact Codex config
  changes required, and a short write-up of what a captured call looks like.

## Acceptance criteria

- Running Codex through the recorder against the default backend produces a
  complete, ordered log of every LLM call's input and output.
- Running the same kind of task through the swapped Qwen backend produces
  logs showing the calls hit the local Qwen server, and Codex completes the
  task (or the write-up honestly documents where the open model's
  tool-calling behavior diverges/breaks — that is an acceptable and expected
  finding to report).
- You can explain how the proxy handles streaming without changing Codex's
  behavior, and the difference between the Responses and Chat Completions
  wire formats.
