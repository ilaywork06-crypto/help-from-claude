# Introduction and Migration Roadmap

> **Audience:** Developers, tech leads, and DevOps/platform engineers working in a large Python + Vue.js monorepo on self-hosted GitLab CI/CD inside a closed (air-gapped) corporate network.
> **Scope:** Why this migration is happening, the target end-state, the priority order for adopting NX, UV, Ruff, and the custom formatter, and a phased roadmap pointing to the rest of this guide series.

---

## Table of Contents

1. [Why We Are Migrating](#1-why-we-are-migrating)
2. [Current State](#2-current-state)
3. [Target End-State Architecture](#3-target-end-state-architecture)
4. [Migration Priority Order](#4-migration-priority-order)
5. [Phased Roadmap](#5-phased-roadmap)
6. [How to Use This Guide Series](#6-how-to-use-this-guide-series)

---

## 1. Why We Are Migrating

Our monorepo has grown to a size and complexity where the existing toolchain no longer scales. The pain points driving this migration:

| Pain point | Current behavior | Impact |
|---|---|---|
| No affected-based builds | Every pipeline runs lint/test/build across the *entire* repo, regardless of what changed | CI takes hours even for a one-line change; wasted compute on shared runners |
| No dependency visibility | Nothing maps which packages depend on which | Unclear blast radius when changing a shared library; regressions surface late |
| Poetry package management | `poetry install` is slow, lockfiles are not truly cross-platform, and workspace/monorepo support is limited | Slow CI setup steps, awkward handling of many interdependent packages and multiple Python versions |
| Fragmented lint/format tooling | flake8 + black + isort + assorted plugins, each with its own config and runtime | Slow feedback loops, inconsistent enforcement, config sprawl |
| Manual, error-prone releases | Versioning and publishing of individual packages is done by hand or with ad hoc scripts | Inconsistent version bumps, missed changelogs, human error |
| Air-gapped network | No direct access to PyPI, npm, Docker Hub, or GitHub | Every tool must be mirrored or vendored internally before it can be adopted |

None of these problems are isolated — they compound in a large, multi-package, multi-version monorepo where a single change can ripple across dozens of libraries and applications.

## 2. Current State

- **Source control / CI:** GitLab Self-Hosted, using standard (non-dynamic) `.gitlab-ci.yml` pipelines.
- **Repository shape:** a single monorepo containing many Python libraries and services plus one or more Vue.js applications, with varying dependency versions across packages.
- **Package management:** Poetry, used per-package with no shared workspace concept.
- **Linting/formatting:** a mix of flake8, black, isort, and similar tools, configured inconsistently across packages.
- **Network:** closed / air-gapped corporate network — no direct internet access from CI runners or developer machines to public registries (PyPI, npm, GitHub Releases, Docker Hub). Everything must go through an internal mirror (GitLab Package Registry, Artifactory/Nexus, or similar) or be vendored in manually.
- **Releases:** manual or script-driven, without a consistent versioning or changelog strategy tied to the packages that actually changed.

## 3. Target End-State Architecture

At a high level, the target toolchain adds an orchestration layer (NX) on top of a modern, fast package manager (UV) and a unified lint/format tool (Ruff), with a thin custom formatter script bridging organization-specific conventions. NX does not replace UV, Ruff, or pytest — it wraps them, adding a dependency graph, affected-based task selection, caching, and release orchestration across the whole monorepo.

```
┌───────────────────────────────────────────────────────────────┐
│                          MONOREPO                              │
│                                                                 │
│   NX — project graph, affected detection, caching, release     │
│         (orchestrates everything below)                        │
│                                                                 │
│   ┌───────────────────────┐     ┌───────────────────────────┐  │
│   │  UV                   │     │  Ruff                     │  │
│   │  Python package mgmt  │     │  Lint + format             │  │
│   │  & workspaces         │     │  (replaces flake8/black/   │  │
│   │  (replaces Poetry)    │     │   isort)                   │  │
│   └───────────────────────┘     └───────────────────────────┘  │
│                                                                 │
│   Custom Formatter — thin wrapper enforcing org conventions    │
│                                                                 │
│   GitLab CI/CD — runs NX affected pipelines, publishes to the  │
│   internal GitLab Package Registry, all within the closed      │
│   network                                                       │
└───────────────────────────────────────────────────────────────┘
```

The rest of this series covers each layer in depth.

## 4. Migration Priority Order

The team has agreed on this order — **NX → UV → Ruff → Custom Formatter** — for the following reasons:

1. **NX first.** NX gives us the project graph, affected-project detection, and caching *before* anything else changes. This means we get immediate visibility into cross-package dependencies and CI time savings while the underlying package manager and lint tools are still the old ones. It also de-risks the rest of the migration: once NX can identify what's affected by a change, every subsequent step (switching package managers, switching linters) can be validated package-by-package instead of repo-wide.
2. **UV second.** Once NX is in place and can run tasks per-project, swapping the package manager underneath (Poetry → UV) is a contained, per-package change. NX's affected detection lets us migrate and validate one package at a time without re-running the whole suite, and its caching means repeated `uv sync` runs during the transition stay cheap. Doing UV before Ruff also matters because the tooling *install* flow changes (dependency groups, workspace members, lockfile) — it's cleaner to stabilize how packages install before changing how their code is linted.
3. **Ruff third.** Ruff depends on a stable package management story (dependency groups for dev/lint tools, a working `uv run`) and benefits from NX affected/caching already being in place, so lint runs are fast and scoped. Migrating linting after packaging avoids conflating two large sets of changes at once.
4. **Custom formatter last.** The custom formatter script is a thin wrapper around Ruff (`ruff format` + `ruff check --fix`) enforcing organization-specific conventions. It only makes sense once Ruff itself is fully adopted and configured — it has nothing to wrap otherwise.

This order minimizes the size and risk of any single change, keeps each step independently verifiable via NX affected, and avoids reworking the same files twice.

## 5. Phased Roadmap

| Phase | Focus | Details in |
|---|---|---|
| 1 | Introduce NX: project graph, `project.json`/`nx.json`, affected detection, local caching | [`02-nx-monorepo-management.md`](02-nx-monorepo-management.md) |
| 2 | Migrate Python package management from Poetry to UV: workspace setup, per-package `pyproject.toml` conversion, lockfile, private registry access | [`03-uv-package-manager-migration.md`](03-uv-package-manager-migration.md) |
| 3 | Adopt Ruff for linting and formatting, then layer in the custom formatter script; retire flake8/black/isort | [`04-ruff-and-custom-formatter.md`](04-ruff-and-custom-formatter.md) |
| 4 | Wire up NX Release for versioning/changelogs/publishing, and make sure reverse-dependency propagation (affected graph) behaves correctly across Python and Vue.js projects | [`05-nx-release-and-reverse-dependencies.md`](05-nx-release-and-reverse-dependencies.md) |
| 5 | Full GitLab CI/CD integration: pipeline structure, caching strategy, remote/self-hosted cache, air-gapped considerations, day-2 operations | [`06-gitlab-ci-integration-and-operations.md`](06-gitlab-ci-integration-and-operations.md) |

Suggested rollout checklist at a glance:

- [ ] NX installed, project graph builds correctly for all Python and Vue.js projects
- [ ] `nx affected` validated against real historical changes (spot-check a few past commits)
- [ ] UV workspace replaces Poetry across all packages; CI installs via `uv sync`
- [ ] Ruff replaces flake8/black/isort; CI lint/format jobs pass on the whole repo
- [ ] Custom formatter integrated as an NX target
- [ ] NX Release configured and dry-run validated for at least one package
- [ ] GitLab CI pipelines fully switched to affected-based execution with working cache

## 6. How to Use This Guide Series

This series is split into six files, each owned independently. Read them in order on a first pass; use them as standalone references afterward.

1. **`01-introduction-and-roadmap.md`** (this file) — why we're migrating, current vs. target state, priority order, and the phased roadmap.
2. **`02-nx-monorepo-management.md`** — installing and configuring NX, the project graph, `nx.json`/`project.json`, affected detection, and caching fundamentals.
3. **`03-uv-package-manager-migration.md`** — moving from Poetry to UV: workspace setup, converting `pyproject.toml` files, lockfiles, private registry configuration, and common migration pitfalls.
4. **`04-ruff-and-custom-formatter.md`** — Ruff configuration for linting and formatting, migrating off flake8/black/isort, and integrating the custom Python formatter script.
5. **`05-nx-release-and-reverse-dependencies.md`** — NX Release for versioning/changelogs/publishing, and ensuring the affected/reverse-dependency graph behaves correctly across Python and Vue.js projects.
6. **`06-gitlab-ci-integration-and-operations.md`** — full GitLab CI/CD pipeline design, caching strategy (including self-hosted remote cache for the air-gapped network), and ongoing operational concerns.
