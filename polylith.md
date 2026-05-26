# Polylith

At a high level: **uv + polylith + hatchling**.

> uv runs everything · Polylith organizes everything · Hatchling ships everything.

```
┌──────────────────────────────────────────────────────────────┐
│  uv             →  Project lifecycle in one ecosystem        │
│                    install · sync · lock · run · invoke build│
├──────────────────────────────────────────────────────────────┤
│  Polylith       →  Architecture pattern (bricks ↔ projects)  │
│                    workspace.toml + [tool.polylith.bricks]   │
│                    Two consumers:                            │
│                      • poly (CLI)    — inspect / scaffold    │
│                      • hatch-polylith-bricks (plugin) — build│
├──────────────────────────────────────────────────────────────┤
│  Hatchling      →  Build backend (source → wheel)            │
│                    + the polylith plugin copies bricks in    │
│                    → wheels become self-contained            │
└──────────────────────────────────────────────────────────────┘
```

---

## uv

`pyproject.toml` is what makes a directory a uv directory. Run `uv sync` in it and uv will:

- create `.venv/` (or honor `UV_PROJECT_ENVIRONMENT`)
- generate `uv.lock`
- install the project + deps in editable mode

### Three flavors of uv project

**1. Single-package project** — typical app/library

```toml
[project]
name = "my-tool"
version = "0.1.0"
dependencies = ["requests"]
```

`uv build` produces a wheel.

**2. Non-package project** — manages a venv + deps but doesn't build a wheel

```toml
[project]
name = "my-workspace-root"
version = "0.0.0"

[tool.uv]
package = false   # ← this is the key
```

This is what the Polylith monorepo (and the skeleton) does at the repo root. The root isn't a deployable artifact; it's just there to declare the workspace.

**3. Workspace (monorepo)** — root + members

```toml
# Root pyproject.toml
[tool.uv.workspace]
members = ["projects/greeter", "projects/analyzer"]
```

Each `members/*/pyproject.toml` is itself a uv project. One shared `uv.lock` at the root; one shared `.venv/`; each member is installed editably.

### What uv is **not**

- uv does **not** require a specific build backend. Hatchling, setuptools, poetry-core, flit — all fine. uv only reads `[project]` and `[tool.uv]`; the build backend reads `[build-system]` and `[tool.hatch.*]`.
- uv does **not** care about Polylith. The two layers are orthogonal.

---

## Polylith

Concept borrowed from: <https://davidvujic.github.io/python-polylith-docs/>

### What `workspace.toml` contains

```toml
[tool.polylith]
namespace = "my_namespace"          # import root: from my_namespace.schemas import X
git_tag_pattern = "v[0-9]*.[0-9]*.[0-9]*"

[tool.polylith.structure]
theme = "loose"                     # see below

[tool.polylith.resources]
brick_docs_enabled = false          # whether `poly` looks for per-brick READMEs

[tool.polylith.test]
enabled = true                      # whether `poly test` is supported
```

`workspace.toml` is what makes a project Polylith. It tells `poly` (polylith-cli) and `hatch-polylith-bricks` how to interpret the repo as a Polylith structure.

- **namespace** — the Python top-level import name. All bricks live under `components/<namespace>/` and resolve as `from <namespace>.schemas import X`.
- **theme** — `loose` keeps bricks one level under the namespace dir (what Torc uses); `tdd` adds an extra nesting layer.

---

## Hatchling

When you see `requires = ["hatchling"]` in `[build-system]`:

Hatchling is a Python build backend — the thing that turns source code into installable wheels and sdists. You never call it directly; it gets invoked behind the scenes by `uv build`, `uv sync`, or `pip install -e .`.

### How it gets invoked

```
You run:        uv sync     or    uv build     or    pip install -e .
                   ↓
The tool reads: pyproject.toml → [build-system] → "hatchling.build"
                   ↓
Hatchling:      resolves source dirs, runs plugins, assembles the package
```

### The config it reads

```toml
[build-system]
requires = ["hatchling"]            # declares the backend
build-backend = "hatchling.build"

[tool.hatch.build]
dev-mode-dirs = ["components", "."]  # editable-install src dirs

[tool.hatch.build.targets.wheel]
packages = ["greeter", "poly_sandbox"]  # what goes in the wheel

[tool.hatch.build.hooks.polylith-bricks]  # enable the polylith plugin
```

| Key | What it does |
|---|---|
| `dev-mode-dirs` | Adds these dirs to `sys.path` when installed editably (`uv sync`). **This is the magic line** that makes `from poly_sandbox.schemas import X` resolve to `components/poly_sandbox/schemas/` in dev. |
| `targets.wheel.packages` | Which top-level packages to include in the built wheel |
| `hooks.<name>` | Activates a plugin (e.g. `polylith-bricks`) |

### The plugin system

`hatch-polylith-bricks` is a hook that runs at build time and copies brick directories from `components/` into the wheel, mapped by:

```toml
[tool.polylith.bricks]
"../../components/poly_sandbox/schemas" = "poly_sandbox/schemas"
```

Without that plugin, your wheel would be missing the bricks.

The plugin is what makes the dual-resolution model work:

1. **Dev** — `components/` on `sys.path` via `dev-mode-dirs`
2. **Prod** — brick contents copied into the wheel by the plugin

---

## Bricks

Bricks are regular Python packages with **no `pyproject.toml`**. They aren't independently installable.

- During development, they're importable via `dev-mode-dirs`.
- During production builds, the `hatch-polylith-bricks` plugin copies them into project wheels.

This dual resolution — source tree in dev, bundled in prod — is the core Polylith mechanism.

---

## Projects

A deployable package with its own `pyproject.toml`, entry point, tests, and Poe tasks. The project-level `pyproject.toml` differs from the root in two key ways:

1. It requires `hatch-polylith-bricks` in its build system (the root doesn't, because the root doesn't ship bricks in a wheel).
2. It declares `[tool.polylith.bricks]` mappings that tell the build plugin which bricks to include in production wheels.

The `[project.scripts]` section registers a CLI command.

### Minimal example — one brick, one project, one test

```
my_repo/
├── pyproject.toml                  workspace root, declares uv workspace
├── workspace.toml                  namespace = "my_ns"
│
├── components/
│   └── my_ns/
│       └── schemas/                ← the one brick
│           ├── __init__.py
│           └── models.py
│
├── projects/
│   └── my_app/
│       ├── pyproject.toml          maps schemas brick → wheel; declares script
│       ├── my_app/                 ← the one project
│       │   ├── __init__.py
│       │   ├── main.py             from my_ns.schemas import Message
│       │   └── core.py
│       └── tests/
│           └── test_core.py        ← the one test
│
└── tests/
    └── components/
        └── my_ns/
            └── test_schemas.py     (optional — brick tests at root)
```

### Minimum content of each file

**`workspace.toml`**

```toml
[tool.polylith]
namespace = "my_ns"

[tool.polylith.structure]
theme = "loose"
```

**Root `pyproject.toml`**

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-repo"
version = "0.0.0"
requires-python = ">=3.11"

[tool.uv]
package = false

[tool.uv.workspace]
members = ["projects/my_app"]

[tool.hatch.build]
dev-mode-dirs = ["components", "."]
```

**`components/my_ns/schemas/models.py`**

```python
from pydantic import BaseModel

class Message(BaseModel):
    text: str
```

**`components/my_ns/schemas/__init__.py`**

```python
from my_ns.schemas.models import Message

__all__ = ["Message"]
```

**`projects/my_app/pyproject.toml`**

```toml
[build-system]
requires = ["hatchling", "hatch-polylith-bricks"]
build-backend = "hatchling.build"

[tool.hatch.build.hooks.polylith-bricks]

[tool.polylith.bricks]
"../../components/my_ns/schemas" = "my_ns/schemas"   # brick → wheel

[tool.hatch.build]
dev-mode-dirs = ["../../components", "."]

[tool.hatch.build.targets.wheel]
packages = ["my_app", "my_ns"]

[project]
name = "my-app"
version = "0.0.0"
requires-python = ">=3.11"
dependencies = ["pydantic"]

[project.scripts]
my-app = "my_app.main:main"
```

**`projects/my_app/my_app/main.py`**

```python
from my_ns.schemas import Message

def main() -> None:
    print(Message(text="hello").text)
```

**`projects/my_app/tests/test_core.py`**

```python
from my_ns.schemas import Message

def test_message():
    assert Message(text="hi").text == "hi"
```

### The full lifecycle on this one example

```bash
$ uv sync                         # creates .venv, installs my_app editably,
                                  # puts components/ on sys.path
$ uv run my-app                   # runs my_app.main:main → prints "hello"
$ uv run pytest                   # runs test_core.py
$ cd projects/my_app && uv build  # produces a wheel containing my_app/ + my_ns/schemas/
```
