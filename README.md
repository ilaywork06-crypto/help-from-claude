# Monorepo Modernization Guide Series

> **Audience:** Developers and DevOps engineers on a large Python + Vue.js monorepo, running GitLab CI/CD self-hosted inside a closed/air-gapped corporate network.
> **Scope:** Migrating from Poetry to a modern toolchain — NX, UV, Ruff, and a custom formatter — with smart, dependency-aware releases.

This repository is a single, consolidated guide series (replacing several earlier overlapping drafts) covering the full migration end to end. Read the files in order — each one assumes the previous ones as background and avoids repeating their content.

## Reading order

| # | File | Covers |
|---|------|--------|
| 1 | [`01-introduction-and-roadmap.md`](01-introduction-and-roadmap.md) | Why this migration is happening, current vs. target state, priority order (NX → UV → Ruff → Custom Formatter), and the phased roadmap |
| 2 | [`02-nx-monorepo-management.md`](02-nx-monorepo-management.md) | NX core concepts, adopting NX in the existing monorepo, `@nxlv/python`, `nx affected`, `project.json`/`nx.json`, caching, and NX-specific pitfalls |
| 3 | [`03-uv-package-manager-migration.md`](03-uv-package-manager-migration.md) | Migrating from Poetry to UV, UV workspaces, multi-version dependency handling, and air-gapped registry/cache configuration for UV |
| 4 | [`04-ruff-and-custom-formatter.md`](04-ruff-and-custom-formatter.md) | Ruff configuration and rollout, plus the spec and reference implementation for the in-house custom Python formatter |
| 5 | [`05-nx-release-and-reverse-dependencies.md`](05-nx-release-and-reverse-dependencies.md) | Selective/smart releases with `nx release`, versioning strategy, and — the key new addition — a strategy and scripts for **releasing reverse dependencies** so downstream consumers of a changed library actually get updated |
| 6 | [`06-gitlab-ci-integration-and-operations.md`](06-gitlab-ci-integration-and-operations.md) | Wiring NX + UV + Ruff into GitLab CI/CD (pipelines, dynamic child pipelines, Docker images, Package Registry), air-gapped operations, cross-cutting pitfalls, and the master implementation checklist |

## Provenance

[`prompt.txt`](prompt.txt) contains the original (Hebrew) request that this guide series was written to answer. The six files above are the current, English, non-duplicative source of truth — earlier Hebrew drafts have been retired in favor of this set.
