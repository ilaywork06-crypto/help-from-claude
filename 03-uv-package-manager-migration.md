# UV Package Manager Migration

> **Audience:** Developers and DevOps engineers migrating a large Python monorepo from Poetry to UV.
> **Scope:** What UV is and why we're adopting it, converting an existing Poetry project/monorepo to UV, modeling the monorepo with UV workspaces, managing many dependency versions safely across sub-projects, air-gapped/private-index configuration for UV itself, common migration pitfalls, and a UV-specific checklist. NX mechanics are covered in [02 — NX Monorepo Fundamentals](02-nx-monorepo-fundamentals.md); Ruff and the custom formatter are covered in [04 — Ruff and the Custom Formatter](04-ruff-and-custom-formatter.md); GitLab CI/CD YAML and Docker specifics live in [06 — GitLab CI/CD Integration](06-gitlab-ci-integration.md).
> **Versions referenced:** UV 0.6+ (examples below use 0.7.x Docker images).

---

## Table of Contents

1. [What Is UV, and Why Move Away from Poetry](#1-what-is-uv-and-why-move-away-from-poetry)
2. [Migrating an Existing Poetry Project to UV](#2-migrating-an-existing-poetry-project-to-uv)
3. [UV Workspaces — Modeling a Real Python Monorepo](#3-uv-workspaces--modeling-a-real-python-monorepo)
4. [Managing Many Dependency Versions Safely Across Sub-Projects](#4-managing-many-dependency-versions-safely-across-sub-projects)
5. [UV in an Air-Gapped Environment](#5-uv-in-an-air-gapped-environment)
6. [Pitfalls and the @nxlv/python Bridge](#6-pitfalls-and-the-nxlvpython-bridge)
7. [UV Migration Checklist](#7-uv-migration-checklist)

---

## 1. What Is UV, and Why Move Away from Poetry

[UV](https://docs.astral.sh/uv) is a Python packaging and project manager built in Rust by Astral — the same company behind Ruff. It is designed as a single, fast replacement for a whole cluster of tools:

| Old tool | Replaced by |
|---|---|
| `poetry install` / `poetry add` / `poetry remove` | `uv sync` / `uv add` / `uv remove` |
| `poetry shell` | `uv run` (no activation needed) |
| `pip` / `pip-tools` | `uv pip`, `uv lock` + `uv export` |
| `pyenv` | `uv python install` / `uv python pin` |
| `virtualenv` | Built into UV |
| `twine` (partially) | `uv publish` |

### Why the move

| Concern | Poetry (current) | UV (target) |
|---|---|---|
| Dependency install speed | pip-based, slow (~48s cold, ~4s warm on a typical project) | 10–100x faster (~7s cold, <1s warm) |
| Lockfile | Works, but not truly cross-platform in practice | Universal lockfile (`uv.lock`) resolves for every OS/Python combination at once |
| Monorepo support | None natively — scripts and workarounds | Native **workspaces**, Cargo-style |
| Python version management | Requires `pyenv` alongside Poetry | Built in (`uv python install/pin`) |
| Dependency grouping | `[tool.poetry.group.*]` | Standardized `[dependency-groups]` (PEP 735) |
| Standards compliance | Poetry's own `[tool.poetry]` schema | PEP 621 `[project]` — the same metadata format setuptools, Hatch, and other modern backends use |

The headline benefit for CI is speed: on a 250k-LOC-scale monorepo, teams commonly see dependency installation drop from tens of seconds to single digits, and warm-cache installs finish in under a second. Combined with NX affected (file 02) and Ruff (file 04), UV is the piece that makes "only rebuild what changed" fast enough to matter in practice.

### Installing UV

**On a network with internet access:**

```bash
# Linux/macOS
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows PowerShell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# Via pip, if pip is already available
pip install uv
```

**In an air-gapped environment**, see [Section 5](#5-uv-in-an-air-gapped-environment) — UV ships as a single static binary, which makes it easy to vendor.

---

## 2. Migrating an Existing Poetry Project to UV

Treat this as its own multi-day workstream, separate from the NX rollout — mixing the two makes it hard to tell which tool caused a given failure. Start with one pilot package, validate it end to end, then repeat across the rest of the monorepo.

### Step 1 — Inventory what you have

```bash
# Find every pyproject.toml in the repo
find . -name "pyproject.toml" -not -path "*/node_modules/*" -not -path "*/.venv/*"

# Check for Poetry dependency groups
grep -r "\[tool.poetry.group" . --include="*.toml"

# Check for custom Poetry package sources
grep -r "\[\[tool.poetry.source\]\]" . --include="*.toml"

# Back up what you have before touching anything
cp pyproject.toml pyproject.toml.poetry.bak
cp poetry.lock poetry.lock.bak

# Export current requirements as a sanity-check baseline
poetry export -f requirements.txt --output requirements-poetry.txt
poetry export -f requirements.txt --with dev --output requirements-dev-poetry.txt
```

### Step 2 — Convert `pyproject.toml` to PEP 621 + UV

**Before (Poetry):**

```toml
[tool.poetry]
name = "my-package"
version = "1.0.0"
description = "A package"
authors = ["Developer <dev@company.com>"]
packages = [{ include = "my_package", from = "src" }]

[tool.poetry.dependencies]
python = "^3.11"
requests = { version = "^2.28", extras = ["security"] }

[tool.poetry.group.dev.dependencies]
pytest = "^8.0"
black = "^24.0"
flake8 = "^7.0"
isort = "^5.0"

[tool.poetry.scripts]
my-cli = "my_package.cli:main"

[[tool.poetry.source]]
name = "company-pypi"
url = "https://pypi.internal.company.com/simple/"
priority = "primary"

[build-system]
requires = ["poetry-core>=1.0.0"]
build-backend = "poetry.core.masonry.api"
```

**After (UV / PEP 621):**

```toml
[project]
name = "my-package"
version = "1.0.0"
description = "A package"
authors = [{ name = "Developer", email = "dev@company.com" }]
requires-python = ">=3.11"
dependencies = [
    "requests[security]>=2.28",
]

[project.scripts]
my-cli = "my_package.cli:main"

[dependency-groups]
dev = [
    "pytest>=8.0",
    "ruff>=0.9.0",   # replaces black + flake8 + isort (see file 04)
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_package"]
```

**What changed and why:**

| Poetry | UV / PEP 621 | Note |
|---|---|---|
| `[tool.poetry]` | `[project]` | Standard PEP 621 metadata block |
| `python = "^3.11"` | `requires-python = ">=3.11"` | UV uses PEP 440 specifiers, not Poetry's caret syntax |
| `[tool.poetry.group.dev.dependencies]` | `[dependency-groups]` → `dev = [...]` | PEP 735 dependency groups |
| `[tool.poetry.scripts]` | `[project.scripts]` | Same concept, PEP 621 location |
| `[tool.poetry.extras]` | `[project.optional-dependencies]` | Install with `uv add "my-package[redis]"` |
| `[[tool.poetry.source]]` | `[[tool.uv.index]]` | See [Section 5](#5-uv-in-an-air-gapped-environment) |
| `build-backend = "poetry.core.masonry.api"` | `build-backend = "hatchling.build"` (or `setuptools`/`flit-core`) | Poetry-core cannot be reused; pick a backend (see [Section 6](#6-pitfalls-and-the-nxlvpython-bridge)) |
| Caret ranges (`^0.27`) | PEP 440 ranges (`>=0.27`) | Caret has no direct UV equivalent — decide real bounds |

UV ships an experimental helper for the boilerplate conversion:

```bash
uv init --from-poetry
```

Treat its output as a first draft, not a final answer — always diff it against the original and re-check dependency groups, scripts, and extras by hand.

### Step 3 — Generate the lockfile and validate

```bash
# Remove the old lockfile and any stale virtualenvs
rm poetry.lock
rm -rf .venv

# Create uv.lock (the workspace-wide equivalent of poetry.lock)
uv lock

# Install everything the lockfile describes
uv sync --all-groups

# Sanity checks
uv run python -c "import my_package; print('OK')"
uv run pytest
uv tree

# Optional: compare against the old requirements export
uv export --format requirements.txt > requirements-uv.txt
diff requirements-poetry.txt requirements-uv.txt
```

### Step 4 — Update CI/CD and local tooling

```yaml
# Before (Poetry)
before_script:
  - pip install poetry
  - poetry install

# After (UV)
before_script:
  - pip install uv   # or copy the pinned binary — see Section 5
  - uv sync --frozen
```

```bash
# Before                       # After
poetry run pytest              uv run pytest
poetry run black .             uv run ruff format .
poetry run flake8 .            uv run ruff check .
poetry shell                   uv run bash / source .venv/bin/activate
poetry update                  uv lock --upgrade
poetry show --tree             uv tree
poetry export -f requirements.txt   uv export --format requirements.txt
```

Full CI YAML and Docker image details (base images, cache blocks, `UV_LINK_MODE`, etc.) are covered in file 06 — the important package-manager-level takeaway here is that every `poetry install`/`poetry run` invocation in scripts, Makefiles, and pipelines needs a UV equivalent before the old tool is removed.

---

## 3. UV Workspaces — Modeling a Real Python Monorepo

UV workspaces are UV's answer to a monorepo of independently-versioned Python packages — conceptually similar to Cargo workspaces in Rust. One `uv.lock` and one shared virtual environment cover the entire workspace, while each member keeps its own `pyproject.toml`, version, and dependency list.

### Workspace root

```toml
# pyproject.toml (workspace root)
[project]
name = "monorepo-root"
version = "0.0.1"
description = "Monorepo workspace root"
requires-python = ">=3.11"
dependencies = []

# ==============================
# UV Workspace Definition
# ==============================
[tool.uv.workspace]
members = [
    "packages/*",
    "apps/*",
]
exclude = [
    "packages/legacy-*",      # not yet migrated
    "packages/experiments/*", # experimental scratch code
]

# ==============================
# Internal cross-package references
# ==============================
[tool.uv.sources]
shared-utils = { workspace = true }
db-models = { workspace = true }
auth-helpers = { workspace = true }

# ==============================
# Shared dev tooling for the whole workspace
# ==============================
[dependency-groups]
dev = [
    "ruff>=0.9.0",
    "pytest>=8.0",
    "pytest-cov>=5.0",
    "mypy>=1.10",
    "pre-commit>=3.7",
]
```

Private-index configuration (`[[tool.uv.index]]`) also lives in this root file — see [Section 5](#5-uv-in-an-air-gapped-environment).

### A workspace member

```toml
# packages/api-service/pyproject.toml
[project]
name = "api-service"
version = "0.1.0"
description = "Company API service"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.34.0",
    "pydantic>=2.10.0",
    # internal workspace dependencies — resolved from the workspace, not an index
    "shared-utils",
    "db-models",
    "auth-helpers",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=6.0.0",
    "httpx>=0.28.0",
]

[tool.uv.sources]
shared-utils = { workspace = true }
db-models = { workspace = true }
auth-helpers = { workspace = true }

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/api_service"]
```

`[tool.uv.sources]` with `{ workspace = true }` is what tells UV (and, in turn, NX via `@nxlv/python`) that a dependency is another package in this repo rather than something to fetch from an index. Every internal cross-package dependency must be declared this way — it is the single most important convention in a UV monorepo.

### Everyday workspace commands

```bash
# Install the entire workspace
uv sync

# Install just one package plus its own dependencies
uv sync --package api-service

# Run a command scoped to one package
uv run --package api-service python -m pytest

# Add an external dependency to one package
uv add httpx --package api-service

# Add an internal (workspace) dependency
uv add shared-utils --package api-service

# Build one package's wheel/sdist
uv build --package shared-utils --out-dir dist/

# Regenerate the lockfile for the whole workspace
uv lock

# Verify the lockfile is current without changing anything (use in CI)
uv lock --check

# Inspect the dependency tree
uv tree
uv tree --package api-service
```

---

## 4. Managing Many Dependency Versions Safely Across Sub-Projects

A monorepo workspace shares **one** resolution and **one** lockfile — UV resolves every member's dependencies together, which is what guarantees a consistent environment, but it also means version conflicts between packages become resolution failures instead of silently-diverging virtualenvs.

### The failure mode

```
lib-a requires:     httpx>=0.27
service-x requires: httpx>=0.25,<0.27
```

```
error: Because lib-a requires httpx>=0.27 and service-x requires httpx<0.27,
we can conclude that lib-a and service-x cannot be used together.
```

UV will refuse to lock rather than silently pick an incompatible version. This is a feature, not a bug — but it means dependency bumps in one package can block `uv lock` for the entire workspace.

### How to avoid and resolve it

- **Keep lower bounds loose, upper bounds rare.** Prefer `httpx>=0.25` over pinning a narrow range; only add an upper bound when a specific breaking change is known.
- **Diagnose with `uv tree`:**
  ```bash
  uv tree --package lib-a
  uv tree --package service-x
  ```
- **Align constraints deliberately** rather than pinning to whatever happens to install:
  ```bash
  uv lock --upgrade-package httpx
  ```
- **`requires-python` also merges across the workspace.** UV enforces the *intersection* of every member's `requires-python`. If `lib-a` declares `>=3.11` and `lib-b` declares `>=3.10`, the effective workspace requirement becomes `>=3.11`. Keep this value consistent across packages to avoid surprise upper-bound-style errors.
- **`uv.lock` merge conflicts in git** are common when multiple branches touch dependencies in parallel. Don't hand-edit the lockfile to resolve a conflict — resolve `pyproject.toml` conflicts first, then regenerate:
  ```bash
  git checkout --theirs uv.lock   # or just remove the conflicted file
  uv lock
  git add uv.lock
  git commit -m "chore: regenerate uv.lock after merge"
  ```
- **Add a lockfile-freshness gate in CI** (`uv lock --check`) so a `pyproject.toml` change that nobody re-locked fails fast instead of silently drifting.

---

## 5. UV in an Air-Gapped Environment

In a closed corporate network, UV needs to be told, explicitly, that there is no public internet: no PyPI, no GitHub releases, no ad-hoc Python downloads.

### Installing the UV binary without internet access

UV ships as a single static binary via GitHub Releases. On a machine that does have internet access:

```bash
# Download from https://github.com/astral-sh/uv/releases/latest
# e.g. uv-x86_64-unknown-linux-gnu.tar.gz
tar -xzf uv-x86_64-unknown-linux-gnu.tar.gz
```

Then transfer the extracted binary to the internal artifact store / internal server and install it:

```bash
sudo cp uv /usr/local/bin/uv
sudo chmod +x /usr/local/bin/uv
uv --version
```

Pin a specific version for CI image stability (e.g. `uv==0.7.0` or the equivalent tagged release binary) rather than always pulling "latest," so a new UV release can't silently change resolution behavior mid-pipeline.

### Configuring a private index

UV must point at an internal package index instead of `pypi.org`. Any internal PyPI-compatible mirror works — GitLab Package Registry, Artifactory, Nexus, devpi, or a `bandersnatch` mirror.

```toml
# pyproject.toml (workspace root)
[[tool.uv.index]]
name = "internal"
url = "https://pypi.internal.company.com/simple/"
default = true
authenticate = "always"

# Only if public packages are explicitly allowed as a fallback —
# otherwise omit this block entirely for maximum isolation.
[[tool.uv.index]]
name = "pypi"
url = "https://pypi.org/simple/"
```

For GitLab Package Registry specifically:

```toml
[[tool.uv.index]]
name = "corporate-pypi"
url = "https://gitlab.company.internal/api/v4/groups/MY_GROUP_ID/-/packages/pypi/simple"
publish-url = "https://gitlab.company.internal/api/v4/projects/MY_PROJECT_ID/packages/pypi"
default = true
```

For maximum isolation (no fallback to any external index at all):

```toml
[tool.uv]
no-index = true
```

> **Critical for Artifactory/Nexus-style mirrors:** disable "Forward PyPI package requests to PyPI.org" (or the equivalent upstream-proxy setting) on the internal repository. Without this, a package missing from the internal mirror silently falls through to the public internet — which defeats the point of an air-gapped setup and can fail outright if there's no route out.

### Authentication

Never commit credentials into `pyproject.toml`. Supply them at runtime:

```bash
# Environment variables — the recommended approach for CI
export UV_INDEX_INTERNAL_USERNAME="ci-user"
export UV_INDEX_INTERNAL_PASSWORD="$CI_REGISTRY_TOKEN"
```

Or via `.netrc`:

```bash
echo "machine pypi.internal.company.com login ci-user password $TOKEN" >> ~/.netrc
```

Or a machine-level `uv.toml` that never lives in the repo:

```toml
# /etc/uv/uv.toml (on the build/CI host, not in git)
[[index]]
url = "https://pypi.internal.company.com/simple/"
authenticate = "always"

[network]
offline = false
retries = 3
```

### Environment variables for air-gapped operation

These matter regardless of which CI system runs them — they are UV settings, not pipeline-specific YAML:

```bash
export UV_PYTHON_DOWNLOADS="never"     # never try to fetch a Python interpreter from the internet
export UV_NO_MANAGED_PYTHON="1"        # use the system/image Python instead of UV-managed ones
export UV_HTTP_TIMEOUT="60"            # internal networks can be slower/higher-latency than expected
export UV_INDEX_URL="https://pypi.internal.company.com/simple/"
```

### Caching wheels locally

`UV_CACHE_DIR` controls where UV keeps downloaded and built wheels. In any CI system, persist this directory between runs (keyed on `uv.lock`) so that a cold pipeline still gets fast, offline-safe installs from the local cache rather than re-resolving from the index every time:

```bash
export UV_CACHE_DIR=".uv-cache"
```

```bash
# Prune stale cache entries in CI to avoid unbounded growth
uv cache prune --ci

# Inspect what's cached
uv cache dir
```

The one setting that trips up nearly every team the first time: `UV_LINK_MODE`. UV defaults to hard-linking wheels from its cache into a project's virtual environment for speed. Most containerized CI runners mount the cache directory and the build workspace on different filesystems/mountpoints, and hard links cannot cross mountpoints — the install then fails (or silently falls back slowly, depending on version). Set this explicitly wherever UV runs inside a container:

```bash
export UV_LINK_MODE="copy"
```

CI-wide YAML wiring for these variables belongs in file 06; the point here is that they are UV configuration knobs, not GitLab-specific ones — the same variables apply whether the runner is GitLab, another CI system, or a developer's own air-gapped VM.

---

## 6. Pitfalls and the @nxlv/python Bridge

### Build backend has no default

Poetry bundled its own build backend (`poetry-core`). UV has no opinion — a backend must be chosen explicitly:

| Backend | When to use it |
|---|---|
| `hatchling` | Default recommendation for ordinary pure-Python libraries |
| `setuptools` | Packages with C extensions or legacy build requirements |
| `flit-core` | Very small, minimal packages |

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_package"]
```

### Version-specifier syntax changes

Poetry's caret (`^0.27`) has no UV equivalent — it must be translated to an explicit PEP 440 range (`>=0.27`, and an upper bound only if genuinely needed). Skipping this during migration is the most common source of "it installed something different than before" surprises.

### `requires-python` intersection across the workspace

Covered in [Section 4](#4-managing-many-dependency-versions-safely-across-sub-projects) — UV computes the intersection of every member's `requires-python`, not the union. A stray `>=3.10` in one legacy package can force the whole workspace to a lower floor than intended, or a stray `>=3.12` can force everyone else up.

### `optional-dependencies` and `scripts` move, but keep the same shape

```toml
# Poetry
[tool.poetry.extras]
redis = ["redis"]

[tool.poetry.scripts]
my-cli = "my_package.cli:main"

# UV / PEP 621
[project.optional-dependencies]
redis = ["redis>=5.0"]

[project.scripts]
my-cli = "my_package.cli:main"
```

Install extras with `uv add "lib-a[redis]"`.

### `UV_LINK_MODE` in containers

Already covered in [Section 5](#5-uv-in-an-air-gapped-environment) — worth repeating because it is, empirically, the single most common UV-in-CI failure: `Failed to hardlink files ... to ...`. Fix is always the same: `UV_LINK_MODE=copy`.

### Editable/workspace installs not resolving in CI

**Symptom:** a workspace member can't find another member's code at import time, even though it works locally.

**Cause:** usually a missing or misspelled `[tool.uv.workspace]` `members` glob, or a package that was added to the repo but never declared as a workspace member.

**Fix:** confirm the root `pyproject.toml` glob actually matches the new package's path, then re-run `uv sync --frozen` from a clean checkout (not an already-populated `.venv`) to catch what a fresh CI runner would actually see.

### The @nxlv/python bridge (details in file 02)

If this monorepo is also adopting NX, note briefly how the two tools interlock — full NX mechanics, `project.json` targets, and executor configuration are covered in file 02, but the UV-side contract is:

- `@nxlv/python` discovers workspace dependencies by scanning each member's `pyproject.toml` for `[tool.uv.sources]` entries with `{ workspace = true }`. **If a workspace dependency isn't declared this way, NX cannot see the edge in its project graph and `nx affected` will miss it.**
- Set `packageManager: "uv"` in the `@nxlv/python` plugin options in `nx.json` so it invokes `uv` rather than defaulting to Poetry.
- `inferDependencies: true` additionally scans Python `import` statements to catch dependencies that exist in code but aren't yet declared in `pyproject.toml` — useful as a safety net during migration, not a replacement for declaring `tool.uv.sources` correctly.
- Executors that shell out to Python (`@nxlv/python:run-commands`) activate the workspace's `.venv` before running; a bare `nx:run-commands` calling `uv run ...` directly does not depend on this and works either way, but mixing the two conventions inconsistently across `project.json` files is a common source of "works in one package, not another" confusion.

---

## 7. UV Migration Checklist

- [ ] UV installed on every developer machine and in the CI image (binary pinned to a known version, not "latest")
- [ ] Root `pyproject.toml` created with `[tool.uv.workspace]` defining `members` (and `exclude` for anything not yet migrated)
- [ ] Every sub-project's `pyproject.toml` converted from `[tool.poetry]` to PEP 621 `[project]`
- [ ] `[tool.poetry.group.*.dependencies]` converted to `[dependency-groups]`
- [ ] Build backend chosen and declared (`hatchling`/`setuptools`/`flit-core`) for every package
- [ ] Internal cross-package dependencies declared under `[tool.uv.sources]` with `{ workspace = true }`
- [ ] Version specifiers converted from Poetry's caret syntax to PEP 440 ranges
- [ ] `uv.lock` generated at the workspace root (`uv lock`) and committed
- [ ] `uv sync` and `uv run pytest` succeed for every package from a clean checkout
- [ ] `poetry.lock` and any `[tool.poetry]` sections removed once the pilot package is verified
- [ ] Private index configured in `[[tool.uv.index]]`, with credentials supplied only via environment variables or `.netrc` — never committed
- [ ] "Forward PyPI requests" (or equivalent upstream-proxy setting) disabled on the internal registry, if full air-gap isolation is required
- [ ] `UV_LINK_MODE=copy` set everywhere UV runs inside a container
- [ ] `UV_PYTHON_DOWNLOADS=never` and `UV_NO_MANAGED_PYTHON=1` set for air-gapped runners
- [ ] `UV_CACHE_DIR` persisted/cached between CI runs, keyed on `uv.lock`
- [ ] A `uv lock --check` gate added in CI so a stale lockfile fails fast
- [ ] CI/local scripts updated from `poetry install`/`poetry run` to `uv sync --frozen`/`uv run`
- [ ] If using NX: `packageManager: "uv"` set in the `@nxlv/python` plugin options, and every workspace edge verified via `nx graph`
