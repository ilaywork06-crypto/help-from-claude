# NX Release and Reverse Dependencies

> **Audience:** Developers, tech leads, and DevOps/platform engineers responsible for versioning and publishing packages out of a large Python + Vue.js monorepo on self-hosted GitLab CI/CD inside a closed (air-gapped) corporate network.
> **Scope:** How `nx release` versions, changelogs, and publishes projects selectively (only what actually changed), how that maps onto Python (`@nxlv/python`) and Vue/JS (`@jscutlery/semver`) packages, and — the focus of this file — how to correctly identify and release the *reverse dependencies* of a changed library so that downstream consumers are not left silently pinned to a stale, already-fixed version.

---

## Table of Contents

1. [NX Release Fundamentals](#1-nx-release-fundamentals)
2. [Versioning Strategies: Independent vs. Fixed](#2-versioning-strategies-independent-vs-fixed)
3. [Conventional Commits and Automatic Version Bumps](#3-conventional-commits-and-automatic-version-bumps)
4. [Configuring `nx.json` for Release](#4-configuring-nxjson-for-release)
5. [Release Groups](#5-release-groups)
6. [Combining NX Affected with Release Groups](#6-combining-nx-affected-with-release-groups)
7. [Releasing Python Packages with @nxlv/python](#7-releasing-python-packages-with-nxlvpython)
8. [Releasing Vue/JS Packages and GitLab Release Objects](#8-releasing-vuejs-packages-and-gitlab-release-objects)
9. [Releasing Reverse Dependencies](#9-releasing-reverse-dependencies)
   - [9.1 The Problem: Stale Transitive Dependencies](#91-the-problem-stale-transitive-dependencies)
   - [9.2 Computing the Reverse Dependency Set](#92-computing-the-reverse-dependency-set)
   - [9.3 Release Propagation Policy](#93-release-propagation-policy)
   - [9.4 Scripting Reverse-Dependency Releases in CI](#94-scripting-reverse-dependency-releases-in-ci)
   - [9.5 Guardrails Against Release Cascades](#95-guardrails-against-release-cascades)
10. [Implementation Checklist](#10-implementation-checklist)

---

## 1. NX Release Fundamentals

`nx release` is NX's built-in release orchestration engine. For a monorepo this large, it replaces ad hoc scripts and manual version bumps with three coordinated steps:

```
1. nx release version     → compute and write new version numbers (pyproject.toml / package.json)
2. nx release changelog   → generate CHANGELOG.md entries from commit history, tag, commit, push
3. nx release publish     → build + publish to the registry (PyPI-compatible or npm-compatible)
```

These can be run individually, or together:

```bash
# Always dry-run first — this is not optional in a monorepo of this size
nx release --dry-run

# Full release: version + changelog + git tag/commit/push + publish
nx release

# First release of a project that has never been tagged before
nx release --first-release --dry-run
nx release --first-release
```

NX itself does not know how to read or write `pyproject.toml` — that requires `@nxlv/python` (see [Section 7](#7-releasing-python-packages-with-nxlvpython)). It also does not create GitLab Release objects natively — that requires either `@jscutlery/semver:gitlab` (JS/TS projects, see [Section 8](#8-releasing-vuejs-packages-and-gitlab-release-objects)) or a manual `curl` call to the GitLab Releases API.

## 2. Versioning Strategies: Independent vs. Fixed

`nx.json`'s `release.projectsRelationship` controls how versions relate to each other across the workspace:

| Strategy | Behavior | When to use |
|---|---|---|
| `"independent"` | Every project gets its own version, bumped only when *it* changes (e.g. `shared-utils@1.5.0`, `auth-helpers@3.2.1`, `data-pipeline@0.8.0` can all differ) | **Recommended default for this monorepo.** Dozens of independently-consumed libraries with different release cadences; a change to one library should not force a version bump — or a publish — of every other library. |
| `"fixed"` (a.k.a. locked-step) | All projects in the release group share one version number, bumped together on every release | Small, tightly-coupled groups of packages that are always installed together (e.g. a UI kit split into `@myorg/ui-core` + `@myorg/ui-icons` that must always match) |

```json
{
  "release": {
    "projects": ["libs/python/*", "libs/frontend/*"],
    "projectsRelationship": "independent"
  }
}
```

Mixing strategies is normal and expected: use `"independent"` at the top level for most Python and Vue libraries, and carve out a `"fixed"` [release group](#5-release-groups) for the handful of packages that truly need to move in lockstep.

## 3. Conventional Commits and Automatic Version Bumps

`nx release` reads Git history and computes the bump automatically when Conventional Commits are used:

| Commit message | Version bump |
|---|---|
| `fix(shared-utils): handle None in date parser` | Patch (`1.0.0` → `1.0.1`) |
| `feat(shared-utils): add pagination helper` | Minor (`1.0.0` → `1.1.0`) |
| `feat!: remove deprecated auth flow` or a `BREAKING CHANGE:` footer | Major (`1.0.0` → `2.0.0`) |
| `chore: update dependencies` / `docs: update README` | No bump |

Critical detail that is easy to miss: **the commit `scope` is what tells NX which project the bump belongs to.** `feat(shared-utils): ...` bumps only `shared-utils`. A commit with no scope, or a scope that doesn't match a project name, will be treated as touching *every* affected project in that invocation — which is rarely what you want. Enforce scoped commit messages with a commit-msg hook:

```yaml
# .pre-commit-config.yaml
- repo: https://github.com/compilerla/conventional-pre-commit
  rev: v3.3.0
  hooks:
    - id: conventional-pre-commit
      stages: [commit-msg]
      args: [feat, fix, chore, docs, refactor, test, ci, perf]
```

## 4. Configuring `nx.json` for Release

A representative baseline configuration for the monorepo, combining independent versioning, per-project changelogs, and a predictable tag pattern:

```json
{
  "release": {
    "projects": ["libs/python/*", "libs/frontend/*", "apps/*"],
    "projectsRelationship": "independent",
    "releaseTagPattern": "{projectName}@{version}",
    "changelog": {
      "automaticFromRef": true,
      "projectChangelogs": {
        "file": "{projectRoot}/CHANGELOG.md",
        "createRelease": "gitlab"
      },
      "workspaceChangelog": {
        "createRelease": false
      }
    },
    "version": {
      "conventionalCommits": true,
      "generatorOptions": {
        "currentVersionResolver": "git-tag"
      },
      "updateDependents": "always"
    },
    "git": {
      "commit": true,
      "commitMessage": "chore(release): {projectName} v{version}",
      "tag": true,
      "tagMessage": "{projectName}@{version}",
      "push": true
    }
  }
}
```

`version.updateDependents: "always"` is the single most relevant setting for this file's later sections: it tells NX to also touch the manifests of projects that declare a workspace dependency on whatever just got a new version, rewriting the declared constraint to point at the new version. It does **not**, by itself, decide whether those dependents should be *rebuilt and republished* — that policy question is what [Section 9](#9-releasing-reverse-dependencies) is about.

## 5. Release Groups

Release groups let different parts of the workspace use entirely different release configuration — different relationship, different tag pattern, different version resolver — under one `nx.json`:

```json
{
  "release": {
    "groups": {
      "python-libs": {
        "projects": ["libs/python/*"],
        "projectsRelationship": "independent",
        "version": {
          "conventionalCommits": true,
          "generatorOptions": {
            "currentVersionResolver": "git-tag"
          }
        },
        "changelog": {
          "projectChangelogs": true
        }
      },
      "vue-ui-kit": {
        "projects": ["libs/frontend/ui-components", "libs/frontend/design-tokens"],
        "projectsRelationship": "fixed",
        "releaseTag": {
          "pattern": "ui-v{version}"
        }
      }
    }
  }
}
```

Groups are also the natural boundary for [blast-radius limiting](#95-guardrails-against-release-cascades) later — a reverse-dependency cascade should rarely be expected to cross group boundaries.

## 6. Combining NX Affected with Release Groups

Never run a bare `nx release` in production — always scope it to a specific project list, tag filter, or group, computed from what actually changed:

```bash
# Release one library
nx release --projects=shared-utils --dry-run
nx release --projects=shared-utils --git-commit --git-push --git-tag

# Release several libraries
nx release --projects=shared-utils,db-models

# Release everything tagged as a library
nx release --projects=tag:type:lib

# Release a whole group
nx release --groups=python-libs
```

The standard selective-release pattern is to intersect `nx affected` with a "library" tag filter, so only libraries that changed since the last release are considered at all:

```bash
AFFECTED_LIBS=$(npx nx show projects --affected \
  --base=$NX_BASE --head=$NX_HEAD \
  --filter-tag=type:lib 2>/dev/null | tr '\n' ',')

if [ -z "$AFFECTED_LIBS" ]; then
  echo "No libraries affected. Skipping release."
  exit 0
fi

npx nx release --projects=$AFFECTED_LIBS --dry-run
```

This is the mechanism that keeps releases *selective*: instead of bumping and publishing the entire monorepo on every merge to `main`, only the libraries whose source actually changed are considered for a version bump — and, as covered next, their reverse dependencies.

## 7. Releasing Python Packages with @nxlv/python

NX's release engine has no built-in knowledge of `pyproject.toml`. For Python projects, plug in `@nxlv/python`'s version actions so the version write step knows how to edit the file correctly:

```json
{
  "release": {
    "version": {
      "conventionalCommits": true,
      "versionActions": "@nxlv/python/release/version-actions",
      "generatorOptions": {
        "fallbackCurrentVersionResolver": "registry"
      }
    }
  }
}
```

The end-to-end flow for a Python project:

```
NX Release Engine
  → reads commit history (Conventional Commits)
  → computes bump (major/minor/patch)
  → delegates to @nxlv/python/release/version-actions
      → updates version in pyproject.toml
      → updates uv.lock
      → writes CHANGELOG.md
  → NX Release Engine resumes: git commit, git tag, git push
```

Publishing uses the dedicated executor, wired to `build` so a fresh wheel/sdist is produced first:

```json
{
  "targets": {
    "publish": {
      "executor": "@nxlv/python:publish",
      "options": {
        "buildTarget": "shared-utils:build",
        "publish": true,
        "registry": "https://gitlab.company.internal/api/v4/projects/{PROJECT_ID}/packages/pypi"
      },
      "dependsOn": ["build"]
    }
  }
}
```

**`@jscutlery/semver` is not the tool for Python** — it only understands `package.json`. Don't reach for it here; it's covered next for the JS/Vue side.

## 8. Releasing Vue/JS Packages and GitLab Release Objects

For Vue/JS projects, `@jscutlery/semver` provides Conventional-Commits-driven versioning of `package.json`, plus a `:gitlab` executor that creates an actual GitLab Release object (not just a Git tag):

```bash
npm install -D @jscutlery/semver
npx nx generate @jscutlery/semver:install
```

```json
{
  "targets": {
    "version": {
      "executor": "@jscutlery/semver:version",
      "options": {
        "preset": "conventional-commits",
        "tagPrefix": "{projectName}@",
        "commitMessageFormat": "chore({projectName}): release version {version}",
        "trackDeps": true,
        "postTargets": ["web-app:build", "web-app:publish", "web-app:gitlab-release"]
      }
    },
    "gitlab-release": {
      "executor": "@jscutlery/semver:gitlab",
      "options": {
        "tag": "{tag}",
        "notes": "{notes}"
      }
    }
  }
}
```

`trackDeps: true` is `@jscutlery/semver`'s own (JS/TS-only) mechanism for patch-bumping a project automatically when one of its workspace dependencies gets a new version — conceptually the same propagation problem this file addresses generally in [Section 9](#9-releasing-reverse-dependencies), just scoped to npm-style `package.json` dependents. The `:gitlab` executor requires the GitLab Release CLI (`glab`) to be available on the runner.

---

## 9. Releasing Reverse Dependencies

Everything above answers "how do I release the library that changed?" This section answers the harder question: **when library A changes, which *other* libraries also need a new release so that their consumers actually get the fix?**

### 9.1 The Problem: Stale Transitive Dependencies

In a workspace this large, most libraries are not leaves — they sit in the middle of a dependency chain. Consider:

```
shared-utils  →  auth-helpers  →  api-service
     ↑                                 ↑
     └─────────── db-models ───────────┘
```

Suppose `shared-utils` ships `fix(shared-utils): correct timezone handling in date parser`, and gets released as `shared-utils@1.5.1`. Two things are true simultaneously:

1. **`nx affected` already knows** that `auth-helpers`, `db-models`, and `api-service` are affected by this change, for the purposes of *building and testing* them (this is the same forward-through-the-graph walk `nx affected` always does — see the NX fundamentals file for how the algorithm works).
2. **`nx release`'s version bump does not automatically follow.** Unless `updateDependents: "always"` is set, `auth-helpers`'s manifest still declares `shared-utils>=1.4.0`, its own `CHANGELOG.md` says nothing happened, and — critically — `auth-helpers` itself was never rebuilt or republished. Anyone who does a fresh `uv sync` or `pip install auth-helpers` gets whatever `shared-utils` version `auth-helpers`'s *last published* lockfile/constraint resolves to. If that constraint is loose (`>=1.4.0`) they might transitively pick up `1.5.1` — but `auth-helpers` was never tested against it, so if `shared-utils`'s fix required any adaptation in `auth-helpers`, that adaptation was never shipped. If the constraint is pinned or `auth-helpers` bundles a locked dependency snapshot (`lockedVersions: true`, `bundleLocalDependencies: true` in `@nxlv/python:build`), consumers of `auth-helpers` stay on the old, buggy `shared-utils` indefinitely, even though the fix has technically "been released."

This is the reverse-dependency release problem: **a fix at the bottom of the graph only reaches real consumers if every project between the fix and the consumer also gets a coordinated new release.** `updateDependents: "always"` handles the narrow case of *rewriting the version constraint string*; it does not decide *which* dependents are worth actually rebuilding and republishing, at what bump size, or where the cascade should stop — that requires an explicit policy, which is what the rest of this section defines.

### 9.2 Computing the Reverse Dependency Set

The NX project graph is directed from dependent → dependency (`api-service` has an edge pointing at `auth-helpers`). To find everything that needs to react to a change in `shared-utils`, you need to walk that graph **in reverse**: starting at the changed project, follow every edge backwards to find direct dependents, then repeat from each of those to find transitive dependents, until you reach a fixed point.

Two building blocks make this possible:

**1. `nx show projects --affected`** already gives you the forward-computed affected set for a given Git range — this is the same reverse walk, but it's driven by the diff, and its purpose is scoping build/test, not deciding what to release:

```bash
npx nx show projects --affected --base=$NX_BASE --head=$NX_HEAD
```

**2. `nx graph --file=<path>`** dumps the full project graph as JSON, which lets you build your own reverse index and walk it deliberately — independent of any particular Git diff, with your own filtering and depth rules:

```bash
npx nx graph --file=graph.json
```

```json
// graph.json (abridged)
{
  "graph": {
    "dependencies": {
      "auth-helpers": [
        { "source": "auth-helpers", "target": "shared-utils", "type": "static" }
      ],
      "api-service": [
        { "source": "api-service", "target": "auth-helpers", "type": "static" },
        { "source": "api-service", "target": "db-models", "type": "static" }
      ],
      "db-models": [
        { "source": "db-models", "target": "shared-utils", "type": "static" }
      ]
    }
  }
}
```

Inverting this (`target → [source, source, ...]`) gives the reverse-dependency index: `shared-utils → [auth-helpers, db-models]`, `auth-helpers → [api-service]`, `db-models → [api-service]`. Walking that index transitively from `shared-utils` yields the full set `{auth-helpers, db-models, api-service}` — exactly the projects that should be *considered* for a coordinated release, not just a rebuild.

### 9.3 Release Propagation Policy

Computing the reverse-dependency set is mechanical; deciding what to do with each project in it is a policy question. The recommended default policy:

| Situation | Action |
|---|---|
| Direct dependent, no conventional commits of its own since last release, dependency's change was patch/non-breaking | Propagate a **patch** bump — new version, changelog entry noting the upstream bump, rebuild, republish |
| Direct dependent that also has its own `feat`/`fix` commits since last release | Use **whichever bump is larger** — the dependent's own commit-driven bump or the propagated patch. Never stack them (a project doesn't get `+1 patch` on top of its own minor bump; take `max()`) |
| Dependency's change was major/breaking (`feat!:` / `BREAKING CHANGE:`) | Propagate at minimum a patch to record the pinned-version bump, but flag the dependent for **manual review** — a breaking change upstream often requires actual code changes downstream, which should show up as the dependent's own `fix`/`feat!` commit, not an automatic bump |
| Transitive dependent (2+ hops away) | Same policy, cascaded outward — but only if the intermediate project genuinely re-exports or is affected by the change (see guardrails for when to stop) |
| Project is internal-only, never published outside the workspace (workspace-protocol resolution only, no external consumers ever read its version) | **Do not force a release.** A cosmetic version bump with no consumer is noise. Let it bump normally on its own next real change |
| Dependency's change was purely additive (new optional export, no behavior change to anything the dependent uses) | **Do not force a release.** Non-breaking, non-consumed additions don't need forced propagation — a project shouldn't be republished just because something it doesn't use got slightly bigger |
| Change only touched tests, docs, or CI config (not shipped in the published artifact) | **Do not propagate at all** — nothing in the published package actually changed |
| Dependent is a deployable application, not a published library | "Release" here usually means *trigger a redeploy pipeline*, not a semver bump/publish — treat separately from library propagation |

The guiding principle: **propagate by default when the change is behavior-relevant and the dependent is externally consumed; skip when the dependent is either not really "released" (internal-only) or not actually affected in a way its own consumers would notice.**

### 9.4 Scripting Reverse-Dependency Releases in CI

The following script sketch computes the full release set — changed projects plus the dependents that need to be re-released — and applies the filtering from Section 9.3, before handing the result to `nx release`. It's deliberately built on the `nx` CLI's JSON-friendly commands (the same ones used elsewhere in this guide series) rather than NX's internal APIs, so it stays stable across NX versions.

```javascript
#!/usr/bin/env node
/**
 * tools/scripts/compute-release-set.js
 *
 * Computes the full set of projects to release for a given change window:
 *   1. Projects that changed directly (via `nx show projects --affected`).
 *   2. Every reverse dependency (dependent) of those projects, walked
 *      transitively across the NX project graph.
 *   3. Filtered down to what's actually releasable (publishable libraries,
 *      not internal-only helpers or plain apps).
 *
 * Usage:
 *   node tools/scripts/compute-release-set.js --base=<last-release-tag> --head=HEAD
 *
 * Output: release-set.json => { changedOnly: string[], propagatedOnly: string[] }
 */

const { execSync } = require('child_process');
const fs = require('fs');

const args = Object.fromEntries(
  process.argv.slice(2).map((a) => {
    const [k, v] = a.replace(/^--/, '').split('=');
    return [k, v ?? 'true'];
  })
);

const BASE = args.base || process.env.NX_BASE || 'HEAD~1';
const HEAD = args.head || process.env.NX_HEAD || 'HEAD';
const MAX_CASCADE_DEPTH = Number(args.maxDepth || 3);

function sh(cmd) {
  return execSync(cmd, { encoding: 'utf8' }).trim();
}

// 1. Projects that actually changed since the last release
function getChangedProjects() {
  const out = sh(
    `npx nx show projects --affected --base=${BASE} --head=${HEAD} --json`
  );
  return out ? JSON.parse(out) : [];
}

// 2. Full project graph, source -> [{ target, type }]
function getProjectDependencies() {
  const file = '.tmp-release-graph.json';
  sh(`npx nx graph --file=${file}`);
  const raw = JSON.parse(fs.readFileSync(file, 'utf8'));
  fs.unlinkSync(file);
  return raw.graph.dependencies;
}

// 3. Invert it: target -> [dependents]
function buildReverseIndex(dependencies) {
  const reverse = {};
  for (const edges of Object.values(dependencies)) {
    for (const edge of edges) {
      reverse[edge.target] ??= [];
      reverse[edge.target].push(edge.source);
    }
  }
  return reverse;
}

function getProjectMeta(name) {
  const info = JSON.parse(sh(`npx nx show project ${name} --json`));
  return { tags: info.tags || [], projectType: info.projectType };
}

// 4. Walk the reverse graph transitively, with cycle + depth guards
function computeReverseDependents(changed, reverseIndex) {
  const releaseSet = new Map(); // project -> { reason, depth }
  const pathStack = new Set();  // for cycle detection on the current DFS path

  function walk(project, depth, reason) {
    if (depth > MAX_CASCADE_DEPTH) {
      console.warn(
        `[cascade] depth limit (${MAX_CASCADE_DEPTH}) reached at "${project}" — ` +
        `stopping propagation here. If this keeps happening, "${project}" (or an ` +
        `ancestor) is probably too central and should be reviewed architecturally.`
      );
      return;
    }
    if (pathStack.has(project)) {
      throw new Error(
        `Cycle detected in reverse-dependency walk at "${project}". ` +
        `Fix the project graph (or split the cycle) before releasing.`
      );
    }
    pathStack.add(project);

    const existing = releaseSet.get(project);
    if (!existing || existing.depth > depth) {
      releaseSet.set(project, { reason, depth });
    }

    for (const dependent of reverseIndex[project] || []) {
      walk(dependent, depth + 1, `dependency-of:${project}`);
    }

    pathStack.delete(project);
  }

  for (const project of changed) {
    releaseSet.set(project, { reason: 'directly-changed', depth: 0 });
    for (const dependent of reverseIndex[project] || []) {
      walk(dependent, 1, `dependency-of:${project}`);
    }
  }

  return releaseSet;
}

// 5. Apply release policy: drop internal-only libs and plain (non-publishable) apps
function filterReleasable(releaseSet) {
  const result = new Map();
  for (const [project, info] of releaseSet) {
    const meta = getProjectMeta(project);
    const isInternalOnly = meta.tags.includes('release:internal-only');
    const isNonPublishableApp =
      meta.projectType === 'application' && !meta.tags.includes('release:publishable');

    if (isInternalOnly || isNonPublishableApp) {
      console.log(`[skip] ${project} (${info.reason}) — not independently releasable`);
      continue;
    }
    result.set(project, info);
  }
  return result;
}

function main() {
  const changed = getChangedProjects();
  if (changed.length === 0) {
    console.log('No projects changed since last release. Nothing to do.');
    return;
  }

  const dependencies = getProjectDependencies();
  const reverseIndex = buildReverseIndex(dependencies);
  const fullSet = computeReverseDependents(changed, reverseIndex);
  const releasable = filterReleasable(fullSet);

  console.log('\nComputed release set:');
  for (const [project, info] of releasable) {
    console.log(`  ${project.padEnd(30)} reason=${info.reason} depth=${info.depth}`);
  }

  const changedOnly = [...releasable.keys()].filter(
    (p) => releasable.get(p).reason === 'directly-changed'
  );
  const propagatedOnly = [...releasable.keys()].filter(
    (p) => releasable.get(p).reason !== 'directly-changed'
  );

  fs.writeFileSync(
    'release-set.json',
    JSON.stringify({ changedOnly, propagatedOnly }, null, 2)
  );
  console.log(`\nWrote release-set.json (${changedOnly.length} changed, ${propagatedOnly.length} propagated)`);
}

main();
```

Consuming it in the CI release job — two `nx release version` passes, since directly-changed projects should have their bump size driven by their own conventional commits, while propagation-only dependents (which by construction have no commits of their own) get an explicit forced patch bump:

```bash
#!/bin/bash
# .ci/release-jobs.gitlab-ci.yml (script block for the release stage)
set -euo pipefail

node tools/scripts/compute-release-set.js --base="$LAST_RELEASE_TAG" --head="$CI_COMMIT_SHA"

CHANGED=$(node -pe "require('./release-set.json').changedOnly.join(',')")
PROPAGATED=$(node -pe "require('./release-set.json').propagatedOnly.join(',')")

if [ -z "$CHANGED" ] && [ -z "$PROPAGATED" ]; then
  echo "Nothing to release."
  exit 0
fi

# Pass 1: projects with their own commits — let conventional commits pick the bump
if [ -n "$CHANGED" ]; then
  npx nx release version --projects="$CHANGED" --dry-run
  npx nx release version --projects="$CHANGED"
fi

# Pass 2: pure reverse-dependency propagation — forced patch bump only
if [ -n "$PROPAGATED" ]; then
  npx nx release version --projects="$PROPAGATED" --specifier=patch --dry-run
  npx nx release version --projects="$PROPAGATED" --specifier=patch
fi

# Changelog + publish across the full computed set
FULL_SET="${CHANGED:+$CHANGED,}${PROPAGATED}"
npx nx release changelog --projects="$FULL_SET" --git-commit --git-push --git-tag
npx nx affected -t publish --projects="$FULL_SET"
```

### 9.5 Guardrails Against Release Cascades

Walking the reverse-dependency graph without limits is how a one-line fix in a leaf utility ends up releasing half the monorepo. Guard against that explicitly:

- **Tag-based cascade opt-out.** Mark projects that should never trigger propagation to their dependents (e.g. experimental libraries, or ones with no meaningful external consumers) with a tag like `release:no-cascade`, and check it in the reverse-walk before recursing further.
- **Publishability filtering.** Only cascade into projects that are actually published/consumed externally (`release:publishable` tag or equivalent). Internal-only helper libraries and plain deployable apps don't need a forced version bump just because something below them changed — see the `filterReleasable` step in the script above.
- **Depth cap.** `MAX_CASCADE_DEPTH` in the script above stops propagation after N hops by default. If you regularly hit the cap, treat it as a signal, not just a technical limit: a library that sits 4+ hops upstream of half the graph is architecturally too central, and every change to it will keep causing wide cascades until it's split or its consumers are decoupled.
- **Cycle detection.** NX's own project graph should already reject true circular *build* dependencies, but implicit dependencies, type-only imports, or manually declared `implicitDependencies` can still produce a cycle that a naive reverse walk would loop on forever. The script above tracks the current DFS path and throws immediately if it revisits a project already on that path — fail the pipeline loudly here rather than silently truncating.
- **Group boundaries as blast-radius limits.** Keep [release groups](#5-release-groups) aligned with real architectural boundaries (e.g. `python-libs` vs. `vue-ui-kit`). A reverse-dependency cascade crossing from one bounded context into an unrelated one is usually a sign that either the dependency itself is misplaced, or the cascade shouldn't be automatic there — require manual sign-off for cross-group propagation rather than auto-releasing it.
- **Auditable output, not silent automation.** Always emit the computed release set (as `release-set.json` above) as a pipeline artifact before publishing anything, and log the `reason`/`depth` for every propagated project. When a release touches ten libraries instead of one, whoever is watching the pipeline needs to see *why* in the log, not just a list of package names.
- **Manual escape hatch.** Provide a way to force-skip propagation for an exceptional case (e.g. a commit trailer like `Release-Cascade: skip` that the script checks for) for situations where the team has already verified downstream compatibility out of band and doesn't want a forced release.

---

## 10. Implementation Checklist

- [ ] `nx.json` `release` block configured with the right `projectsRelationship` (`independent` by default; `fixed` only for genuinely coupled groups)
- [ ] Conventional Commits enforced via commit-msg hook, with project-scoped messages (`feat(project-name): ...`)
- [ ] `@nxlv/python/release/version-actions` wired in for Python projects; `@jscutlery/semver` wired in only for JS/Vue projects
- [ ] Selective release verified: `nx release --dry-run` on a single library, confirming only that library (and nothing else) is touched
- [ ] `updateDependents` policy decided (`"always"` vs. leaving it off in favor of the custom script) and documented for the team
- [ ] Reverse-dependency tags in place (`release:publishable`, `release:internal-only`, `release:no-cascade`) across all projects
- [ ] `tools/scripts/compute-release-set.js` (or equivalent) implemented and checked into the repo
- [ ] `nx graph --file=` export automated and available as a CI artifact for every release run
- [ ] Cascade depth cap configured and tuned against the real graph (not left at an arbitrary default)
- [ ] Cycle detection guard fails the pipeline loudly rather than silently truncating or looping
- [ ] `release-set.json` (or equivalent computed release list) published as a pipeline artifact and reviewed before `nx release publish` runs
- [ ] Changelog entries distinguish "released due to own change" from "released due to upstream dependency propagation"
- [ ] Dry-run validated against at least one real historical change that had known downstream dependents, before enabling automatic cascade releases in CI
- [ ] Rollback/hotfix procedure documented for the case where a cascade release itself introduces a regression across multiple packages at once
