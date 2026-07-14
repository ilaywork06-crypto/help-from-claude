# Ruff and the Custom Formatter

> **Audience:** Developers, tech leads, and DevOps/platform engineers working in a large Python + Vue.js monorepo on self-hosted GitLab CI/CD inside a closed (air-gapped) corporate network.
> **Scope:** Adopting Ruff as the single lint/format tool for the Python codebase, configuring it correctly for a monorepo, retiring flake8/black/isort, and building the organization's custom Python formatter script on top of Ruff. NX target mechanics are covered in `02-nx-monorepo-management.md`; GitLab CI YAML specifics are covered in `06-gitlab-ci-integration-and-operations.md`.

---

## Table of Contents

1. [What Ruff Is and Why We're Adopting It](#1-what-ruff-is-and-why-were-adopting-it)
2. [Installing Ruff in the Workspace](#2-installing-ruff-in-the-workspace)
3. [Ruff Configuration for a Monorepo](#3-ruff-configuration-for-a-monorepo)
4. [Migrating Off flake8 / black / isort](#4-migrating-off-flake8--black--isort)
5. [Ruff as an NX Target](#5-ruff-as-an-nx-target)
6. [The Custom Python Formatter](#6-the-custom-python-formatter)
7. [Reference Implementation](#7-reference-implementation)
8. [Known Pitfalls](#8-known-pitfalls)
9. [Checklist](#9-checklist)

---

## 1. What Ruff Is and Why We're Adopting It

Ruff is a Python linter and formatter built in Rust by Astral (the same company behind UV). A single Ruff invocation replaces a whole family of tools that previously ran as separate, independently-configured processes:

| Old tool | Ruff equivalent | Typical speedup |
|---|---|---|
| `flake8` | `ruff check` | ~100x |
| `pylint` | `ruff check --select PL` | ~300–1000x |
| `isort` | `ruff check --select I` | ~100x |
| `black` | `ruff format` | ~50–60x |
| `pyupgrade` | `ruff check --select UP` | ~50x |
| `autoflake` | `ruff check --select F` | ~50x |
| `bandit` | `ruff check --select S` | ~50x |
| `pydocstyle` | `ruff check --select D` | ~50x |

Measured on a ~250,000-line codebase:

| Tool | Run time |
|---|---|
| pylint | ~150 s |
| flake8 | ~45 s |
| black | ~30 s |
| isort | ~15 s |
| **Ruff (lint)** | **~0.4 s** |
| **Ruff (format)** | **~0.5 s** |

Beyond raw speed, the practical win for a large monorepo is **consolidation**: one binary, one configuration file format, one dependency to install and mirror internally, and one CI job type instead of four or five. This directly supports the migration goals from `01-introduction-and-roadmap.md` — fast, uniform, low-maintenance linting/formatting across dozens of interdependent packages.

This guide assumes **Ruff 0.9+** (the version referenced throughout this series), which includes the stable `ruff format` command in addition to `ruff check`.

## 2. Installing Ruff in the Workspace

Ruff is added as a UV dev/lint dependency, not installed globally, so every developer and CI job uses the exact pinned version from the lockfile.

```toml
# pyproject.toml (workspace root) or a package's pyproject.toml
[dependency-groups]
lint = [
    "ruff>=0.9.0",
]
```

```bash
# add to the workspace root
uv add --dev ruff

# or to a specific package's lint group
uv add --group lint ruff --package shared-utils
```

```bash
# verify
uv run ruff --version
uv run ruff check --version
```

In an air-gapped network, Ruff is installed the same way as any other dependency — via UV from the internal package index (see `03-uv-package-manager-migration.md` for private registry configuration). No separate binary download is required as long as Ruff is published to (or mirrored into) that index.

## 3. Ruff Configuration for a Monorepo

### 3.1 Root config file: `ruff.toml` vs. `pyproject.toml`

Ruff reads its configuration from either a standalone `ruff.toml` / `.ruff.toml` file or a `[tool.ruff]` table in `pyproject.toml`. In a UV workspace, every package already has its own `pyproject.toml` for project metadata — putting Ruff config there as well works, but a **standalone `ruff.toml` at the workspace root** is generally cleaner for a monorepo:

- It separates tool configuration from package metadata, so root `pyproject.toml` stays focused on the UV workspace definition.
- Per-package overrides become a small, explicit `ruff.toml` (via `extend`) instead of a large `[tool.ruff]` block duplicated across every `pyproject.toml`.
- It is one of the simplest files to point NX cache `inputs` at (see [Section 5](#5-ruff-as-an-nx-target)).

Either approach is valid; this guide uses `ruff.toml` for examples.

### 3.2 Full root `ruff.toml`

```toml
# ruff.toml (workspace root)

target-version = "py311"
line-length = 100
indent-width = 4
respect-gitignore = true

exclude = [
    ".git",
    ".venv",
    ".nx",
    ".uv-cache",
    "__pycache__",
    "*.egg-info",
    "dist",
    "build",
    ".tox",
    ".eggs",
    "node_modules",
    "migrations",        # Django/Alembic migrations — not worth linting
    "**/generated/**",    # generated code
    "**/_vendor/**",      # vendored third-party code
]

[lint]
select = [
    "E",      # pycodestyle errors
    "W",      # pycodestyle warnings
    "F",      # Pyflakes
    "I",      # isort
    "B",      # flake8-bugbear
    "C4",     # flake8-comprehensions
    "UP",     # pyupgrade
    "N",      # pep8-naming
    "S",      # flake8-bandit (security)
    "SIM",    # flake8-simplify
    "RUF",    # Ruff-specific rules
    "PERF",   # Perflint
    "PL",     # Pylint
]

ignore = [
    "E501",    # line too long — handled by the formatter, not the linter
    "ISC001",  # implicit string concat — conflicts with `ruff format`
    "COM812",  # missing trailing comma — conflicts with `ruff format`
    "PLR0913", # too many arguments — sometimes legitimately needed
    "PLR2004", # magic value in comparison — sometimes clearer inline
]

fixable = ["ALL"]
unfixable = [
    "F841",  # unused local variable — sometimes kept as documentation
]

[lint.per-file-ignores]
"**/__init__.py" = ["F401"]        # re-exports are expected here
"**/tests/**" = ["S101", "ANN", "PLR2004"]  # assert / annotations / magic values OK in tests
"**/migrations/**" = ["ALL"]
"tools/**" = ["T20"]                # print() is fine in internal tooling

[lint.isort]
known-first-party = ["shared_utils", "db_models", "auth_helpers"]
force-sort-within-sections = true
split-on-trailing-comma = true

[lint.pydocstyle]
convention = "google"

[lint.flake8-bugbear]
extend-immutable-calls = [
    "fastapi.Depends",
    "fastapi.Query",
    "fastapi.Body",
    "fastapi.Header",
    "fastapi.Cookie",
    "fastapi.Form",
    "fastapi.File",
]

[lint.mccabe]
max-complexity = 10

[format]
quote-style = "double"
indent-style = "space"
line-ending = "lf"
docstring-code-format = true
skip-magic-trailing-comma = false
```

`known-first-party` should list every internal package name (with underscores, matching the importable module name) so isort correctly groups internal imports separately from third-party ones — keep this list in sync as new workspace members are added.

### 3.3 Per-package overrides

Individual packages — most commonly legacy code being migrated gradually — can extend and selectively override the root config:

```toml
# packages/legacy-lib/ruff.toml
extend = "../../ruff.toml"   # inherit everything from the root

[lint]
extend-select = ["ANN"]      # opt this package into stricter type-annotation checks

ignore = [
    "N803",   # legacy argument naming conventions
]

[lint.per-file-ignores]
"src/legacy_lib/old_module.py" = ["E", "W", "C4"]  # ignore broad categories in one known-bad file
```

This hierarchy — one root config, thin per-package `extend` files — keeps the common case (most packages use the shared rules unmodified) simple while still allowing exceptions where the codebase genuinely needs them.

## 4. Migrating Off flake8 / black / isort

1. **Inventory existing configuration.**

   ```bash
   grep -r "flake8\|black\|isort\|pylint\|autopep8" pyproject.toml requirements*.txt
   cat .flake8 setup.cfg 2>/dev/null
   ```

2. **Translate the old config into `ruff.toml`.**

   ```ini
   # .flake8 (old)
   [flake8]
   max-line-length = 88
   extend-ignore = E203, E266, W503
   exclude = .git, __pycache__, migrations
   per-file-ignores =
       __init__.py: F401
       tests/*: S101
   ```

   ```toml
   # ruff.toml (new)
   line-length = 88
   exclude = [".git", "__pycache__", "migrations"]

   [lint]
   select = ["E", "W", "F"]
   ignore = ["E203", "E266", "W503"]

   [lint.per-file-ignores]
   "**/__init__.py" = ["F401"]
   "**/tests/**" = ["S101"]
   ```

3. **Run a one-time bulk fix and format, as a dedicated commit.**

   ```bash
   uv run ruff check . --fix --unsafe-fixes
   uv run ruff format .

   git add -A
   git commit -m "chore: apply Ruff formatting (initial migration)"
   ```

   > **This commit will cause merge conflicts in every open PR.** Announce the exact commit SHA to the team in advance and ask contributors to rebase or merge from `main` immediately after it lands. There is no way to avoid this churn — a repo-wide reformat is inherently disruptive; the only mitigation is coordination and timing (e.g., land it right before a quiet period, not mid-sprint).

4. **Remove the old tools.**

   ```bash
   uv remove flake8 black isort autopep8 pylint
   ```

   Delete the now-obsolete `[tool.black]`, `[tool.isort]` sections from `pyproject.toml` and any standalone `.flake8` / `setup.cfg` linter sections.

5. **For code that isn't ready to be fully compliant**, use scoped `noqa` comments as a temporary, incremental adoption path rather than disabling whole rule categories:

   ```python
   import legacy_module  # noqa: F401

   def legacy_function(self, x, y, z=None, **kwargs):  # noqa: ANN
       pass
   ```

   Track these and clean them up gradually rather than leaving them permanently.

## 5. Ruff as an NX Target

Full NX target syntax (`project.json`, `nx.json` `targetDefaults`, executors, caching semantics) is covered in `02-nx-monorepo-management.md`. This section only covers the Ruff-specific wiring.

Two approaches, depending on whether the `@nxlv/python` NX plugin is in use:

- **With `@nxlv/python`:** use its dedicated `@nxlv/python:ruff` executor rather than a generic `nx:run-commands` call. It activates the package's virtual environment before invoking Ruff and integrates with NX's caching correctly.

  ```json
  {
    "targets": {
      "lint": {
        "executor": "@nxlv/python:ruff",
        "options": {
          "lintFilePatterns": ["packages/shared-utils/src", "packages/shared-utils/tests"]
        },
        "cache": true,
        "inputs": ["default", "{workspaceRoot}/ruff.toml"]
      }
    }
  }
  ```

- **Without a Python-aware plugin:** define `lint` / `format` / `format-check` targets with `nx:run-commands`, invoking `uv run ruff ...` directly:

  ```json
  {
    "targets": {
      "lint": {
        "executor": "nx:run-commands",
        "options": {
          "command": "uv run ruff check {projectRoot}",
          "cwd": "{workspaceRoot}"
        },
        "cache": true,
        "inputs": ["default", "{workspaceRoot}/ruff.toml"]
      },
      "format-check": {
        "executor": "nx:run-commands",
        "options": {
          "command": "uv run ruff format --check {projectRoot}",
          "cwd": "{workspaceRoot}"
        },
        "cache": true,
        "inputs": ["default", "{workspaceRoot}/ruff.toml"]
      }
    }
  }
  ```

Regardless of executor, always include the root `ruff.toml` (and any per-package `ruff.toml`) in the target's cache `inputs` — otherwise a rule change won't invalidate cached lint results, and CI will keep reporting stale green checks.

With this in place, the normal workflow is affected-based, as established in `02-nx-monorepo-management.md`:

```bash
nx affected -t lint
nx affected -t format-check
nx run-many -t format
```

**Standalone, without NX**, the equivalent commands against a single directory are:

```bash
uv run ruff check packages/shared-utils
uv run ruff format --check packages/shared-utils
```

### GitLab Code Quality output

Independent of NX, Ruff can emit a report GitLab renders directly on the merge request:

```bash
uv run ruff check --output-format=gitlab --output-file=ruff-code-quality.json .
```

The corresponding `.gitlab-ci.yml` job configuration (stages, `artifacts: reports:`, rules) is covered in `06-gitlab-ci-integration-and-operations.md`.

## 6. The Custom Python Formatter

### 6.1 The original ask

> "A Python script that takes a directory as a command-line argument and applies formatting to all the code in it."

The script is a thin, opinionated wrapper around Ruff — not a replacement for it. Ruff already does the heavy lifting (import sorting, style normalization, safe autofixes); the custom script exists to:

- provide one consistent, memorable entry point (`format_code.py <dir>`) instead of remembering two or three separate Ruff invocations and flag combinations,
- sequence Ruff's sub-commands in a specific, deliberate order,
- layer in **house-specific rules that Ruff has no built-in check for** (e.g., license header presence, forbidden internal API usage, naming conventions specific to the organization), and
- expose a clean `--check` mode with CI-friendly exit codes, so the same script drives both local formatting and CI enforcement.

### 6.2 CLI design

```
format_code.py <directory> [--check] [--no-lint] [--no-format] [--unsafe-fixes] [--output-format {text,json,gitlab,github}] [-v]
```

| Argument | Purpose |
|---|---|
| `directory` (positional) | Directory to format, relative to the workspace root or absolute. |
| `--check` | Verify only — make no changes; exit non-zero if anything would change. This is the mode CI should always use. |
| `--no-lint` | Skip the lint-fix pass; format only. |
| `--no-format` | Skip the format pass; lint only. |
| `--unsafe-fixes` | Allow Ruff's unsafe autofixes (opt-in, not the CI default). |
| `--output-format` | Lint output format, forwarded to `ruff check` (useful for GitLab Code Quality integration). |
| `-v` / `--verbose` | Print each underlying command and its result. |

### 6.3 Directory walking

The script itself does **not** need to manually walk the file tree with `os.walk` / `pathlib.rglob` for the Ruff-covered checks — `ruff check` and `ruff format` already accept a directory argument and recurse through it themselves, honoring `exclude` and `.gitignore` from `ruff.toml`. Delegating the walk to Ruff avoids duplicating its exclude logic and keeps the script simple.

A manual walk (via `pathlib.Path.rglob("*.py")`) is only needed for the **house-specific checks that Ruff doesn't perform** — for example, verifying every source file starts with a license header. That logic lives in a separate, explicit pass in the script rather than being smuggled into the Ruff invocations.

### 6.4 Integration order with Ruff

The recommended sequence, and the reasoning behind it:

1. **`ruff check --select I,F401,UP --fix`** — a narrow, structural autofix pass limited to import sorting, unused imports, and syntax upgrades. These are the fixes most likely to change code shape (reordering imports, removing lines).
2. **`ruff format`** — normalizes whitespace, quotes, and line length. Running this *after* step 1 ensures formatting is applied to the final code shape, not re-broken by a later autofix.
3. **House-specific checks** — the organization's own rules that Ruff has no equivalent for (license headers, forbidden imports, custom naming conventions). These run last since they should validate the already-formatted result.
4. **`ruff check` (full, no `--fix`)** — a final, non-mutating pass over the complete rule set, to surface anything that couldn't be auto-fixed and needs a human. In `--check` mode, steps 1–2 also run without `--fix`/with `--check`/`--diff` instead of mutating.

This ordering is a sensible default; teams may reverse steps 1–2 (format first, then lint-fix) if their house rules or Ruff rule selection make that safer — the important part is that the order is deliberate and documented, not that one specific ordering is universally correct.

### 6.5 Idempotency

Running the script twice in a row against unchanged input must produce **no further changes** on the second run, and `--check` mode must exit `0` immediately after a successful non-check run. This falls out naturally as long as:

- Ruff's own passes are deterministic (they are, for a fixed config and version),
- the house-specific pass only *adds* things that are already compliant once present (e.g., it won't re-insert a header that's already there), and
- `--unsafe-fixes` is not enabled by default in CI, since unsafe fixes are more likely to interact with each other across multiple runs.

### 6.6 Exit codes for CI

| Code | Meaning |
|---|---|
| `0` | Success — files are already correctly formatted (`--check`), or all formatting was applied cleanly. |
| `1` | Formatting/lint issues found in `--check` mode, or unfixed lint errors remain. CI should fail the job. |
| `2` | Configuration or runtime error (bad path, directory outside the workspace, Ruff not found, etc.) — distinct from "found real issues," so tooling problems don't get silently treated as lint failures. |

## 7. Reference Implementation

```python
#!/usr/bin/env python3
"""
Custom Python Formatter
=======================
Applies Ruff formatting and lint fixes to a directory, plus any
organization-specific rules Ruff does not cover.

Usage:
    python tools/scripts/format_code.py <directory> [options]
    python tools/scripts/format_code.py packages/shared-utils
    python tools/scripts/format_code.py packages/shared-utils --check
    python tools/scripts/format_code.py . --unsafe-fixes

Exit codes:
    0 - Success: no changes needed (--check) or all formatting applied
    1 - Formatting/lint issues found (--check) or unfixed lint errors remain
    2 - Configuration or runtime error
"""

from __future__ import annotations

import argparse
import subprocess
import sys
from pathlib import Path

# Rules limited to import sorting, unused imports, and syntax upgrades —
# the narrow, structural fix pass that runs before formatting.
STRUCTURAL_FIX_RULES = "I,F401,UP"

# House rule: every first-party source file must start with this header.
LICENSE_HEADER = "# Copyright (c) Company. All rights reserved."


def parse_args() -> argparse.Namespace:
    """Parse CLI arguments."""
    parser = argparse.ArgumentParser(
        description="Apply Ruff formatting/linting plus house rules to a directory",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog=__doc__,
    )
    parser.add_argument("directory", type=Path, help="Directory to format")
    parser.add_argument(
        "--check",
        action="store_true",
        help="Check only, make no changes (exit 1 if changes are needed)",
    )
    parser.add_argument(
        "--no-lint", action="store_true", help="Skip the lint-fix pass; format only"
    )
    parser.add_argument(
        "--no-format", action="store_true", help="Skip the format pass; lint only"
    )
    parser.add_argument(
        "--unsafe-fixes", action="store_true", help="Allow Ruff's unsafe autofixes"
    )
    parser.add_argument(
        "--output-format",
        choices=["text", "json", "gitlab", "github"],
        default="text",
        help="Output format for lint results",
    )
    parser.add_argument("--verbose", "-v", action="store_true", help="Verbose output")
    return parser.parse_args()


def find_workspace_root() -> Path:
    """Find the workspace root by looking for uv.lock or nx.json."""
    current = Path.cwd()
    for parent in [current, *current.parents]:
        if (parent / "uv.lock").exists() or (parent / "nx.json").exists():
            return parent
    return current


def validate_directory(directory: Path, workspace_root: Path) -> Path:
    """Resolve the target directory and ensure it exists and is in-workspace."""
    resolved = directory.resolve() if directory.is_absolute() else (workspace_root / directory).resolve()

    if not resolved.exists():
        print(f"Error: directory '{resolved}' does not exist", file=sys.stderr)
        sys.exit(2)
    if not resolved.is_dir():
        print(f"Error: '{resolved}' is not a directory", file=sys.stderr)
        sys.exit(2)

    try:
        resolved.relative_to(workspace_root.resolve())
    except ValueError:
        print(
            f"Error: '{resolved}' is outside workspace root '{workspace_root}'",
            file=sys.stderr,
        )
        sys.exit(2)

    return resolved


def run(cmd: list[str], cwd: Path, verbose: bool) -> int:
    """Run a subprocess command, streaming output, and return its exit code."""
    if verbose:
        print(f"Running: {' '.join(cmd)}", file=sys.stderr)
    result = subprocess.run(cmd, cwd=cwd, text=True, check=False)
    return result.returncode


def run_structural_fix(target: Path, workspace_root: Path, check_only: bool, verbose: bool) -> int:
    """Step 1: narrow autofix for imports/unused-imports/pyupgrade."""
    rel = target.relative_to(workspace_root)
    cmd = ["uv", "run", "ruff", "check", str(rel), "--select", STRUCTURAL_FIX_RULES]
    cmd += ["--diff"] if check_only else ["--fix"]
    return run(cmd, workspace_root, verbose)


def run_format(target: Path, workspace_root: Path, check_only: bool, verbose: bool) -> int:
    """Step 2: Ruff format pass."""
    rel = target.relative_to(workspace_root)
    cmd = ["uv", "run", "ruff", "format", str(rel)]
    if check_only:
        cmd += ["--check", "--diff"]
    return run(cmd, workspace_root, verbose)


def run_house_rules(target: Path, check_only: bool, verbose: bool) -> int:
    """Step 3: house-specific checks Ruff has no rule for (e.g. license header)."""
    violations = []
    for py_file in target.rglob("*.py"):
        if "__pycache__" in py_file.parts:
            continue
        text = py_file.read_text(encoding="utf-8")
        if not text.startswith(LICENSE_HEADER):
            if check_only:
                violations.append(py_file)
            else:
                py_file.write_text(f"{LICENSE_HEADER}\n{text}", encoding="utf-8")
                if verbose:
                    print(f"Inserted license header: {py_file}")

    if violations:
        for f in violations:
            print(f"Missing license header: {f}", file=sys.stderr)
        return 1
    return 0


def run_final_lint(target: Path, workspace_root: Path, output_format: str, verbose: bool) -> int:
    """Step 4: full, non-mutating lint pass to surface anything left unfixed."""
    rel = target.relative_to(workspace_root)
    cmd = ["uv", "run", "ruff", "check", str(rel)]
    if output_format != "text":
        cmd += ["--output-format", output_format]
    return run(cmd, workspace_root, verbose)


def main() -> int:
    args = parse_args()
    workspace_root = find_workspace_root()
    target = validate_directory(args.directory, workspace_root)

    if args.verbose:
        print(f"Workspace root: {workspace_root}", file=sys.stderr)
        print(f"Target: {target}", file=sys.stderr)
        print(f"Mode: {'check' if args.check else 'fix'}", file=sys.stderr)

    codes: list[int] = []

    if not args.no_lint:
        codes.append(run_structural_fix(target, workspace_root, args.check, args.verbose))

    if not args.no_format:
        codes.append(run_format(target, workspace_root, args.check, args.verbose))

    codes.append(run_house_rules(target, args.check, args.verbose))

    if not args.no_lint:
        codes.append(run_final_lint(target, workspace_root, args.output_format, args.verbose))

    final_code = max(codes) if codes else 0

    if args.verbose:
        status = "SUCCESS" if final_code == 0 else "ISSUES FOUND"
        print(f"=== {status} (exit code: {final_code}) ===", file=sys.stderr)

    return final_code


if __name__ == "__main__":
    sys.exit(main())
```

Usage:

```bash
# format a package in place
uv run python tools/scripts/format_code.py packages/shared-utils

# CI-friendly check mode — fails the job if anything needs formatting
uv run python tools/scripts/format_code.py packages/shared-utils --check

# format the entire monorepo
uv run python tools/scripts/format_code.py . --verbose

# lint only, no formatting
uv run python tools/scripts/format_code.py packages/shared-utils --no-format
```

Wiring this into NX (target definitions, `nx affected -t format-check`, cache `inputs` including the script itself so a script change invalidates the cache) follows the same pattern shown in [Section 5](#5-ruff-as-an-nx-target) — see `02-nx-monorepo-management.md` for the full target/executor reference.

## 8. Known Pitfalls

| Pitfall | Symptom | Fix |
|---|---|---|
| **`ISC001` / `COM812` vs. the formatter** | `ruff check` and `ruff format` disagree / fight each other over implicit string concatenation and trailing commas | Add both to `ignore` in `[lint]` — they're formatter-owned concerns |
| **`E501` after adopting `ruff format`** | Line-length lint errors that the formatter should have already resolved | Ignore `E501` in `[lint]`; line length is the formatter's job, not the linter's |
| **B008 false positives with FastAPI/Pydantic** | `Depends()`, `Query()`, etc. flagged as "function call in default argument" | Add them to `[lint.flake8-bugbear] extend-immutable-calls` |
| **`UP007` (`Optional[X]` → `X \| None`) fires unexpectedly** | Rule assumes a newer Python than the codebase targets | Set `target-version` correctly in `ruff.toml` (or ignore the rule if a truly mixed-version codebase requires the older syntax) |
| **`D401` imperative-mood docstring rule** | Flags docstrings like `"""Returns the user."""` | Ignore `D401` if that phrasing convention is accepted, rather than rewriting every docstring |
| **Massive one-time diff / merge conflicts** | The initial `ruff format` + `ruff check --fix` commit touches nearly every file | Land it as one dedicated, isolated commit; announce it ahead of time; ask contributors to rebase open branches immediately after |
| **Thousands of violations on first run** | `ruff check . --select ALL --statistics` returns an overwhelming count | Normal for a first run. Start from a deliberately curated `select` list (not `ALL`), fix incrementally, and use `per-file-ignores` for legacy code rather than disabling rules globally |
| **Leftover config from flake8/black/isort** | Conflicting or dead `[tool.black]`, `[tool.isort]`, `.flake8` sections still present | Remove them explicitly as part of the migration — Ruff ignores them, but they confuse future readers and other tooling |
| **Per-package `ruff.toml` silently diverges from root** | A package's overrides drift far enough that CI passes locally but fails elsewhere, or vice versa | Keep per-package files thin (`extend` + a small, explicit diff); review them whenever the root config changes |
| **Stale cached lint/format results in CI** | A rule change in `ruff.toml` doesn't get picked up; NX/CI reports a stale pass | Ensure `ruff.toml` (root and per-package) is listed in the relevant target's cache `inputs` (see [Section 5](#5-ruff-as-an-nx-target)) |
| **Custom formatter script becomes non-idempotent** | A second run of `format_code.py` still reports changes needed | Usually caused by `--unsafe-fixes` interacting with a house rule, or a house-rule pass that isn't safe to re-apply (e.g., re-inserting a header without checking it's already there) |

## 9. Checklist

- [ ] Ruff added as a dev/lint dependency via UV (`uv add --dev ruff`), not a global install
- [ ] Root `ruff.toml` defined: `target-version`, `line-length`, `exclude`, `[lint].select`/`ignore`, `[format]`
- [ ] `known-first-party` in `[lint.isort]` lists every internal package
- [ ] `per-file-ignores` set for tests, migrations, and internal tooling directories
- [ ] Per-package `ruff.toml` overrides created only where genuinely needed, via `extend = "../../ruff.toml"`
- [ ] `ISC001` / `COM812` / `E501` ignored (formatter-owned concerns)
- [ ] Initial `ruff check --fix --unsafe-fixes` + `ruff format .` run and committed as one isolated, announced commit
- [ ] flake8 / black / isort / pylint removed from dependencies and their config sections deleted
- [ ] Ruff wired as an NX `lint` / `format` / `format-check` target, with `ruff.toml` in cache `inputs`
- [ ] `nx affected -t lint` and `nx affected -t format-check` validated against real changes
- [ ] Custom formatter script (`tools/scripts/format_code.py`) written, with `--check` mode returning the correct exit codes
- [ ] Custom formatter's Ruff-vs-house-rule ordering documented and deliberate, not accidental
- [ ] Custom formatter verified idempotent (running twice produces no further changes)
- [ ] Custom formatter wired as an NX target alongside the raw Ruff targets
- [ ] GitLab Code Quality report (`--output-format=gitlab`) validated to render correctly on an MR
- [ ] Team notified of the reformat commit and any temporary `noqa` usage tracked for cleanup
