# NX Monorepo Management

> **Audience:** Developers and DevOps engineers working in a large Python + Vue.js monorepo on self-hosted GitLab CI/CD, inside an air-gapped corporate network.
> **Scope:** What NX is, its core concepts, how to introduce it into an existing (brownfield) monorepo, how it integrates with Python through `@nxlv/python`, the Affected algorithm, caching (local and self-hosted remote), the dependency graph, and common pitfalls.
> **Out of scope:** UV package-manager mechanics (see `03-uv-package-management.md`), Ruff and the custom formatter (see `04-ruff-and-formatting.md`), NX Release / versioning / publishing (see `05-nx-release-and-versioning.md`), and GitLab CI pipeline YAML, Docker images, and registry configuration (see `06-gitlab-ci-integration.md`).

---

## Table of Contents

1. [What Is NX and Why It Fits This Monorepo](#1-what-is-nx-and-why-it-fits-this-monorepo)
2. [Core NX Concepts](#2-core-nx-concepts)
3. [Adding NX to an Existing (Brownfield) Monorepo](#3-adding-nx-to-an-existing-brownfield-monorepo)
4. [NX for Python via `@nxlv/python`](#4-nx-for-python-via-nxlvpython)
5. [NX Affected](#5-nx-affected)
6. [`project.json` Structure](#6-projectjson-structure)
7. [`nx.json` Structure](#7-nxjson-structure)
8. [Tags and Implicit Dependencies](#8-tags-and-implicit-dependencies)
9. [The Dependency Graph](#9-the-dependency-graph)
10. [NX Cache: Local and Self-Hosted Remote](#10-nx-cache-local-and-self-hosted-remote)
11. [Distributed Task Execution (Agents)](#11-distributed-task-execution-agents)
12. [Common Pitfalls](#12-common-pitfalls)

---

## 1. What Is NX and Why It Fits This Monorepo

NX is a **smart build system for monorepos**, originally built by Nrwl. It does not replace your existing tools (UV, pytest, Vite, Ruff) — it wraps them and adds an intelligence layer on top:

- **Project graph** — automatically analyzes the dependencies between every project in the monorepo.
- **Affected detection** — instead of running tests/builds against the entire codebase, NX identifies exactly what changed and what depends on it.
- **Computation cache** — a task is never re-run with the same inputs; the previous result is replayed instantly.
- **Task orchestration** — runs tasks in parallel, in the correct dependency order.
- **Release management** (see file 05) — selective, per-project versioning and publishing.

NX itself is a Node.js/TypeScript tool, but it is language-agnostic: it can orchestrate arbitrary shell commands, which is exactly what makes it a good fit for a Python + Vue.js monorepo. Vue/Vite get first-class official plugins (`@nx/vue`, `@nx/vite`); Python is supported through the community plugin `@nxlv/python` (see [Section 4](#4-nx-for-python-via-nxlvpython)) or, at minimum, through generic `nx:run-commands` targets.

**Why this matters at scale:**

| Problem with the current setup | What NX changes |
|---|---|
| Every CI run builds/tests the whole monorepo | `nx affected` runs only what actually changed |
| No visibility into cross-project dependencies | `nx graph` renders the dependency graph |
| Repeated work re-executed on unchanged code | Local + remote computation cache |
| No enforced architectural boundaries | Tags + module-boundary rules |

---

## 2. Core NX Concepts

### Mental model

```
Workspace (the entire Git repository)
├── Projects (a single app or library)
│   ├── Targets (an actionable task: build/test/lint/publish)
│   │   ├── Executor   — what actually runs the task
│   │   ├── Options    — task-specific settings
│   │   ├── Inputs     — what affects cache invalidation
│   │   └── Outputs    — what gets cached
│   └── Tags (organizational metadata)
└── nx.json (workspace-level configuration)
```

NX builds a **Project Graph** — a directed graph of the dependencies between projects. From the Project Graph it derives a **Task Graph**, which determines the order tasks must run in.

### Workspace

The workspace is the whole Git repository. NX identifies it by the presence of `nx.json` at the root. Any folder containing a `project.json` (or, for JS/TS projects, a `package.json` with an `nx` key) is treated as a project.

### Projects

A project is a logical unit inside the workspace — a Python library, a Vue application, a backend service. Each project has:
- a unique **name**
- a **root** directory
- a **type**: `application` or `library`
- **tags** for organization
- **targets** it exposes

### Targets

A target is an action that can be run against a project: `build`, `test`, `lint`, `format`, `publish`, etc.

```bash
# Run a specific target on a specific project
nx run my-python-lib:test

# Run a target across all projects
nx run-many -t test

# Run a target only on what changed
nx affected -t test
```

### Executors

An executor is the "engine" that runs a target:

1. **Built-in executors** — e.g. `@nx/vite:build`, `@nx/vite:test`.
2. **`nx:run-commands`** — runs an arbitrary shell command. This is the most useful executor for Python projects that don't have a dedicated plugin.
3. **`@nxlv/python:*` executors** — purpose-built executors provided by the `@nxlv/python` plugin (build, lint, publish, run-commands, etc.) that understand the Python/UV project layout.

```json
{
  "name": "my-python-lib",
  "targets": {
    "test": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run pytest",
        "cwd": "{projectRoot}"
      }
    },
    "lint": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run ruff check .",
        "cwd": "{projectRoot}"
      }
    }
  }
}
```

### Generators

A generator is a script (TypeScript) that scaffolds or modifies code — creating a new project with the correct standardized layout, adding a feature to an existing one, or enforcing organizational conventions.

```bash
# Create a new Vue application
nx g @nx/vue:application apps/my-app --bundler=vite --style=scss

# Create a shared Vue library
nx g @nx/vue:library libs/ui-components --publishable --importPath=@myorg/ui

# Create a new Python project via @nxlv/python
nx g @nxlv/python:uv-project my-new-lib --directory=libs/python/my-new-lib --projectType=library
```

### Plugins

A plugin extends NX with new executors, generators, and **inference** — automatic detection of projects and targets from configuration files it understands (e.g. `pyproject.toml`, `package.json`, `vite.config.ts`).

| Plugin | Origin | Purpose |
|---|---|---|
| `@nxlv/python` | Community (Lucas Vieira) | The central Python plugin — generators, executors, UV/Poetry integration, project-graph inference, release support |
| `@nx/vue` | Official (Nrwl) | Vue.js generators and executors |
| `@nx/vite` | Official (Nrwl) | Vite build/test/dev-server |
| `@nx/eslint` | Official (Nrwl) | ESLint integration for JS/TS/Vue |

There is **no official `@nx/python` plugin** from Nrwl — Python support in the NX ecosystem is entirely community-driven through `@nxlv/python`. Without it, NX still works for Python via plain `nx:run-commands` targets, but you lose automatic Python dependency inference, the dedicated `ruff`/`build`/`publish` executors, and the Python release integration.

---

## 3. Adding NX to an Existing (Brownfield) Monorepo

### Prerequisites

```bash
node --version   # >= 18.0
npm --version    # >= 9.0
```

NX itself is JavaScript/TypeScript tooling. It manages the Node.js side of the workspace (NX, Vue, Vite) and additionally orchestrates your existing Python tooling — it is the "coordination layer" between the two ecosystems.

### Step 1 — Initialize NX

```bash
# From the monorepo root
npx nx@latest init
```

This creates:
- `nx.json` — workspace configuration
- `package.json` (if one doesn't exist yet) with `nx` as a dev dependency

### Step 2 — Install the plugins you need

```bash
npm install -D \
  nx@latest \
  @nxlv/python@latest \
  @nx/vue@latest \
  @nx/vite@latest \
  @nx/eslint@latest
```

> **Version note:** these guides are written against NX 22+, with `@nxlv/python` in the 22.x line and `@nx/vue`/`@nx/vite` in the 21.x line. Pin exact versions in `package.json` and verify compatibility before upgrading.

### Step 3 — Write a baseline `nx.json`

Start minimal, then iterate (see [Section 7](#7-nxjson-structure) for the full reference):

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "defaultBase": "main",
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/**/*.spec.py",
      "!{projectRoot}/**/test_*.py",
      "!{projectRoot}/tests/**/*",
      "!{projectRoot}/**/*.md"
    ],
    "sharedGlobals": [
      "{workspaceRoot}/nx.json",
      "{workspaceRoot}/uv.lock",
      "{workspaceRoot}/pyproject.toml"
    ]
  },
  "targetDefaults": {
    "build": { "cache": true, "dependsOn": ["^build"], "inputs": ["production", "^production"] },
    "test": { "cache": true, "inputs": ["default", "^production"] },
    "lint": { "cache": true, "inputs": ["default", "{workspaceRoot}/ruff.toml"] }
  },
  "parallel": 4
}
```

### Step 4 — Add a `project.json` to every existing project

Every folder that should become an NX project needs a `project.json`. For a Python library that does not yet use `@nxlv/python`:

```json
{
  "name": "my-python-lib",
  "root": "packages/my-python-lib",
  "sourceRoot": "packages/my-python-lib/src",
  "projectType": "library",
  "tags": ["lang:python", "scope:shared"],
  "targets": {
    "build": {
      "executor": "nx:run-commands",
      "options": { "command": "uv build", "cwd": "{projectRoot}" },
      "outputs": ["{projectRoot}/dist"],
      "cache": true
    },
    "test": {
      "executor": "nx:run-commands",
      "options": { "command": "uv run pytest tests/ -v --tb=short", "cwd": "{projectRoot}" },
      "cache": true,
      "inputs": ["{projectRoot}/**/*.py", "{projectRoot}/pyproject.toml"]
    },
    "lint": {
      "executor": "nx:run-commands",
      "options": { "command": "uv run ruff check .", "cwd": "{projectRoot}" },
      "cache": true
    }
  }
}
```

This can be done manually for a handful of projects, or bulk-generated with a small script that walks every `pyproject.toml` in the repo. If you adopt `@nxlv/python`, its generator produces this file (with the correct executors) for you — see [Section 4](#4-nx-for-python-via-nxlvpython).

### Step 5 — Update `.gitignore`

```
# NX
.nx/cache
.nx/workspace-data

# Python
__pycache__/
*.pyc
.venv/
dist/
*.egg-info/
```

### Step 6 — Validate the workspace

```bash
# Confirm NX sees every project
npx nx show projects

# Inspect the dependency graph
npx nx graph

# Sanity-check a single project
npx nx run my-python-lib:test
```

### Suggested rollout order

- [ ] Initialize NX and commit a minimal `nx.json`
- [ ] Add `project.json` to every existing app/library
- [ ] Verify `nx show projects` lists every expected project
- [ ] Verify `nx graph` shows the correct dependency edges
- [ ] Wire `nx affected` into CI in parallel with the existing pipeline (non-blocking) before cutting over
- [ ] Add caching (local first, then remote — [Section 10](#10-nx-cache-local-and-self-hosted-remote))
- [ ] Introduce tags and boundary rules ([Section 8](#8-tags-and-implicit-dependencies))

---

## 4. NX for Python via `@nxlv/python`

### What the plugin bridges

`@nxlv/python` is the reason NX and Python work together at all. Without it:
- NX does not know a given folder is a Python project.
- NX cannot build a project graph from `pyproject.toml`.
- NX has no dedicated way to run Python build/test/lint correctly (venv activation, UV/Poetry semantics).

**What it provides:**

```
@nxlv/python
├── Generators
│   ├── uv-project     — scaffold a new Python project using UV
│   ├── poetry-project  — scaffold a new Python project using Poetry
│   └── pkg-sync        — sync detected imports into pyproject.toml (experimental)
│
├── Executors
│   ├── @nxlv/python:build          — build wheel/sdist
│   ├── @nxlv/python:publish        — publish to PyPI / a private registry
│   ├── @nxlv/python:install        — install dependencies
│   ├── @nxlv/python:lock           — update the lockfile
│   ├── @nxlv/python:sync           — sync the environment from the lockfile
│   ├── @nxlv/python:ruff           — run Ruff lint
│   └── @nxlv/python:run-commands   — run a command with the project's venv activated
│
└── NX integration
    └── inferDependencies: true  — build project-graph edges from pyproject.toml
```

### Installing and enabling it

```bash
npm install @nxlv/python --save-dev
```

```json
// nx.json
{
  "plugins": [
    {
      "plugin": "@nxlv/python",
      "options": {
        "packageManager": "uv",
        "inferDependencies": true
      }
    }
  ],
  "sync": {
    "globalGenerators": ["@nxlv/python:pkg-sync"]
  }
}
```

### Generating new Python projects

```bash
# Library
npx nx generate @nxlv/python:uv-project my-new-lib \
  --directory=libs/python/my-new-lib \
  --projectType=library \
  --linter=ruff \
  --unitTestRunner=pytest \
  --publishable

# Application
npx nx generate @nxlv/python:uv-project my-new-app \
  --directory=apps/my-new-app \
  --projectType=application \
  --linter=ruff \
  --unitTestRunner=pytest
```

The generator produces a correctly-shaped `pyproject.toml`, a `project.json` with all standard targets wired to `@nxlv/python` executors, and the source/test skeleton.

### How the plugin builds project-graph edges from Python

1. `@nxlv/python` scans every workspace member's `pyproject.toml`.
2. It looks for `[tool.uv.sources]` entries marked `{ workspace = true }` — these are intra-workspace dependencies.
3. It adds an edge in the Project Graph for each one: if `api-service` depends on `shared-utils`, an edge is created.
4. NX then propagates change detection through the graph: a change in `shared-utils` marks `api-service` as affected.

```toml
# apps/api-service/pyproject.toml
dependencies = ["shared-utils"]

[tool.uv.sources]
shared-utils = { workspace = true }
```

### `inferDependencies: true` — deeper, import-based detection

With `inferDependencies: true`, the plugin additionally:
1. Scans Python source files for `import` statements.
2. Detects imports of other workspace members.
3. Warns when an import exists but the dependency isn't declared in `pyproject.toml`.

Combined with the `@nxlv/python:pkg-sync` generator:

```bash
# Check what's missing
npx nx sync:check

# Add missing declared dependencies automatically
npx nx sync
```

Example: if `api-service` has `from shared_utils import helper` but `shared-utils` is not declared as a dependency, `nx sync` adds it automatically.

### The critical executor detail: `@nxlv/python:run-commands` vs `nx:run-commands`

> **Always use `@nxlv/python:run-commands` (not the generic `nx:run-commands`) for Python targets.** The plugin's executor activates the project's virtual environment before running the command; the generic one does not. Using the wrong one is one of the most common sources of "module not found" / "command not found" failures in a fresh setup.

```json
{
  "targets": {
    "test": {
      "executor": "@nxlv/python:run-commands",
      "options": {
        "command": "uv run --frozen pytest src/ tests/ -v --cov=src",
        "cwd": "libs/python/shared-utils"
      },
      "cache": true,
      "outputs": ["{projectRoot}/coverage.xml", "{projectRoot}/.pytest_cache"]
    }
  }
}
```

---

## 5. NX Affected

### How it works

`nx affected` performs three steps:

1. **Git diff** — determines which files changed between two commits (`--base` and `--head`).
2. **Project mapping** — maps every changed file to the project it belongs to.
3. **Dependency propagation** — walks the Project Graph and marks every project that transitively depends on a changed project as "affected" too.

```
Change in: libs/python/shared-utils
    ↓
NX identifies affected:
  - apps/api-service      (depends on shared-utils)
  - apps/data-pipeline    (depends on shared-utils)
  - libs/python/auth-helpers (depends on shared-utils)
```

Given a graph `lib-shared ← lib-a ← service-x` and `lib-a ← api-server`:
- a change in `lib-shared` marks `lib-shared`, `lib-a`, `service-x`, and `api-server` as affected
- a change in `lib-a` only marks `lib-a`, `service-x`, and `api-server`

### Everyday commands

```bash
# Show which projects are affected (no execution)
nx show projects --affected --base=main --head=HEAD

# Show the affected sub-graph
nx affected:graph
nx graph --affected

# Run one target on everything affected
nx affected -t build

# Run several targets, in parallel
nx affected -t lint test build --parallel=3

# Dry run — see what would execute without running it
nx affected -t build --dry-run

# Include uncommitted working-tree changes
nx affected -t test --uncommitted
```

### Base/head in CI

`nx affected` needs a `--base` and `--head` commit to diff against. In GitLab CI, these map onto GitLab's own pipeline variables:

| GitLab variable | Meaning | Used as |
|---|---|---|
| `CI_COMMIT_SHA` | the current commit | `NX_HEAD` |
| `CI_MERGE_REQUEST_DIFF_BASE_SHA` | the MR's base commit | `NX_BASE` in an MR pipeline |
| `CI_COMMIT_BEFORE_SHA` | the commit before this push | `NX_BASE` in a branch-push pipeline |

```bash
export NX_HEAD="$CI_COMMIT_SHA"
export NX_BASE="${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}"
```

A shallow clone breaks this: if the runner's `GIT_DEPTH` doesn't reach far enough back, NX cannot compute a meaningful diff and falls back to treating everything as affected. Fetch enough history (`GIT_DEPTH: "0"` for a full clone, or a generous fixed depth) — full CI-pipeline wiring for this lives in file 06.

### Filtering affected output by tag

```bash
nx affected -t test --projects="tag:lang:python"
nx affected -t build --projects="tag:team:backend"
```

---

## 6. `project.json` Structure

Every field, annotated:

```json
{
  // Unique project identifier in the workspace
  "name": "my-library",

  // Project root, relative to the workspace root
  "root": "packages/my-library",

  // Source directory
  "sourceRoot": "packages/my-library/src",

  // "application" or "library"
  "projectType": "library",

  // Organizational tags — see Section 8
  "tags": ["lang:python", "scope:core", "type:lib"],

  // Dependencies NX cannot infer on its own ("!" excludes one)
  "implicitDependencies": ["shared-config", "!legacy-module"],

  "targets": {
    "build": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv build --no-sources",
        "cwd": "{projectRoot}",
        "env": { "PYTHONPATH": "{workspaceRoot}/packages" }
      },

      // Named configurations (e.g. production vs. development)
      "configurations": {
        "production": { "options": { "command": "uv build --no-sources --wheel" } }
      },
      "defaultConfiguration": "production",

      // Tasks that must run first
      "dependsOn": ["^build", "lint"],

      "cache": true,

      // What affects the cache key
      "inputs": [
        "{projectRoot}/src/**/*.py",
        "{projectRoot}/pyproject.toml",
        "^production",
        { "env": "BUILD_VERSION" },
        { "externalDependencies": ["uv", "python"] }
      ],

      // What gets stored in the cache
      "outputs": ["{projectRoot}/dist/**"]
    },

    "test": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run pytest tests/ -v --junitxml=test-results.xml",
        "cwd": "{projectRoot}"
      },
      "cache": true,
      "inputs": [
        "{projectRoot}/**/*.py",
        "!{projectRoot}/**/__pycache__/**",
        "{projectRoot}/pyproject.toml",
        "{workspaceRoot}/uv.lock"
      ],
      "outputs": ["{projectRoot}/test-results.xml", "{projectRoot}/.coverage"]
    }
  }
}
```

`^build` means "the `build` target of every project this one depends on" — it is what enforces correct build ordering across the graph.

---

## 7. `nx.json` Structure

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",

  // Base branch used to compute affected
  "defaultBase": "main",

  // Max parallel tasks
  "parallel": 4,

  "cacheDirectory": ".nx/cache",
  "maxCacheSize": "10GB",

  // Reusable groups of input globs
  "namedInputs": {
    "default": [
      "{projectRoot}/**/*",
      "!{projectRoot}/**/__pycache__/**",
      "!{projectRoot}/**/*.pyc",
      "!{projectRoot}/.venv/**",
      "sharedGlobals"
    ],
    "production": [
      "default",
      "!{projectRoot}/**/*.spec.py",
      "!{projectRoot}/**/test_*.py",
      "!{projectRoot}/tests/**",
      "!{projectRoot}/**/*.md"
    ],
    "sharedGlobals": [
      "{workspaceRoot}/nx.json",
      "{workspaceRoot}/uv.lock",
      "{workspaceRoot}/pyproject.toml"
    ]
  },

  // Defaults applied to every project's target of the same name
  "targetDefaults": {
    "build": { "dependsOn": ["^build"], "cache": true, "inputs": ["production", "^production"] },
    "test": { "cache": true, "inputs": ["default", "^production"] },
    "lint": {
      "cache": true,
      "inputs": ["default", "{workspaceRoot}/ruff.toml", "{workspaceRoot}/pyproject.toml"]
    },
    "e2e": { "cache": false }
  },

  "plugins": [
    { "plugin": "@nxlv/python", "options": { "packageManager": "uv", "inferDependencies": true } },
    "@nx/vue/plugin",
    "@nx/vite/plugin"
  ],

  "generators": {
    "@nxlv/python:uv-project": { "linter": "ruff", "unitTestRunner": "pytest" },
    "@nx/vue:application": { "bundler": "vite", "style": "scss" }
  }
}
```

`release` configuration (versioning, changelogs, publishing) also lives in `nx.json` but is covered in full in file 05, not here.

---

## 8. Tags and Implicit Dependencies

### Tags

Tags are metadata labels attached to a project in `project.json`. They serve three purposes:

1. **Organization** — who owns what.
2. **Enforcement** — which projects are allowed to depend on which (module boundaries).
3. **Filtering** — running `nx affected`/`nx run-many` against a subset of the graph.

Common convention: `category:value`.

```json
{
  "tags": [
    "lang:python",
    "scope:shared",
    "type:lib",
    "team:backend",
    "visibility:internal"
  ]
}
```

```bash
# Run tests only on Python projects
nx run-many -t test --projects="tag:lang:python"

# Build only what belongs to the backend team, among affected projects
nx affected -t build --projects="tag:team:backend"

# Run everything except e2e
nx run-many -t lint --exclude="tag:type:e2e"
```

### Module boundary enforcement

For TypeScript/Vue projects, `@nx/enforce-module-boundaries` (an ESLint rule) enforces dependency constraints based on tags:

```json
{
  "rules": {
    "@nx/enforce-module-boundaries": [
      "error",
      {
        "depConstraints": [
          { "sourceTag": "type:app", "onlyDependOnLibsWithTags": ["type:lib", "type:util"] },
          { "sourceTag": "scope:core", "notDependOnLibsWithTags": ["scope:data"] },
          { "sourceTag": "scope:frontend", "onlyDependOnLibsWithTags": ["scope:frontend", "scope:shared"] }
        ]
      }
    ]
  }
}
```

For Python, there is no equivalent built-in NX rule; boundary enforcement has to be implemented as a custom lint/conformance check (or a `nx sync`-style script) if it's required.

### Implicit dependencies

Use `implicitDependencies` when NX cannot infer a dependency on its own — for example, a shared config file with no explicit import, or a case where a change in project X should trigger a rebuild of Y even without a direct code reference:

```json
{
  "name": "backend-service",
  "implicitDependencies": ["shared-config", "core-lib", "!legacy-module"]
}
```

`!legacy-module` explicitly overrides/excludes an inferred dependency.

---

## 9. The Dependency Graph

### Viewing it

```bash
# Open the interactive graph in a browser
npx nx graph

# Focus on one project
npx nx graph --focus=core-lib

# Save as JSON for tooling/CI artifacts
npx nx graph --file=dependency-graph.json

# Graph for a specific task run
npx nx affected -t test --graph
```

### What it shows

- **Projects** as nodes.
- **Dependencies** as directed edges.
- **Affected** projects highlighted, when run with `--affected` or as part of `nx affected ... --graph`.

### How teams should use it

- **Before onboarding a new service**, check the graph to see what it will realistically depend on and what already depends on similar libraries.
- **Before releasing or refactoring a shared library**, focus the graph on it (`nx graph --focus=core-lib`) and read the incoming edges — that's the blast radius.
- **To catch circular dependencies** — NX will flag cycles directly in the graph view (and `nx affected`/build will fail or time out if one exists). The fix is always structural: extract the shared code into a third, lower-level package.
- **In CI**, export the graph as a build artifact (`nx graph --file=graph.json`) on every pipeline run so architecture drift is visible in review, not just locally.
- **Treat "no incoming edges" as a signal**, not a red flag by itself — it usually just means the project is independently publishable/deployable.

---

## 10. NX Cache: Local and Self-Hosted Remote

### How the cache works

On every target run, NX:
1. Computes a hash from: input files, config files, environment variables, and dependency versions.
2. Looks up that hash in the cache.
3. On a **hit**, replays the stored result immediately (no execution).
4. On a **miss**, executes the task and stores the result.

```bash
# Local cache lives under .nx/cache
npx nx reset             # clear the local cache
npx nx run my-lib:test --skip-nx-cache   # force a real run, bypass cache
```

> Cache only applies to **deterministic** tasks — the same inputs must always produce the same outputs. Never cache a task with an external side effect (publishing, deployment, database migration): set `"cache": false` on it explicitly.

### Remote cache without NX Cloud

Since this environment is air-gapped, NX Cloud (the hosted remote-cache/DTE service) is not an option. Two self-hosted alternatives:

#### Option A: `@nx/s3-cache` backed by MinIO (recommended)

MinIO is an S3-compatible object store that runs entirely on-premises.

```yaml
# docker-compose.minio.yml
services:
  minio:
    image: minio/minio:latest   # mirror this image into your internal registry
    ports: ["9000:9000", "9001:9001"]
    volumes: ["minio-data:/data"]
    environment:
      MINIO_ROOT_USER: nx-cache-admin
      MINIO_ROOT_PASSWORD: "CHANGE_ME_STRONG_PASSWORD"
    command: server /data --console-address ":9001"
volumes:
  minio-data:
```

```json
// nx.json
{
  "s3": {
    "region": "us-east-1",
    "bucket": "nx-remote-cache",
    "endpoint": "http://minio.company.internal:9000",
    "forcePathStyle": true,
    "disableChecksum": true,
    "cacheKeyPrefix": "my-monorepo",
    "localMode": "read-only",
    "ciMode": "read-write"
  }
}
```

```bash
npm install -D @nx/s3-cache
```

Credentials (`AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`) are supplied as masked CI/CD variables, never committed.

> **Security warning — CVE-2025-36852 ("CREEP"):** bucket-based remote caches are vulnerable to cache poisoning. An MR opened by any contributor can write a malicious cache entry that later gets replayed into a production build. Mitigate by making cache writes **read-only from MR pipelines** and **read-write only from protected branches** (`localMode: "read-only"`, `ciMode: "read-write"`, plus pipeline-level restrictions on which jobs get write credentials at all).

#### Option B: `portable-nx-cache` (self-hosted cache server over GitLab CI cache)

A lightweight alternative that runs a small cache server process inside the job and persists its storage through GitLab's own CI cache mechanism, rather than requiring a separate object store:

```yaml
.nx-portable-cache:
  variables:
    PORTABLE_NX_CACHE_PORT: "8080"
    NX_SELF_HOSTED_REMOTE_CACHE_SERVER: "http://localhost:8080"
    NX_SELF_HOSTED_REMOTE_CACHE_ACCESS_TOKEN: "$NX_CACHE_TOKEN"
  before_script:
    - curl -fsSL https://artifactory.company.internal/tools/portable-nx-cache-linux-amd64 -o /tmp/portable-nx-cache
    - chmod +x /tmp/portable-nx-cache
    - PORTABLE_NX_CACHE_DIR=.portable-nx-cache PORTABLE_NX_CACHE_PORT=$PORTABLE_NX_CACHE_PORT /tmp/portable-nx-cache &
    - until curl -sf http://localhost:$PORTABLE_NX_CACHE_PORT/ready; do sleep 1; done
```

Both options avoid any dependency on NX Cloud, keeping the cache entirely inside the corporate network.

### Controlling cache invalidation with `namedInputs`

A common trap: putting a workspace-wide file like `uv.lock` directly into every project's `default` inputs invalidates the cache for the *entire* monorepo on every dependency bump. Isolate it into `sharedGlobals` and only pull that into targets that genuinely need it:

```json
{
  "namedInputs": {
    "default": ["{projectRoot}/**/*"],
    "sharedGlobals": ["{workspaceRoot}/uv.lock", "{workspaceRoot}/nx.json"]
  },
  "targetDefaults": {
    "test": { "inputs": ["default", "sharedGlobals"] }
  }
}
```

---

## 11. Distributed Task Execution (Agents)

NX **Distributed Task Execution (DTE)** splits the tasks in an `nx affected` run across multiple parallel workers ("agents") instead of running them serially (or only locally parallel) on one machine. In its full form — dynamic re-balancing of tasks across agents as they finish, plus automatic re-runs of flaky tasks — DTE is an NX Cloud feature and is **not available** without either NX Cloud or a self-hosted NX Cloud-compatible service.

In an air-gapped environment without NX Cloud, distribution has to be done **manually**, by statically partitioning work across separate CI jobs rather than dynamically balancing it. The simplest and most robust approach is to split by project type or tag and run each partition as its own job:

```yaml
build:python-affected:
  script:
    - npx nx affected -t build --base=$NX_BASE --head=$NX_HEAD --projects='libs/python/*,apps/api-*'

build:frontend-affected:
  script:
    - npx nx affected -t build --base=$NX_BASE --head=$NX_HEAD --projects='apps/frontend-*,libs/frontend/*'

test:python-affected:
  needs: [build:python-affected]
  script:
    - npx nx affected -t test --base=$NX_BASE --head=$NX_HEAD --projects='libs/python/*,apps/api-*'
```

This gives coarse-grained parallelism (by domain) at the cost of NX Cloud's fine-grained, dynamically-balanced task distribution. For most monorepos of this scale, combining this manual split with `--parallel=N` inside each job and a working remote cache ([Section 10](#10-nx-cache-local-and-self-hosted-remote)) recovers most of the practical benefit without depending on an external SaaS. Full CI-side wiring of these jobs belongs in file 06.

---

## 12. Common Pitfalls

### Cache misses despite no relevant change

**Symptom:** tasks re-run every time even though nothing that matters changed.
**Cause:** `inputs` defined too broadly — often including volatile files such as `__pycache__`, `.pytest_cache`, or coverage output.
**Fix:**
```json
"inputs": [
  "{projectRoot}/**/*.py",
  "!{projectRoot}/**/__pycache__/**",
  "!{projectRoot}/**/*.pyc",
  "!{projectRoot}/.venv/**"
]
```
Diagnose with `NX_VERBOSE_LOGGING=true nx test my-lib` to see exactly what changed the hash.

### A project isn't recognized by NX

**Symptom:** `nx show projects` (or `nx graph`) doesn't list an expected project.
**Causes:** missing `project.json`; the folder is excluded via `.gitignore`/`.nxignore`; a duplicate project name.
**Fix:** `npx nx show projects --verbose`, and `npx nx show project <name> --json` to inspect what NX actually parsed.

### Circular dependencies

**Symptom:** `nx graph` shows a cycle; affected/build hangs or fails.
**Fix:** there is no configuration fix — extract the shared code both projects need into a third, lower-level library.

### `uv.lock` (or any workspace-wide file) marks everything as affected

**Symptom:** any dependency bump triggers a full-workspace rebuild.
**Fix:** isolate workspace-wide files into a dedicated named input (`sharedGlobals`) and only include it in targets that truly depend on it — see [Section 10](#10-nx-cache-local-and-self-hosted-remote).

### Wrong build order

**Symptom:** a project builds before the library it depends on.
**Fix:** make sure `build` (and any similarly-ordered target) declares `"dependsOn": ["^build"]` in `targetDefaults`.

### A cacheable task has a side effect

**Symptom:** a task that mutates external state (e.g. a DB migration) gets silently skipped on a cache hit.
**Fix:** any task with side effects must be `"cache": false` explicitly — never rely on the default.

### NX doesn't detect a Python dependency

**Symptom:** a change in `core-lib` doesn't mark `data-utils` as affected even though it imports from it.
**Causes:** `@nxlv/python` isn't installed/enabled, or `inferDependencies` is off (it defaults to `false`), or `[tool.uv.sources]` doesn't declare `{ workspace = true }` for the internal dependency.
**Fix:**
```json
{ "plugins": [{ "plugin": "@nxlv/python", "options": { "packageManager": "uv", "inferDependencies": true } }] }
```
and/or add an explicit `implicitDependencies` entry as a fallback while the graph is verified.

### Using `nx:run-commands` instead of `@nxlv/python:run-commands`

**Symptom:** Python commands fail with "command not found" or import errors, even though they work when run manually.
**Cause:** the generic executor does not activate the project's virtual environment; the `@nxlv/python` one does.
**Fix:** always use `@nxlv/python:run-commands` for Python targets when the plugin is installed.

### Shallow clone breaks `nx affected` in CI

**Symptom:** `nx affected` behaves as if everything changed, every run.
**Cause:** the CI runner's git clone doesn't go back far enough for NX to diff against `NX_BASE`.
**Fix:** increase `GIT_DEPTH` (full CI variable configuration is covered in file 06).

### NX daemon issues in CI

**Symptom:** connection errors to the NX daemon in ephemeral CI containers.
**Fix:** disable it — `NX_DAEMON=false`. The daemon is a local-development optimization and generally should not run in short-lived CI jobs.

### Checklist: rolling out NX cleanly

- [ ] Every project has a valid `project.json` (or is picked up by plugin inference)
- [ ] `nx graph` accurately reflects real dependencies, with no unexpected cycles
- [ ] `inputs`/`outputs` are scoped tightly enough that cache hits are the common case
- [ ] Workspace-wide files are isolated into `sharedGlobals` rather than polluting every project's `default` inputs
- [ ] `@nxlv/python:run-commands` (not `nx:run-commands`) is used for every Python target, if the plugin is in use
- [ ] `NX_BASE`/`NX_HEAD` are computed correctly for both MR and branch-push pipelines, with sufficient `GIT_DEPTH`
- [ ] A remote cache (MinIO/S3-compatible or `portable-nx-cache`) is in place and write access is restricted to protected branches
- [ ] Tags are applied consistently and, where relevant, boundary rules are enforced
