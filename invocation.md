# Invocation

The layer **above** the interpreter — how a Polylith monorepo gets driven at runtime: developer workflows on one side, project entry points on the other.

> `Polylith.md` covers how code is **packaged** (uv · Polylith · Hatchling).
> This file covers how code is **invoked & configured** (Poe · Hydra · hydra-zen).

```
┌──────────────────────────────────────────────────────────────┐
│  Poe          →  Developer task runner                       │
│                  poe pytest · poe lint · poe login           │
├──────────────────────────────────────────────────────────────┤
│  Hydra        →  Config composition + CLI overrides + sweeps │
│                  @hydra.main · defaults lists · _target_     │
├──────────────────────────────────────────────────────────────┤
│  hydra-zen    →  Type-safe structured configs FROM Python    │
│                  builds(MyClass, populate_full_signature)    │
│                  store(Config, name="...")                   │
└──────────────────────────────────────────────────────────────┘
```

---

## Running the monorepo

### Dev workflow — what you type day-to-day

These are the **developer-facing shortcuts**, named by the team. Same commands work from the repo root or any project directory.

```bash
# Testing
poe pytest                       # run tests scoped to wherever you are
poe pytest -k test_message       # passthrough args to pytest

# Linting / formatting
poe lint                         # ruff check
poe fmt                          # ruff format

# Cleanup
poe clean                        # remove build artifacts, caches

# Setup / auth
poe setup                        # sync deps + install hooks + login
poe login                        # auth chain (okta, sso, codeartifact, ...)
poe sync-deps                    # uv lock + uv sync --all-extras

# Discovery
poe                              # list all available tasks
poe -h pytest                    # show what the `pytest` task resolves to
```

### Running each project's CLI

These are the **product-facing entry points** — what an orchestrator (cron, ECS, Step Functions, Anyscale) would also invoke. Each project declares its CLI in `[project.scripts]`.

```bash
# Run the actual project (no Hydra)
uv run my-app

# Run with a Hydra config picked by name
uv run my-app                                   # default config

# Override any field from the command line
uv run my-app name=Alice                        # set value
uv run my-app runner=python                     # switch a group
uv run my-app runner.num_cpus=8                 # nested override
uv run my-app +debug=true                       # add a new field
uv run my-app ~name                             # delete a field

# Sweep across combinations
uv run my-app --multirun name=Alice,Bob runner=ray,python
```

The rule:

| You're invoking… | Use |
|---|---|
| The project's actual CLI (declared in `[project.scripts]`) | `uv run <name>` |
| A dev workflow (test, lint, login, clean, setup, …) | `poe <task>` |

Sanity check: **if an orchestrator would run it in production, it's `uv run …`. If only a developer at a laptop runs it, it's `poe …`.**

---

## The skeleton

The file layout that makes the workflow above possible:

```
my_repo/
├── pyproject.toml               root: [tool.poe] include = ["scripts/tasks.toml"]
├── workspace.toml               Polylith namespace
├── uv.lock
│
├── scripts/
│   └── tasks.toml               SHARED dev tasks (pytest, lint, login, clean, fmt)
│
├── components/
│   └── my_ns/
│       ├── schemas/             bricks
│       └── ...
│
└── projects/
    └── my_app/
        ├── pyproject.toml       per-project:
        │                          [project.scripts] my-app = "my_app.main:main"
        │                          [tool.poe] include = ["../../scripts/tasks.toml", "./tasks.toml"]
        │                          dependencies = ["hydra-core", "hydra-zen"]
        │
        ├── tasks.toml           PROJECT-LOCAL dev tasks (run-default, build, ...)
        │
        ├── configs/             Hydra config files
        │   ├── main.yaml        the primary entry config
        │   ├── runner/          a Hydra config group
        │   │   ├── ray.yaml
        │   │   └── python.yaml
        │   └── model/
        │       ├── v1.yaml
        │       └── v2.yaml
        │
        ├── my_app/
        │   ├── __init__.py
        │   ├── main.py          @hydra.main entry point
        │   ├── core.py          business logic
        │   └── configs.py       hydra-zen builds(...) + store(...) calls
        │
        └── tests/
            └── test_core.py
```

Three TOML files at three layers cooperate:

| File | Purpose |
|---|---|
| Root `pyproject.toml` | Declares the uv workspace, includes shared Poe tasks |
| `scripts/tasks.toml` | Common dev tasks reused across projects |
| Per-project `pyproject.toml` | Declares the project's CLI and Hydra/hydra-zen deps |
| Per-project `tasks.toml` | Project-local dev shortcuts |

---

## The elements

| Tool | Role | Lives in |
|---|---|---|
| **Poe** | Dev task runner — names commands you'd otherwise type by hand | `[tool.poe]` + `tasks.toml` files |
| **Hydra** | Config composition + CLI overrides + multirun sweeps | `configs/*.yaml` + `@hydra.main` decorator |
| **hydra-zen** | Type-safe Structured Configs generated from Python signatures | `builds(...)` + `store(...)` calls in your code |
| **uv** (already covered in `Polylith.md`) | Runs the registered project CLIs in the project venv | `[project.scripts]` |

`poe` and `uv` are *runners* — they execute things. `Hydra` and `hydra-zen` are *config plumbing* — they shape what gets passed in.

---

## Mechanism underneath

### How `poe pytest` runs

When you type `poe pytest`:

```
1. Poe walks up from cwd looking for a pyproject.toml with [tool.poe]
2. Reads [tool.poe] config (shell_interpreter, include, envfile, ...)
3. Reads [tool.poe.tasks] from this file, then merges each file in `include`
        merged = {**this_file_tasks, **included_file_1_tasks, **included_file_2_tasks}
        (later wins on name conflicts → project-local overrides shared)
4. Looks up "pytest" in the merged dict → finds the command string
5. Activates the project's .venv (prepends .venv/bin to PATH, sets VIRTUAL_ENV)
6. Spawns a subprocess via the configured shell: bash -c "pytest -v"
7. Streams stdout/stderr through; exits with the subprocess's exit code
```

**Poe is a TOML-reader + venv-aware subprocess launcher.** No DSL, no graph engine, no daemon, no incremental builds. The whole tool is ~5,000 lines of Python.

The venv awareness is what distinguishes it from a shell alias: before spawning, Poe finds the nearest `.venv/`, prepends `.venv/bin` to `PATH`, and sets `VIRTUAL_ENV`. That's why `pytest = "pytest -v"` works without typing `uv run pytest` — the venv is already active for the duration of the subprocess.

---

### How `my-app runner=ray runner.num_cpus=8` runs

This is where Hydra + hydra-zen cooperate. The trace shows both pieces in their natural roles before we explain each separately.

```
Setup time (at import, when `my-app` first loads my_app.configs):
─────────────────────────────────────────────────────────────────
1. hydra-zen reads Python class signatures and registers schemas:
        builds(RayRunner, populate_full_signature=True)
            → generates a dataclass mirroring RayRunner.__init__
        store(RayRunnerConfig, name="ray", group="runner")
            → registers in Hydra's ConfigStore so `runner=ray` resolves

Run time (when `my-app runner=ray runner.num_cpus=8` is invoked):
─────────────────────────────────────────────────────────────────
2. @hydra.main reads (config_path, config_name) → "configs/main.yaml"

3. Hydra walks the `defaults:` list left to right:
        - runner: ray         →  loads from ConfigStore (hydra-zen-generated)
        - model: v2           →  loads configs/model/v2.yaml
        - _self_              →  loads the rest of main.yaml
        merging each into an accumulating OmegaConf tree

4. Hydra parses CLI overrides:
        runner=ray            →  pick "ray" from the runner group
        runner.num_cpus=8     →  set cfg.runner.num_cpus = 8

5. Hydra validates the resolved tree against the Structured Configs
   in the store — type-checks field names/types against the dataclass

6. Hydra (optionally) chdirs into outputs/<date>/<time>/ so each run
   gets an isolated working directory

7. Hydra calls main(cfg) with the resolved DictConfig

8. When the user code calls hydra.utils.instantiate(cfg.runner):
        - reads cfg.runner._target_ → "my_app.runners.RayRunner"
        - imports that class
        - constructs RayRunner(**cfg.runner_fields)
        - returns the instance
```

**Three layers cooperate:**

```
┌─────────────────────────────────────────────────────────┐
│  Hydra                                                  │
│  • Composition: defaults lists, groups                  │
│  • CLI overrides, multirun, sweeps                      │
│  • Working-directory management                         │
│  • @hydra.main entry-point decorator                    │
│  • hydra.utils.instantiate(cfg) for _target_:           │
├─────────────────────────────────────────────────────────┤
│  hydra-zen                                              │
│  • builds(Target) → dataclass mirroring the signature   │
│  • store() → register in Hydra's ConfigStore            │
│  • Type safety, signature drift catching                │
├─────────────────────────────────────────────────────────┤
│  OmegaConf  (Hydra's underlying dict-like config layer) │
│  • Merge, interpolation, type checks                    │
│  • Backing data structure for DictConfig / ListConfig   │
└─────────────────────────────────────────────────────────┘
```

- **hydra-zen authors the schemas** from Python classes/functions at import time
- **Hydra composes, overrides, and dispatches** at runtime
- **OmegaConf holds the data** as a dict-like structure with type checking

You can use Hydra alone (writing dataclasses by hand for Structured Configs, or skipping them entirely). hydra-zen is purely an ergonomic upgrade — it removes hand-written dataclass boilerplate.

---

## Hydra — in detail

**Hydra** (from Meta / Facebook AI Research, open-sourced 2019, <https://hydra.cc>). A configuration composition framework.

### Setup

```toml
[project]
dependencies = ["hydra-core"]
```

```
configs/
├── main.yaml               # primary entry config
├── runner/                 # a config group
│   ├── ray.yaml
│   └── python.yaml
└── model/                  # another config group
    ├── v1.yaml
    └── v2.yaml
```

```yaml
# configs/main.yaml
defaults:
  - runner: ray
  - model: v2
  - _self_

name: World
greeting_style: cheerful
```

```python
# my_app/main.py
import hydra
from omegaconf import DictConfig

@hydra.main(version_base=None, config_path="configs", config_name="main")
def main(cfg: DictConfig) -> None:
    print(f"name={cfg.name}, runner={cfg.runner}")

if __name__ == "__main__":
    main()
```

### Override syntax cheat sheet

```bash
my-app name=Alice                    # set value
my-app +debug=true                   # add a new field
my-app ~name                         # delete a field
my-app runner=python                 # switch a group
my-app runner.num_cpus=8             # nested override
my-app 'name="quoted string"'        # quoting
my-app --multirun name=A,B,C         # sweep
```

### `_target_` instantiation

A YAML field can name a Python class; `hydra.utils.instantiate(cfg.section)` constructs it:

```yaml
# configs/runner/ray.yaml
_target_: my_app.runners.RayRunner
num_cpus: 4
num_gpus: 0
```

```python
from hydra.utils import instantiate

@hydra.main(...)
def main(cfg):
    runner = instantiate(cfg.runner)   # → RayRunner(num_cpus=4, num_gpus=0)
    runner.execute(...)
```

This is the core of Hydra's "config IS the program" pattern: the YAML names what to construct, Hydra does the construction.

### Key capabilities

| Capability | What it gives you |
|---|---|
| **Defaults list** | Compose configs from groups (`runner=ray` + `model=v2`) |
| **CLI overrides** | Any field settable from the command line via dotted path |
| **`_target_:`** | YAML names a Python class; Hydra constructs the instance |
| **`--multirun`** | Sweep over override combinations; each run isolated |
| **Working-dir isolation** | Each run gets its own output dir under `outputs/<date>/<time>/` |
| **Interpolation** | `${other.field}` references resolve at compose time |
| **Plugins** | Custom launchers (joblib, submitit, Ray), sweepers, search strategies |

### When Hydra shines

- A human is iterating in a terminal, overriding fields constantly
- You need composition (pick `runner=ray`, `model=v2` independently)
- You want sweeps without writing your own loop
- Config IS the program structure (graph engines, training pipelines)
- `_target_:` instantiation is a feature, not a foot-gun, for your workflow

---

## hydra-zen — in detail

**hydra-zen** (from MIT Lincoln Lab, <https://github.com/mit-ll-responsible-ai/hydra-zen>). Companion to Hydra. Generates Structured Configs from Python signatures so you don't write dataclasses by hand.

### Setup

```toml
[project]
dependencies = ["hydra-core", "hydra-zen"]
```

```python
# my_app/configs.py
from hydra_zen import builds, store
from my_app.runners import RayRunner, PythonRunner

# Generate Structured Configs from class signatures:
RayRunnerConfig    = builds(RayRunner,    populate_full_signature=True, zen_partial=True)
PythonRunnerConfig = builds(PythonRunner, populate_full_signature=True, zen_partial=True)

# Register them in Hydra's ConfigStore:
store(RayRunnerConfig,    name="ray",    group="runner")
store(PythonRunnerConfig, name="python", group="runner")
```

After this, `runner=ray` and `runner=python` resolve via the store — no `configs/runner/*.yaml` files needed.

### How `builds(...)` works

`builds(RayRunner, populate_full_signature=True)`:

```
1. Call inspect.signature(RayRunner.__init__) → list of Parameter objects
        │
        ▼
2. For each parameter:
   • Read its name, annotation, default
   • Translate to a dataclass field with that type and default
   • Skip *args / **kwargs (with a warning)
        │
        ▼
3. Build a new @dataclass with those fields, plus:
   • _target_: str = "my_app.runners.RayRunner"   (round-trip pointer)
        │
        ▼
4. Return the dataclass type
```

If `RayRunner.__init__` gains a parameter, the generated config tracks it automatically — no drift.

### How `store(...)` works

```
1. Get Hydra's global ConfigStore instance
2. Call store.store(group="runner", name="ray", node=Config)
3. Now Hydra can resolve `runner=ray` from the store —
   the Structured Config IS the schema, no YAML needed
```

When Hydra later resolves `runner=ray`, it finds the registered config, applies any overrides, then on `instantiate(cfg.runner)` reads `_target_` and constructs `RayRunner(**fields)`.

### Variants of `builds`

```python
# Eager: instantiation produces a RayRunner directly
builds(RayRunner, populate_full_signature=True)

# Lazy: instantiation produces functools.partial(RayRunner, **fields)
# Lets a caller construct WHEN it wants to
builds(RayRunner, populate_full_signature=True, zen_partial=True)

# Override specific defaults at config-creation time
builds(RayRunner, num_cpus=8, populate_full_signature=True)

# Compose: pre-bake some fields, leave others Hydra-overridable
builds(RayRunner, num_cpus=8, num_gpus=2, populate_full_signature=True)
```

### Two main primitives

| Primitive | What it does |
|---|---|
| `builds(target, **overrides, populate_full_signature=True)` | Generate a Structured Config dataclass from a class/function signature. Adds `_target_` for round-trip instantiation. |
| `store(config, name=..., group=...)` | Register the config in Hydra's ConfigStore so it's selectable via `<group>=<name>`. |

Other helpers: `make_config` (config without a `_target_`), `instantiate` (re-export of Hydra's instantiate), `zen` (a decorator alternative to `@hydra.main`).

---

## Poe — in detail

**Poe the Poet** (`poethepoet` on PyPI, `poe` as command). Python-native task runner.

### Setup

```toml
# pyproject.toml
[dependency-groups]
dev = ["poethepoet"]

[tool.poe]
include = ["scripts/tasks.toml"]
shell_interpreter = "bash"
```

After `uv sync`, the `poe` command is available in `.venv/bin/`.

### Declaring tasks

Two options — pick whichever fits.

**Option A — inline in `pyproject.toml`:**

```toml
[tool.poe.tasks]
pytest = "pytest -v"
lint   = "ruff check ."
```

**Option B — in a separate `tasks.toml`, brought in via `include`:**

```toml
# tasks.toml
[tool.poe.tasks]
pytest = "pytest -v"
lint   = "ruff check ."
```

The file format is identical. From the user's perspective, **invocation is the same** (`poe pytest`) — Poe merges all sources into one task namespace at startup.

### Five task types

```toml
[tool.poe.tasks]

# 1. cmd (default) — exec one program with args; no shell parsing
pytest = "pytest -v"

# 2. shell — full shell line; supports pipes, redirects, env vars
clean = { shell = "rm -rf dist && find . -name __pycache__ -delete" }

# 3. script — call a Python function directly (no subprocess)
release-notes = { script = "scripts.release:generate_notes" }

# 4. sequence — run multiple tasks in order; stops on first failure
ci = ["lint", "pytest", "build"]

# 5. ref — alias to another task with extra args
test-fast = { ref = "pytest -m 'not slow'" }
```

There are also `expr` (eval a Python expression) and `switch` (conditional dispatch).

### Task args, env, cwd, deps

```toml
[tool.poe.tasks.deploy]
shell = """
  echo "Building $VERSION..."
  uv build
  aws s3 cp dist/*.whl s3://$BUCKET/
"""
args = [
  { name = "VERSION", positional = true, required = true },
  { name = "BUCKET",  default = "my-builds" },
]
env  = { LOG_LEVEL = "INFO" }
cwd  = "${POE_ROOT}"
help = "Build the wheel and upload it to S3"
```

```bash
poe deploy 1.2.3 --bucket=my-test
```

### What Poe is **not**

| Not | Why |
|---|---|
| A build system | No targets, no incremental computation, no mtime-based dependency graphs |
| A workflow engine | No retry logic, no DAG scheduling, no parallelism beyond OS subprocess |
| A package manager | Doesn't install anything; you `uv add` deps, Poe just runs them |
| Tied to Poetry | Despite the name, works with uv, pip, PDM, Hatch, etc. |
| In runtime wheels | A dev-group dep — never shipped to production |
