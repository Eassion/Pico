# pico

`pico` is a lightweight local coding agent that runs directly in your terminal. It reads your workspace, uses a constrained set of tools to read files, modify files, and run commands, and persists session state in a local `.pico/` directory.

Think of it as a command-line assistant that works persistently inside your repo — not a chat window disconnected from your codebase.

## What it's good for

- Debugging test failures in a local repo
- Reading current code structure and proposing changes
- Small-step iteration grounded in existing files
- Preserving context across sessions so you can pick up where you left off

## Features

- Package name: `pico`, CLI command: `pico`, module entry: `python -m pico`
- Sessions saved in `.pico/sessions/`
- Per-run artifacts saved in `.pico/runs/<run_id>/`
- Three model backends: Ollama, OpenAI-compatible Responses API, Anthropic-compatible Messages API
- Context compression with checkpoint creation
- Durable memory promotion across sessions
- Safety sandbox: path-escape rejection, repeated-call detection, secret redaction
- **Built-in benchmark evaluation system** for deterministic regression testing

## Installation

Requires Python 3.10+.

With `uv`:

```bash
uv sync
```

Or with pip in editable mode:

```bash
pip install -e .
```

## Quick start

Start interactive mode in the current repo:

```bash
uv run pico
```

Point to a different working directory:

```bash
uv run pico --cwd /path/to/repo
```

Run a one-shot task:

```bash
uv run pico "inspect the test failures and propose a fix"
```

Or via module entry:

```bash
python -m pico
```

## Model backends

### Ollama

```bash
ollama serve
ollama pull qwen3.5:4b
uv run pico --provider ollama --model qwen3.5:4b
```

### OpenAI-compatible

```bash
export OPENAI_API_BASE="https://your-api.example/v1"
export OPENAI_API_KEY="your-api-key"
export OPENAI_MODEL="gpt-5.4"
uv run pico --provider openai
```

### Anthropic-compatible

```bash
export ANTHROPIC_API_BASE="https://www.right.codes/claude/v1"
export ANTHROPIC_API_KEY="your-api-key"
export ANTHROPIC_MODEL="claude-sonnet-4-6"
uv run pico --provider anthropic
```

If your server reuses the same API key across multiple compatible endpoints, `pico` falls back from `ANTHROPIC_API_KEY` to `RIGHT_CODES_API_KEY` or `OPENAI_API_KEY`.

## REPL commands

- `/help` — show built-in commands
- `/memory` — view consolidated working memory
- `/session` — show current session file path
- `/reset` — clear current session state
- `/exit` or `/quit` — exit REPL

## Security & persistence

High-risk operations (shell execution, file writes) are gated behind an approval policy:

- `--approval ask` — prompt for each high-risk action
- `--approval auto` — auto-approve all
- `--approval never` — reject all high-risk actions

After each run, these files are written under `.pico/runs/<run_id>/`:

- `task_state.json`
- `trace.jsonl`
- `report.json`

All content stays local — nothing leaves your machine unless you configure a remote model backend.

---

## Architecture

```
pico/
├── cli.py              # CLI entry point (argparse + REPL loop)
├── runtime.py           # Core agent loop: prompt → model → parse → tool → repeat
├── models.py            # Model backends (Fake, Ollama, OpenAI, Anthropic)
├── context_manager.py   # Prompt construction with budget-aware section ordering
├── tools.py             # Tool implementations (read, write, patch, shell, delegate)
├── task_state.py        # Per-run state tracking (steps, attempts, stop reason)
├── run_store.py         # Persistence layer for traces, reports, task state
├── memory.py            # Hierarchical memory (working, episodic, file summaries, durable)
├── workspace.py         # Workspace context and fingerprinting
├── evaluator.py         # Benchmark evaluation engine
└── metrics.py           # Experiment suite and ablation studies
```

The agent follows a simple loop:

1. **Build prompt** — `ContextManager` assembles sections (prefix, memory, history, current request) under a token budget, with compression when needed.
2. **Call model** — `ModelClient.complete(prompt, max_tokens)` returns raw text.
3. **Parse output** — Raw text is parsed for `<tool>` calls, `<final>` answers, or format errors (retry).
4. **Execute tool** — Valid tools run inside the workspace; safety checks (path escape, repeated calls) gate execution.
5. **Record & loop** — Tool results are appended to conversation history; loop continues until `<final>` or step budget exhausted.

## Evaluation system

pico includes a built-in benchmark evaluation framework designed for **deterministic, reproducible regression testing** of the agent core. It answers four questions:

1. **Can the agent complete tasks correctly?** (artifact verification)
2. **Does it stay within step budget?** (efficiency)
3. **Can it recover from errors?** (robustness)
4. **Does each subsystem pull its weight?** (ablation experiments)

### Five-layer architecture

```
Layer 5: Experiment scripts (scripts/)
         Large-scale orchestration across providers and experiment types
         ↓
Layer 4: Metrics system (pico/metrics.py)
         Ablation studies, provider comparisons, context stress matrix, report rendering
         ↓
Layer 3: Benchmark evaluator (pico/evaluator.py)
         Core engine: load tasks → run agent with fake model → verify → score
         ↓
Layer 2: Task definitions (benchmarks/coding_tasks.json)
         Declarative JSON: 12 tasks specifying fixture, tools, budget, verifier
         ↓
Layer 1: Fixture repos (tests/fixtures/)
         Known-initial-state repos that serve as the starting point for each task
```

### Layer 1: Fixture repos

Two minimal repos provide clean starting state for all tasks:

**`tests/fixtures/bench_repo_readme/README.md`:**
```
# Bench Repo README
This is a placeholder benchmark fixture.
## Notes
- Placeholder note about the repo.
- Placeholder note about the file layout.
```

**`tests/fixtures/bench_repo_patch/sample.txt`:**
```
alpha
beta
gamma
placeholder
```

Each task copies its fixture to a temp directory before running — tasks never interfere with each other.

### Layer 2: Task definitions

Each of the 12 tasks in `benchmarks/coding_tasks.json` is a JSON object with these fields:

| Field | Purpose |
|-------|---------|
| `id` | Unique identifier (also keys into `SCRIPTED_MODEL_OUTPUTS`) |
| `prompt` | The user instruction the agent receives |
| `fixture_repo` | Which fixture to copy as the starting workspace |
| `allowed_tools` | Tools the agent is permitted to use |
| `step_budget` | Maximum tool calls before failure |
| `expected_artifact` | Human-readable description of the expected result |
| `verifier` | Shell command that exits 0 on success, non-zero on failure |
| `category` | Classification label |

### Layer 3: Benchmark evaluator

`BenchmarkEvaluator` (in `pico/evaluator.py`) runs each task through a fixed pipeline:

```
Copy fixture → Create FakeModelClient → Create Agent → agent.ask(prompt) → Run verifier → Collect row
```

A task is considered **passed** only when all four conditions hold:

| Condition | What it checks |
|-----------|---------------|
| `within_budget` | Tool steps ≤ step budget |
| `verifier_passed` | Shell verifier exits with code 0 |
| `expected_artifact_exists` | The expected output file exists on disk |
| `non_failure_stop_reason` | Agent stopped with `final_answer_returned`, not an error |

### Layer 4: Metrics system

`pico/metrics.py` (~1670 lines) runs higher-level analyses:

- **Ablation studies** — Measures pass-rate impact of disabling individual subsystems (memory, context reduction, recovery).
- **Provider experiments** — Runs identical tasks against real GPT and Claude APIs to compare results.
- **Context stress matrix** — Tests compression behavior under combinations of history length × note count × request length.
- **Security experiments** — Validates path-escape rejection, symlink traversal, secret redaction, and other safety invariants.
- **Markdown report generation** — Renders aggregated metrics into human-readable documentation.

### Layer 5: Experiment scripts

Top-level orchestration in `scripts/`:

- `run_provider_experiments.py` — Runs the full benchmark against real GPT and Claude API endpoints.
- `run_large_scale_experiments.py` — Runs all experiments (provider + all synthetic/real suites) and writes all artifacts.
- `collect_resume_metrics.py` — Collects metrics from existing artifacts without re-running tasks.

### FakeModelClient: deterministic model simulation

The key to deterministic testing is `FakeModelClient` (in `pico/models.py`). It's a 15-line class that replaces the LLM with a **queue of scripted outputs**:

```python
class FakeModelClient:
    def __init__(self, outputs):
        self.outputs = list(outputs)      # internal queue

    def complete(self, prompt, max_new_tokens, **kwargs):
        if not self.outputs:
            raise RuntimeError("fake model ran out of outputs")
        return self.outputs.pop(0)        # pop next scripted response
```

Each task has a pre-written script in `SCRIPTED_MODEL_OUTPUTS` that defines exactly what the "model" should return on each turn. For example:

```python
"sample_beta_locked": [
    # Turn 1: call patch_file
    '<tool name="patch_file" path="sample.txt">'
    '<old_text>beta</old_text><new_text>beta-locked</new_text>'
    '</tool>',
    # Turn 2: stop
    '<final>Done.</final>',
],
```

Because every part of the agent pipeline — prompt building, parsing, tool execution, verification — remains identical whether the model is real or fake, the same benchmark can run in **two modes**:

| Mode | Model | Used for |
|------|-------|----------|
| Deterministic regression | `FakeModelClient` | CI, local pre-commit, verifying core logic hasn't regressed |
| Provider comparison | Real GPT / Claude API | Measuring real-world model performance on identical tasks |

The switch happens through a `model_client_factory` parameter — pass `None` for fake, pass a factory function for real models. The agent code never knows the difference.

### The 12 benchmark tasks

The 12 tasks span five categories, covering the agent's runtime from normal operation to edge-case recovery:

**Category 1: Documentation (2 tasks) — basic file editing**

| Task | What it tests |
|------|--------------|
| `readme_intro_locked` | Replace a specific sentence in README via `patch_file` |
| `readme_schema_note` | Replace a markdown list item in README |

**Category 2: Text-edit (2 tasks) — single-line replacement**

| Task | What it tests |
|------|--------------|
| `sample_beta_locked` | Replace "beta" with "beta-locked" in sample.txt |
| `sample_gamma_locked` | Replace "gamma" with "gamma-locked" in sample.txt |

**Category 3: Tool-boundary (3 tasks) — error recovery**

| Task | What it tests |
|------|--------------|
| `invalid_patch_recovery` | First tool call has malformed args → agent detects error → model retries with correct format → succeeds |
| `path_escape_recovery` | Model tries to read `../outside.txt` (path escape) → agent rejects → model pivots to legal task → succeeds |
| `repeated_read_recovery` | Model calls `read_file` with identical args 3 times → agent rejects repeats → model pivots → succeeds |

**Category 4: Recovery (3 tasks) — session resumption from anomalous state**

| Task | What it tests |
|------|--------------|
| `context_reduction_checkpoint` | Overstuffed prompt triggers compression → checkpoint created → agent continues from compressed state |
| `freshness_reanchor_resume` | Restored checkpoint has stale file summary → agent detects mismatch → re-reads file → creates new checkpoint |
| `workspace_mismatch_resume` | Restored checkpoint has wrong workspace fingerprint → agent discards old state → rebuilds runtime |

**Category 5: Durable-contract (2 tasks) — persistent memory promotion**

| Task | What it tests |
|------|--------------|
| `durable_promotion_accept` | Stable project conventions are correctly extracted and written to `.pico/memory/` files |
| `durable_promotion_reject` | Transient facts (secret-shaped API keys, task-specific goals) are filtered out; only stable conventions survive |

### Coverage matrix

```
                      edit  recover  safety  dedup  compress  resume  durable
readme_intro           ✓
readme_schema          ✓
sample_beta            ✓
sample_gamma           ✓
invalid_patch                ✓
path_escape                  ✓       ✓
repeated_read                ✓               ✓
context_ckpt                                           ✓
freshness                                                      ✓
workspace                                                      ✓
durable_accept                                                          ✓
durable_reject                                                          ✓
```

### Running the evaluation

Run the deterministic benchmark (fake model, seconds to complete):

```python
from pico.evaluator import run_fixed_benchmark
artifact = run_fixed_benchmark()
print(artifact["summary"])
# {'total_tasks': 12, 'passed': 12, 'pass_rate': 1.0, ...}
```

Or via pytest:

```bash
uv run pytest tests/test_evaluator.py -v
```

Run with real models for provider comparison:

```bash
uv run python scripts/run_provider_experiments.py
```

## Development

Lint with Ruff:

```bash
uv run ruff check .
```

Run all tests:

```bash
uv run pytest tests/ -v
```
