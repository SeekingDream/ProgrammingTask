# General Instructions for Research Interns

Welcome! This directory contains a set of programming tasks. Please read these
general instructions carefully **before** starting any individual task.

## Task index

| # | Task | Difficulty |
|---|------|:---:|
| 1 | [Compiler Backdoor Attack Toolbox (`DLCLAttack`)](01_compiler_backdoor_attack.md) | 4 / 5 |
| 2 | ~~[White-box Efficiency Attack Toolbox (`llm_efficiency_attack`)](02_efficiency_attack_toolbox.md)~~ | 2 / 5 |
| 3 | [Formal Error Bound Between a Model and Its Compiled Version](03_compiler_error_bound.md) | 5 / 5 |
| 4 | [Cross-Model KV Cache Reuse for Multi-Agent Inference (`KVBridge`)](04_kv_cache_multi_agent_inference.md) | 3.5 / 5 |
| 5 | [Adversarial Probing & Intermediate-State Dumping for DNN Verifiers (`VerifierProbe`)](05_verifier_adversarial_probe.md) | 5 / 5 |
| 6 | [Static + Dynamic Escape Finder for Python Sandboxes (`SandboxEscapeFinder`)](06_sandbox_escape_finder.md) | 5 / 5 |
| 7 | [LLM Call Recorder & Backend Swap for the Codex Agent Harness (`CodexProbe`)](07_codex_llm_call_recorder.md) | 3 / 5 |

## What we expect from you

1. **Pick a task** from this directory (each task lives in its own `*.md` file).
   If you are unsure which one to start with, ask your supervisor.

2. **Read the referenced paper and codebase first.** Most tasks ask you to
   refactor or build on top of an existing research project. Understanding the
   original method is part of the job — do not skip it.

3. **Deliver working, well-structured code**, not a one-off script. Unless a
   task says otherwise, your deliverable is an installable Python package with a
   clean public API, examples, tests, and a short README.

4. **Some problems with strikethrough are not accepted submission anymore.**
   

## Using AI programming tools

- **You may use any AI programming tool you like** (Claude Code, Cursor,
  Copilot, ChatGPT, etc.). We encourage it.
- **However, you must fully understand every line of code you submit.** AI tools
  are assistants, not substitutes for understanding.
- During review, **we will ask you questions about the code structure and design
  decisions**, for example:
  - Why is the project organized into these modules/classes?
  - What does this function do, and why is it implemented this way?
  - Where would you change the code to support a new model / dataset / attack?
  - What are the failure modes and edge cases of this component?
  - How would you test this part, and why?
- If you cannot explain a piece of code, **do not submit it.** Rewrite or
  simplify it until you can.

## Definition of done (applies to every task)

A task is considered complete when **all** of the following hold:

- [ ] The code runs end-to-end on at least one real Hugging Face model and
      produces the expected output.
- [ ] The public API matches the signature given in the task description.
- [ ] The package installs cleanly (`pip install -e .`) with pinned, listed
      dependencies.
- [ ] There is a `README.md` with installation steps and a runnable usage
      example (the snippet from the task should work as written).
- [ ] There is at least a minimal test suite (`pytest`) covering the core path.
- [ ] Configuration is data-driven (a JSON/dict `config`), not hard-coded.
- [ ] Code is readable: clear names, type hints, docstrings on public functions,
      and meaningful logging.
- [ ] You can verbally walk through the architecture and justify your choices.

## Suggested workflow

1. Reproduce the original repo's results (or at least get it running) before
   refactoring.
2. Sketch the target API and module layout; agree on it with your supervisor.
3. Refactor incrementally, keeping the code runnable at each step.
4. Generalize from the original (often single-model/single-task) setup to
   "any Hugging Face model."
5. Write tests and the README.
6. Prepare to explain your design.

## Deliverable layout (recommended)

```
<package_name>/
├── src/<package_name>/
│   ├── __init__.py          # exports the public API (e.g. Attacker)
│   ├── attacker.py          # core logic
│   ├── config.py            # config schema / loading & validation
│   └── ...
├── tests/
├── examples/
│   └── quickstart.py        # the exact snippet from the task
├── README.md
└── pyproject.toml
```

Good luck, and ask questions early.
</content>
</invoke>
