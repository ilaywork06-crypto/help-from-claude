# GitLab CI/CD Integration and Operations

> **Audience:** DevOps engineers and senior developers responsible for the CI/CD pipeline of a large Python + Vue.js monorepo.
> **Scope:** This file covers the operational and infrastructure layer only — how GitLab CI/CD, Docker images, the package registry, and air-gapped networking tie NX, UV, and Ruff together into one working pipeline. It does **not** re-explain NX concepts (see `02-nx-workspace-fundamentals.md`), UV workspace mechanics (see `03-uv-dependency-management.md`), Ruff/formatter configuration (see `04-ruff-and-custom-formatter.md`), or the release/reverse-dependency strategy (see `05-nx-release-and-reverse-dependencies.md`).

---

## Table of Contents

1. [Pipeline Architecture Overview](#1-pipeline-architecture-overview)
2. [Full Reference `.gitlab-ci.yml`](#2-full-reference-gitlab-ciyml)
3. [Dynamic Child Pipelines Driven by NX Affected](#3-dynamic-child-pipelines-driven-by-nx-affected)
4. [CI Docker Images: NX + UV + Ruff](#4-ci-docker-images-nx--uv--ruff)
5. [GitLab Package Registry as a Private PyPI Index](#5-gitlab-package-registry-as-a-private-pypi-index)
6. [Vue.js in the Same Pipeline](#6-vuejs-in-the-same-pipeline)
7. [Reusable CI Template Library](#7-reusable-ci-template-library)
8. [Caching Strategy in GitLab CI](#8-caching-strategy-in-gitlab-ci)
9. [Air-Gapped / Closed-Network Configuration](#9-air-gapped--closed-network-configuration)
10. [Common Pitfalls (CI and Infrastructure)](#10-common-pitfalls-ci-and-infrastructure)
11. [Master Implementation Checklist](#11-master-implementation-checklist)

---

## 1. Pipeline Architecture Overview

The pipeline exists to answer one question as cheaply as possible: **what actually needs to build, lint, and test for this change?** NX affected supplies the answer; GitLab CI/CD, UV, and Ruff execute it.

```
┌──────────────────────────────────────────────────────────────┐
│                       GitLab CI Pipeline                     │
│                                                                │
│  validate → generate → build → test → lint → publish → release│
│                                                                │
│  - "validate": workspace sanity, lockfile check, format check │
│  - "generate": compute NX_BASE/NX_HEAD, optionally emit a     │
│                 dynamic child pipeline                        │
│  - NX affected fans out only the Python + Vue projects that   │
│    actually changed                                           │
│  - UV installs/builds Python packages; Ruff lints/formats them│
│  - "publish": wheels → GitLab Package Registry                │
│  - "release": nx release (see 05-nx-release-and-              │
│    reverse-dependencies.md)                                    │
└──────────────────────────────────────────────────────────────┘
```

Design principles that hold across every job in this file:

- **One custom Docker image** (§4) carries Node.js, NX, UV, and Ruff pre-installed — no job should install tools at runtime.
- **`NX_BASE`/`NX_HEAD` are computed once**, consistently, in every job that calls `nx affected` (see the shared snippet in §2).
- **Every external dependency resolves internally** — Docker images, npm packages, Python packages, and CI runner cache all point at in-network services (§9).
- **Cache is layered**: node_modules, the UV package cache, and the NX computation cache are cached independently, each keyed off the file that actually invalidates it (§8).

---

## 2. Full Reference `.gitlab-ci.yml`

This is a complete, working pipeline skeleton. It combines `nx affected`, UV, and Ruff across `validate → test → build → release`. Later sections show how to split this into includable templates (§7) and swap in a dynamic child pipeline (§3).

```yaml
# .gitlab-ci.yml

variables:
  CI_IMAGE: "registry.company.internal/ci/nx-uv-ruff:latest"

  # NX
  CI: "true"
  NX_DAEMON: "false"                 # no daemon in CI
  NX_VERBOSE_LOGGING: "false"

  # UV — see 09-... for why each of these matters
  UV_CACHE_DIR: ".uv-cache"
  UV_LINK_MODE: "copy"               # required: GitLab CI mounts break hardlinks
  UV_FROZEN: "true"                  # never let CI silently rewrite uv.lock
  UV_PYTHON_DOWNLOADS: "never"       # air-gapped: use the interpreter baked into the image
  UV_SYSTEM_PYTHON: "0"

  # Full history so NX can diff correctly (see §10)
  GIT_DEPTH: "0"

stages:
  - validate
  - test
  - build
  - publish
  - release

default:
  image: $CI_IMAGE
  interruptible: true

# ==========================================================
# Shared before_script: compute NX_BASE / NX_HEAD once
# ==========================================================
.nx-base: &nx-base
  before_script:
    - npm ci --cache .npm --prefer-offline
    - export NX_HEAD="$CI_COMMIT_SHA"
    - |
      if [ -n "$CI_MERGE_REQUEST_DIFF_BASE_SHA" ]; then
        export NX_BASE="$CI_MERGE_REQUEST_DIFF_BASE_SHA"
      elif [ "$CI_COMMIT_BEFORE_SHA" = "0000000000000000000000000000000000000000" ]; then
        export NX_BASE="$(git merge-base HEAD origin/$CI_DEFAULT_BRANCH || echo HEAD~1)"
      else
        export NX_BASE="${CI_COMMIT_BEFORE_SHA:-HEAD~1}"
      fi
    - echo "NX_BASE=$NX_BASE  NX_HEAD=$NX_HEAD"

.cache-layers: &cache-layers
  cache:
    - key:
        prefix: "v1-node"
        files: [package-lock.json]
      paths: [node_modules/, .npm/]
      policy: pull-push
    - key:
        prefix: "v1-uv"
        files: [uv.lock]
      paths: [.uv-cache/]
      policy: pull-push
    - key:
        prefix: "v1-nx"
        files: [nx.json, package-lock.json, uv.lock]
      paths: [.nx/cache/]
      policy: pull-push

# ==========================================================
# validate
# ==========================================================
validate:lockfile:
  stage: validate
  needs: []
  script:
    - uv lock --check
  rules:
    - changes: ["**/pyproject.toml", "pyproject.toml", "uv.lock"]

validate:format:
  stage: validate
  extends: .nx-base
  <<: *cache-layers
  script:
    - npx nx affected -t format-check --base=$NX_BASE --head=$NX_HEAD --parallel=4

validate:lint:
  stage: validate
  extends: .nx-base
  <<: *cache-layers
  script:
    - npx nx affected -t lint --base=$NX_BASE --head=$NX_HEAD --parallel=4
  artifacts:
    reports:
      codequality: ruff-code-quality.json
    when: always
    expire_in: 7 days

# ==========================================================
# test
# ==========================================================
test:affected:
  stage: test
  extends: .nx-base
  <<: *cache-layers
  needs: ["validate:lint", "validate:format"]
  script:
    - npx nx affected -t test --base=$NX_BASE --head=$NX_HEAD --parallel=2
  artifacts:
    reports:
      junit: "**/test-results.xml"
      coverage_report:
        coverage_format: cobertura
        path: "**/coverage.xml"
    when: always
    expire_in: 7 days

# ==========================================================
# build
# ==========================================================
build:affected:
  stage: build
  extends: .nx-base
  <<: *cache-layers
  needs: ["test:affected"]
  script:
    - npx nx affected -t build --base=$NX_BASE --head=$NX_HEAD --parallel=4
  artifacts:
    paths: ["**/dist/"]
    expire_in: 7 days
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ==========================================================
# publish — see §5 for registry configuration
# ==========================================================
publish:packages:
  stage: publish
  extends: .nx-base
  needs: ["build:affected"]
  script:
    - npx nx affected -t publish --base=$NX_BASE --head=$NX_HEAD
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual

# ==========================================================
# release — mechanics/strategy in 05-nx-release-and-reverse-dependencies.md
# ==========================================================
release:
  stage: release
  extends: .nx-base
  needs: ["publish:packages"]
  variables:
    GIT_PUSH_TOKEN: "$RELEASE_DEPLOY_TOKEN"   # must carry write_repository scope
  before_script:
    - !reference [.nx-base, before_script]
    - git config user.email "ci-release@company.internal"
    - git config user.name "GitLab CI Release Bot"
    - git remote set-url origin "https://gitlab-ci-token:${GIT_PUSH_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git"
  script:
    - npx nx release --dry-run
    - npx nx release
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual
  environment:
    name: production
```

Key CI-layer decisions embedded above:

| Decision | Why |
|---|---|
| `GIT_DEPTH: "0"` | Shallow clones truncate history NX needs to diff against — see pitfall in §10. |
| `UV_FROZEN: "true"` | CI must never silently mutate `uv.lock`; a stale lock should fail the pipeline, not auto-fix it. |
| Single `.nx-base` anchor | Every job computes `NX_BASE`/`NX_HEAD` identically — divergence here is a common source of "affected ran everything" bugs. |
| `release` uses a deploy token, not `CI_JOB_TOKEN` | `CI_JOB_TOKEN` cannot push to protected branches; see §10. |

---

## 3. Dynamic Child Pipelines Driven by NX Affected

Static YAML can't express "one job per affected project" — the project list isn't known until `nx affected` runs. GitLab's **dynamic child pipelines** solve this: a first-stage job generates a `.gitlab-ci.yml` file at runtime, and a second-stage job triggers it as a child pipeline.

```yaml
# .gitlab-ci.yml (excerpt)

stages:
  - generate
  - trigger

generate:child-pipeline:
  stage: generate
  before_script:
    - npm ci --cache .npm --prefer-offline
    - export NX_HEAD=$CI_COMMIT_SHA
    - export NX_BASE=${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}
  script:
    - node tools/scripts/generate-child-pipeline.js
  artifacts:
    paths:
      - dynamic-child-pipeline.yml
    expire_in: 1 hour
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'

trigger:child-pipeline:
  stage: trigger
  trigger:
    include:
      - artifact: dynamic-child-pipeline.yml
        job: generate:child-pipeline    # must match the producing job name exactly
    strategy: depend                    # parent pipeline waits on & reflects child status
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

The generator script queries `nx show projects --affected --withTarget=<target>` for each target, then emits one job per affected project (with `needs` wiring `test` after `build` where both are affected). Trimmed to the essentials:

```javascript
#!/usr/bin/env node
// tools/scripts/generate-child-pipeline.js
const { execSync } = require('child_process');
const fs = require('fs');
const yaml = require('js-yaml');

const NX_BASE = process.env.NX_BASE || 'HEAD~1';
const NX_HEAD = process.env.NX_HEAD || 'HEAD';
const CI_IMAGE = process.env.CI_IMAGE || 'registry.company.internal/ci/nx-uv-ruff:latest';

function affected(target) {
  try {
    const out = execSync(
      `npx nx show projects --affected --base=${NX_BASE} --head=${NX_HEAD} --withTarget=${target}`,
      { encoding: 'utf8' }
    ).trim();
    return out ? out.split('\n').filter(Boolean) : [];
  } catch {
    return [];
  }
}

function projectRoot(name) {
  const info = JSON.parse(execSync(`npx nx show project ${name} --json`, { encoding: 'utf8' }));
  return info.root || name;
}

function job(project, target, stage) {
  return {
    stage,
    image: CI_IMAGE,
    interruptible: true,
    before_script: ['npm ci --cache .npm --prefer-offline'],
    script: [`npx nx run ${project}:${target}`],
    ...(target === 'build' && {
      artifacts: { paths: [`${projectRoot(project)}/dist`], expire_in: '7 days' },
    }),
  };
}

const lint = affected('lint');
const build = affected('build');
const test = affected('test');

if (!lint.length && !build.length && !test.length) {
  fs.writeFileSync('dynamic-child-pipeline.yml', yaml.dump({
    stages: ['noop'],
    'nothing-affected': { stage: 'noop', image: 'alpine:latest', script: ['echo "No affected projects"'] },
  }));
  process.exit(0);
}

const pipeline = { stages: ['lint', 'build', 'test'], variables: { CI_IMAGE } };
for (const p of lint) pipeline[`lint:${p}`] = job(p, 'lint', 'lint');
for (const p of build) pipeline[`build:${p}`] = job(p, 'build', 'build');
for (const p of test) {
  const j = job(p, 'test', 'test');
  if (build.includes(p)) j.needs = [`build:${p}`];
  pipeline[`test:${p}`] = j;
}

fs.writeFileSync('dynamic-child-pipeline.yml', yaml.dump(pipeline, { lineWidth: 120 }));
```

Two rules keep this reliable:

- **Always emit a valid pipeline, even for "nothing affected."** GitLab's `trigger` job fails if the included YAML has no stages/jobs at all, so emit a harmless no-op job instead of an empty file.
- **The `artifact:`/`job:` pair in the `trigger` block must reference the exact job name that produced the artifact** — a rename on one side without the other is the single most common breakage here (see §10).

A pre-built alternative that avoids maintaining the generator script yourself is [`nx-gitlab-ci-filter-affected`](https://github.com/jase88/nx-gitlab-ci-filter-affected), which reads a static pipeline template and filters it down to affected projects:

```yaml
generate:affected-pipeline:
  stage: generate
  script:
    - npm ci --cache .npm --prefer-offline
    - export NX_HEAD=$CI_COMMIT_SHA
    - export NX_BASE=${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}
    - npx nx-gitlab-ci-filter-affected --input .ci/pipeline-template.yml --output affected-pipeline.yml
  artifacts:
    paths: [affected-pipeline.yml]
    expire_in: 1 hour

trigger:affected:
  stage: trigger
  trigger:
    include:
      - artifact: affected-pipeline.yml
        job: generate:affected-pipeline
    strategy: depend
```

---

## 4. CI Docker Images: NX + UV + Ruff

Every job in this pipeline should run from one pre-built image — installing Node.js, UV, or Ruff at job runtime wastes minutes per job and is exactly the kind of external download that fails in an air-gapped runner.

### 4.1 Full multi-stage Dockerfile

```dockerfile
# docker/ci-runner/Dockerfile
FROM ubuntu:22.04 AS base

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl git ca-certificates build-essential libssl-dev libffi-dev \
    && rm -rf /var/lib/apt/lists/*

# --- Node.js (for NX) ---
FROM base AS node-stage
ARG NODE_VERSION=20.18.0
# In an air-gapped build, fetch this tarball from an internal mirror instead.
RUN curl -fsSL "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-x64.tar.gz" \
    | tar -xz -C /usr/local --strip-components=1

# --- Python + UV ---
FROM node-stage AS python-stage
ARG PYTHON_VERSION=3.12.9
ARG UV_VERSION=0.7.0
RUN curl -fsSL "https://www.python.org/ftp/python/${PYTHON_VERSION}/Python-${PYTHON_VERSION}.tgz" \
    | tar -xz -C /tmp && \
    cd /tmp/Python-${PYTHON_VERSION} && \
    ./configure --enable-optimizations --with-lto && \
    make -j$(nproc) && make install && \
    rm -rf /tmp/Python-${PYTHON_VERSION}
RUN ln -sf /usr/local/bin/python3 /usr/local/bin/python && \
    ln -sf /usr/local/bin/pip3 /usr/local/bin/pip
RUN pip install uv==${UV_VERSION}

# --- Ruff ---
FROM python-stage AS ruff-stage
ARG RUFF_VERSION=0.9.10
RUN pip install ruff==${RUFF_VERSION} twine

# --- Final image ---
FROM ruff-stage AS final
ENV UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never \
    UV_CACHE_DIR=/tmp/.uv-cache \
    NX_DAEMON=false
RUN node --version && npm --version && python --version && uv --version && ruff --version
WORKDIR /workspace
CMD ["bash"]
```

### 4.2 Faster variant — build on Astral's UV base image

```dockerfile
# docker/ci-runner/Dockerfile.uv-base
FROM ghcr.io/astral-sh/uv:0.7.0-python3.12-bookworm-slim AS uv-base

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl git ca-certificates && rm -rf /var/lib/apt/lists/*

ARG NODE_VERSION=20
RUN curl -fsSL https://deb.nodesource.com/setup_${NODE_VERSION}.x | bash - && \
    apt-get install -y nodejs && rm -rf /var/lib/apt/lists/*

RUN pip install ruff twine

ENV UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never \
    NX_DAEMON=false

WORKDIR /workspace
RUN node --version && uv --version && ruff --version
```

### 4.3 Build and push to the internal registry

Rebuild the image only when its own Dockerfile changes — not on every commit:

```yaml
build:ci-docker-image:
  stage: validate
  image: docker:24
  services: [docker:24-dind]
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    CI_IMAGE_TAG: "${CI_REGISTRY_IMAGE}/ci-runner:${CI_COMMIT_SHORT_SHA}"
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker build -t "$CI_IMAGE_TAG" -t "${CI_REGISTRY_IMAGE}/ci-runner:latest" docker/ci-runner/
    - docker push "$CI_IMAGE_TAG"
    - docker push "${CI_REGISTRY_IMAGE}/ci-runner:latest"
  rules:
    - changes: ["docker/ci-runner/Dockerfile", "docker/ci-runner/Dockerfile.uv-base"]
      if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
```

---

## 5. GitLab Package Registry as a Private PyPI Index

GitLab's Package Registry can serve as the workspace's private PyPI, both for consuming shared internal libraries and for publishing releases (built by NX/UV, published as configured in `05-nx-release-and-reverse-dependencies.md`).

### 5.1 Consuming — index configuration

```toml
# pyproject.toml (workspace root)
[[tool.uv.index]]
name = "corporate-pypi"
url = "https://gitlab.company.internal/api/v4/groups/MY_GROUP_ID/-/packages/pypi/simple"
default = true

# Public PyPI as a fallback — omit entirely on a fully closed network
[[tool.uv.index]]
name = "pypi"
url = "https://pypi.org/simple"
```

Never hardcode credentials in `pyproject.toml`. Authenticate via environment variables in CI:

```yaml
variables:
  UV_INDEX_CORPORATE_PYPI_USERNAME: "gitlab-ci-token"
  UV_INDEX_CORPORATE_PYPI_PASSWORD: "$CI_JOB_TOKEN"
```

or via `.netrc`:

```yaml
before_script:
  - |
    cat > ~/.netrc << EOF
    machine gitlab.company.internal
    login gitlab-ci-token
    password ${CI_JOB_TOKEN}
    EOF
  - chmod 600 ~/.netrc
```

For a **group-level** index (shared across many projects publishing to the same registry):

```toml
[[tool.uv.index]]
name = "gitlab-group"
url = "https://gitlab.company.internal/api/v4/groups/{group_id}/-/packages/pypi/simple"
default = true
```

For local development, `~/.config/uv/uv.toml`:

```toml
[[index]]
name = "corporate-pypi"
url = "https://gitlab.company.internal/api/v4/groups/MY_GROUP_ID/-/packages/pypi/simple"
default = true
```

### 5.2 Publishing — CI job

`uv publish` targets the project-level PyPI endpoint directly:

```bash
uv publish \
  --publish-url "https://gitlab.company.internal/api/v4/projects/${CI_PROJECT_ID}/packages/pypi" \
  --username "gitlab-ci-token" \
  --password "${CI_JOB_TOKEN}"
```

Or, if `twine` is preferred (e.g. from a NX `@nxlv/python:publish` executor or a plain script):

```yaml
publish:python-package:
  stage: publish
  script:
    - uv build --package $PACKAGE_NAME --out-dir dist/
    - |
      uv run twine upload \
        --repository-url "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/pypi" \
        --username "gitlab-ci-token" \
        --password "${CI_JOB_TOKEN}" \
        dist/*.whl dist/*.tar.gz
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual
```

### 5.3 Critical setting: disable PyPI forwarding

By default, GitLab forwards package lookups to PyPI.org when a package isn't found locally. On a closed network — or simply for supply-chain hygiene — this must be disabled explicitly:

> **Group Settings → Packages and registries → Package registry → disable "Forward PyPI package requests to PyPI.org".**

Without this, a package missing from the internal registry silently resolves from the public internet instead of failing loudly, which defeats the purpose of an air-gapped index.

---

## 6. Vue.js in the Same Pipeline

Python and Vue.js projects live in the same NX workspace and the same pipeline; NX affected treats both uniformly, but the underlying executors and jobs differ.

### 6.1 `project.json` for a Vue app (via `@nx/vite`)

```json
{
  "name": "frontend-dashboard",
  "projectType": "application",
  "sourceRoot": "apps/frontend-dashboard/src",
  "targets": {
    "build": {
      "executor": "@nx/vite:build",
      "options": {
        "outputPath": "dist/apps/frontend-dashboard",
        "configFile": "apps/frontend-dashboard/vite.config.ts"
      },
      "cache": true,
      "inputs": ["production", "^production"],
      "outputs": ["{projectRoot}/dist"]
    },
    "test": {
      "executor": "@nx/vite:test",
      "options": { "configFile": "apps/frontend-dashboard/vite.config.ts" },
      "cache": true
    },
    "lint": {
      "executor": "@nx/eslint:lint",
      "options": { "lintFilePatterns": ["apps/frontend-dashboard/**/*.{ts,vue}"] },
      "cache": true
    }
  }
}
```

### 6.2 Frontend jobs, filtered to affected Vue projects

```yaml
# .ci/frontend-jobs.gitlab-ci.yml

frontend:build-affected:
  stage: build
  extends: .nx-base
  cache:
    - key: { prefix: "v1-node", files: [package-lock.json] }
      paths: [node_modules/, .npm/]
      policy: pull
    - key: { prefix: "v1-nx", files: [nx.json, package-lock.json] }
      paths: [.nx/cache/]
      policy: pull-push
  script:
    - npx nx affected -t build --base=$NX_BASE --head=$NX_HEAD --projects='apps/frontend-*,libs/frontend/*' --parallel=2
  artifacts:
    paths: [dist/apps/]
    expire_in: 7 days

frontend:test-affected:
  stage: test
  extends: .nx-base
  needs: [frontend:build-affected]
  script:
    - npx nx affected -t test --base=$NX_BASE --head=$NX_HEAD --projects='apps/frontend-*,libs/frontend/*'
  artifacts:
    reports:
      junit: "**/test-results.xml"
      coverage_report:
        coverage_format: cobertura
        path: "**/coverage/cobertura-coverage.xml"

frontend:deploy-affected:
  stage: publish
  needs: [frontend:build-affected, frontend:test-affected]
  extends: .nx-base
  script:
    - |
      AFFECTED_APPS=$(npx nx show projects --affected --base=$NX_BASE --head=$NX_HEAD \
        --withTarget=build --projects='apps/frontend-*' 2>/dev/null || echo "")
    - |
      for app in $AFFECTED_APPS; do
        echo "Deploying $app..."
        # deployment logic — Kubernetes, S3/CDN, Nginx, etc.
      done
  rules:
    - if: '$CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH'
      when: manual
```

Splitting Python and frontend jobs by `--projects='apps/frontend-*,libs/frontend/*'` (or the mirrored Python glob) keeps a single pipeline definition from running Vue-only tooling against Python-only changes and vice versa — cheaper than two entirely separate pipelines, without losing the isolation.

---

## 7. Reusable CI Template Library

A monorepo with dozens of similar projects needs its pipeline definition split into includable templates rather than one growing `.gitlab-ci.yml`.

```yaml
# .gitlab-ci.yml — the orchestrator, wires templates together
include:
  - local: .ci/variables.gitlab-ci.yml
  - local: .ci/python-jobs.gitlab-ci.yml
  - local: .ci/frontend-jobs.gitlab-ci.yml
  - local: .ci/release-jobs.gitlab-ci.yml
  - local: .ci/docker-jobs.gitlab-ci.yml

stages:
  - validate
  - generate
  - build
  - test
  - lint
  - publish
  - release
```

```yaml
# .ci/variables.gitlab-ci.yml
variables:
  CI_IMAGE: "registry.company.internal/ci/nx-uv-ruff:latest"
  UV_CACHE_DIR: ".uv-cache"
  UV_LINK_MODE: "copy"
  UV_PYTHON_DOWNLOADS: "never"
  UV_INDEX_CORPORATE_PYPI_USERNAME: "gitlab-ci-token"
  UV_INDEX_CORPORATE_PYPI_PASSWORD: "$CI_JOB_TOKEN"
  NX_DAEMON: "false"
  NX_VERBOSE_LOGGING: "false"
  CACHE_VERSION: "v4"
  GIT_DEPTH: "50"
  GIT_STRATEGY: "fetch"
```

```yaml
# .ci/python-jobs.gitlab-ci.yml
.python-base:
  image: $CI_IMAGE
  interruptible: true
  before_script:
    - npm ci --cache .npm --prefer-offline --quiet
    - export NX_HEAD=$CI_COMMIT_SHA
    - export NX_BASE=${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}

python:lint:
  extends: .python-base
  stage: lint
  script:
    - npx nx affected -t lint --base=$NX_BASE --head=$NX_HEAD --parallel=4
  artifacts:
    reports: { codequality: ruff-code-quality.json }
    when: always
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

python:test:
  extends: .python-base
  stage: test
  script:
    - npx nx affected -t test --base=$NX_BASE --head=$NX_HEAD --parallel=2
  artifacts:
    reports:
      junit: "**/test-results.xml"
      coverage_report: { coverage_format: cobertura, path: "**/coverage.xml" }

python:build:
  extends: .python-base
  stage: build
  script:
    - npx nx affected -t build --base=$NX_BASE --head=$NX_HEAD
  artifacts:
    paths: ["**/dist/*.whl", "**/dist/*.tar.gz"]
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'
```

Splitting this way means adding a new project category (e.g. a data-pipeline job family) is a new `.ci/*.yml` file plus one `include:` line, not a merge conflict in a monolithic pipeline file that every team edits.

---

## 8. Caching Strategy in GitLab CI

Four independent cache layers cover this pipeline; each is keyed off the one file that should actually invalidate it.

| Layer | Contents | Key basis | Policy |
|---|---|---|---|
| `node_modules/`, `.npm/` | JS deps for NX itself | `package-lock.json` | pull-push |
| `.uv-cache/` | Downloaded Python wheels/sdists | `uv.lock` | pull-push |
| `.nx/cache/` | NX task computation results | `nx.json` + `package-lock.json` + `uv.lock` | pull-push |
| `dist/` | Build artifacts | n/a — job artifact, not cache | — |

```yaml
.cache-node: &cache-node
  key: { prefix: "${CACHE_VERSION}-node-${CI_JOB_NAME}", files: [package-lock.json] }
  paths: [node_modules/, .npm/]
  policy: pull-push
  fallback_keys: ["${CACHE_VERSION}-node-"]

.cache-uv: &cache-uv
  key: { prefix: "${CACHE_VERSION}-uv", files: [uv.lock] }
  paths: [.uv-cache/]
  policy: pull-push
  fallback_keys: ["${CACHE_VERSION}-uv-"]

.cache-nx: &cache-nx
  key: { prefix: "${CACHE_VERSION}-nx", files: [nx.json, package-lock.json, uv.lock] }
  paths: [.nx/cache/]
  policy: pull-push
  fallback_keys: ["${CACHE_VERSION}-nx-"]
```

Practical rules that keep this from rotting:

- **MR pipelines read, `main` writes.** Give merge request jobs `policy: pull` and reserve `pull-push` for the default branch, so a bad MR can't poison the shared cache for everyone else.
- **Bump `CACHE_VERSION` to hard-reset everything** rather than deleting caches by hand.
- **A dedicated `cache:warm-all` job on `main`** that runs `uv sync`, `npm ci`, and prunes the UV cache keeps subsequent MR pipelines fast without every job re-downloading dependencies.
- For a true **remote/shared NX computation cache** across runners in a closed network (MinIO as an S3-compatible backend, or a filesystem-based `portable-nx-cache` server), see §9 — this is an air-gapped concern, not a plain GitLab-cache one.

---

## 9. Air-Gapped / Closed-Network Configuration

Everything in this pipeline that normally reaches the public internet needs an internal substitute. This section is the consolidated map across NX, UV, Ruff, Docker, and GitLab Runner itself.

```
╔════════════════════════════════════════════════════════════╗
║                    Corporate Network                        ║
║                                                              ║
║  GitLab Self-Hosted  ◄──►  Internal Registries               ║
║  - source, CI, Package Registry   - Docker Registry/Harbor   ║
║                                    - PyPI mirror (GitLab      ║
║  MinIO (NX remote cache)             Package Reg./Nexus)      ║
║                                    - npm mirror (Verdaccio)   ║
║  GitLab Runners (on-prem, Docker)                             ║
╚════════════════════════════════════════════════════════════╝
```

| Component | Public dependency | Internal substitute |
|---|---|---|
| NX / npm packages | registry.npmjs.org | Verdaccio or Nexus npm proxy |
| Python packages | pypi.org | GitLab Package Registry, Artifactory, Nexus, or Devpi |
| UV / Ruff binaries | GitHub Releases | Mirrored into Artifactory/GitLab generic package registry, baked into the CI image |
| Docker base images | Docker Hub / ghcr.io | GitLab Container Registry (mirrored) |
| Node.js runtime | nodejs.org | Internal mirror or OS package manager |
| NX remote cache | NX Cloud | MinIO (S3-compatible, on-prem) or a self-hosted cache server |

### 9.1 Python packages — internal PyPI mirror

```toml
[[tool.uv.index]]
name = "artifactory"
url = "https://artifactory.internal.company.com/artifactory/api/pypi/pypi-virtual/simple/"
publish-url = "https://artifactory.internal.company.com/artifactory/api/pypi/pypi-local"
default = true

[tool.uv]
no-index = true              # maximum isolation: only this index is ever consulted
index-strategy = "first-index"
python-preference = "only-system"
python-downloads = "never"
```

> **Critical Artifactory/Nexus setting:** disable "Forward requests to PyPI.org" on the virtual/proxy repository (the same concern as GitLab's own PyPI forwarding toggle in §5.3). Without it, a missing internal package silently falls through to the public internet.

### 9.2 UV binary and runtime, offline

```bash
# On a machine with internet access, download the release tarball:
# https://github.com/astral-sh/uv/releases
tar -xzf uv-x86_64-unknown-linux-gnu.tar.gz
sudo cp uv /usr/local/bin/ && sudo chmod +x /usr/local/bin/uv
```

```bash
export UV_NO_MANAGED_PYTHON=1
export UV_PYTHON_DOWNLOADS=never
export UV_INDEX_URL="https://artifactory.internal.company.com/artifactory/api/pypi/pypi-virtual/simple/"
```

In practice, none of this should run at job time — bake the UV and Ruff binaries into the CI image (§4) so no job ever attempts an outbound download.

### 9.3 npm packages — Verdaccio

```yaml
# docker-compose.verdaccio.yml
services:
  verdaccio:
    image: verdaccio/verdaccio:5
    ports: ["4873:4873"]
    volumes:
      - verdaccio-storage:/verdaccio/storage
      - ./verdaccio-config:/verdaccio/conf
volumes:
  verdaccio-storage:
```

```yaml
# verdaccio-config/config.yaml
storage: /verdaccio/storage/data
auth:
  htpasswd: { file: /verdaccio/storage/htpasswd, max_users: -1 }
uplinks:
  npmjs: { url: https://registry.npmjs.org/, timeout: 30s, max_fails: 2, fail_timeout: 5m }
packages:
  '@mycompany/*': { access: $authenticated, publish: $authenticated, unpublish: $authenticated }
  '**': { access: $anonymous, publish: $authenticated, proxy: npmjs }
```

```yaml
# .gitlab-ci.yml
variables:
  NPM_CONFIG_REGISTRY: "http://verdaccio.company.internal:4873/"
before_script:
  - echo "//verdaccio.company.internal:4873/:_authToken=${NPM_TOKEN}" >> ~/.npmrc
  - npm ci --cache .npm --prefer-offline
```

### 9.4 Docker images and GitLab Runner

Mirror every external image into the internal registry, then forbid the runner from ever reaching outside it:

```bash
#!/bin/bash
# tools/scripts/sync-docker-images.sh — run on a machine with internet access
INTERNAL_REGISTRY="registry.company.internal"
IMAGES=(
  "ghcr.io/astral-sh/uv:0.7.0-python3.12-bookworm-slim"
  "ghcr.io/astral-sh/ruff:0.9.10-alpine"
  "node:20-bookworm-slim"
  "minio/minio:latest"
  "verdaccio/verdaccio:5"
)
for image in "${IMAGES[@]}"; do
  name=$(basename "$image"); short_name=${name%%:*}; tag=${name##*:}
  docker pull "$image"
  docker tag "$image" "$INTERNAL_REGISTRY/mirrors/$short_name:$tag"
  docker push "$INTERNAL_REGISTRY/mirrors/$short_name:$tag"
done
```

```toml
# /etc/gitlab-runner/config.toml
[[runners]]
  name = "corporate-runner"
  executor = "docker"
  [runners.docker]
    pull_policy = ["if-not-present", "never"]   # never fall back to Docker Hub
    allowed_images = ["registry.company.internal/*"]
  [runners.cache]
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "minio.company.internal:9000"
      BucketName = "runner-cache"
```

### 9.5 NX remote cache without NX Cloud

**Option A — MinIO via `@nx/s3-cache` (recommended):**

```json
// nx.json
{
  "s3": {
    "region": "us-east-1",
    "bucket": "nx-remote-cache",
    "endpoint": "http://minio.company.internal:9000",
    "forcePathStyle": true,
    "cacheKeyPrefix": "my-monorepo",
    "localMode": "read-only",
    "ciMode": "read-write"
  }
}
```

```yaml
variables:
  AWS_ACCESS_KEY_ID: "$MINIO_ACCESS_KEY"
  AWS_SECRET_ACCESS_KEY: "$MINIO_SECRET_KEY"
  NX_KEY: "$NX_ACTIVATION_KEY"   # required for self-hosted cache since NX 20.8
```

> **Security warning — CVE-2025-36852 ("CREEP"):** bucket-based remote caches are vulnerable to cache poisoning — an MR opened by any contributor can write a cache entry that later corrupts a production build if that entry is trusted on `main`. Restrict write access to protected-branch pipelines only: `localMode: "read-only"` for MRs, `ciMode: "read-write"` for `main`.

**Option B — filesystem-based `portable-nx-cache` server**, layered on top of the ordinary GitLab CI cache when a full S3 deployment isn't justified — see the pipeline snippet in §8 for the cache block; the server itself runs as a backgrounded process in `before_script`, with the pipeline waiting on a `/ready` healthcheck before proceeding.

### 9.6 pre-commit hooks offline

```yaml
# .pre-commit-config.yaml — local hooks instead of GitHub-hosted repos
repos:
  - repo: local
    hooks:
      - id: ruff-check
        name: Ruff lint
        entry: uv run ruff check --fix
        language: system
        types: [python]
      - id: ruff-format
        name: Ruff format
        entry: uv run ruff format
        language: system
        types: [python]
```

---

## 10. Common Pitfalls (CI and Infrastructure)

These are cross-cutting operational footguns — issues in how GitLab CI, Docker, and the registry interact with NX/UV/Ruff, not the tools' own internal behavior (which is covered in their respective files).

| # | Symptom | Root Cause | Fix |
|---|---|---|---|
| 1 | `nx affected` runs every project, even on a one-line change | Shallow clone — GitLab's default `GIT_DEPTH` truncates history NX needs to diff | Set `GIT_DEPTH: "0"` (or at least a generous fixed depth) |
| 2 | `Failed to hardlink files` / "Invalid cross-device link" from UV | GitLab CI mounts the build directory and cache on separate filesystems; hardlinks can't cross that boundary | `UV_LINK_MODE: "copy"` — mandatory in every Docker-based CI job |
| 3 | `trigger` job fails with "artifact not found" for a dynamic child pipeline | The `artifact:`/`job:` pair in `trigger.include` doesn't exactly match the job that produced the file | Keep the producing job's name and the `job:` reference in sync; rename both together |
| 4 | Every pipeline starts from a cold cache | Runner cache isn't shared (local runner cache, not S3-backed) or the cache key changes unexpectedly | Configure `[runners.cache]` with a shared S3/MinIO backend; audit cache `key`/`prefix` for accidental variability |
| 5 | A remote NX cache entry corrupts a `main` build after an MR ran | CVE-2025-36852 (CREEP): bucket-based remote caches accept writes from untrusted MR pipelines | Restrict cache write access to protected branches only (`localMode: read-only`, `ciMode: read-write`) |
| 6 | A "missing" internal package silently resolves from the public internet | GitLab / Artifactory "forward to PyPI.org" proxying is still enabled | Disable PyPI forwarding at the group/repository level explicitly |
| 7 | `401 Unauthorized` pulling internal Python packages in CI | `UV_INDEX_*_USERNAME`/`PASSWORD` (or `CI_JOB_TOKEN`) not wired into the job's variables | Confirm the CI variables are set and scoped to the right index name; test with `.netrc` as a fallback |
| 8 | Ruff's GitLab Code Quality report doesn't appear on the MR | Output isn't valid GitLab codequality JSON, or the artifact path in `reports.codequality` doesn't match the file actually written | Validate with `ruff check --output-format=gitlab . \| python3 -m json.tool`; double-check the artifact path |
| 9 | GitLab Runner tries to reach Docker Hub and times out | `pull_policy` isn't restricted, or an image reference in the pipeline isn't pointed at the internal mirror | Set `pull_policy = ["if-not-present", "never"]` and `allowed_images` to the internal registry only |
| 10 | The `release` job can't push the version-bump commit/tag | `CI_JOB_TOKEN` cannot push to protected branches by design | Use a project/group deploy token or PAT with `write_repository` scope, injected as its own CI variable |
| 11 | npm install unexpectedly pulls from the public registry | Verdaccio/Nexus proxy isn't configured as the effective `.npmrc` registry for that job | Set `NPM_CONFIG_REGISTRY` (or write `.npmrc` in `before_script`) in every job, not just interactively on developer machines |

---

## 11. Master Implementation Checklist

This is the full, ordered checklist for the migration end to end. Steps are grouped by the file that owns their concepts; this file owns Phase 0 (infrastructure) and Phase 4 (pipeline wiring). Detailed how-to for each grouped step lives in the referenced file.

### Phase 0 — Infrastructure and Air-Gapped Prerequisites (this file, §4, §9)

- [ ] GitLab Self-Hosted with Container Registry and Package Registry enabled
- [ ] Internal npm registry running (Verdaccio/Nexus) and reachable from CI runners
- [ ] Internal PyPI mirror running (GitLab Package Registry, Artifactory, Nexus, or Devpi)
- [ ] "Forward PyPI package requests to PyPI.org" disabled at the group level
- [ ] All required Docker base images mirrored into the internal registry
- [ ] GitLab Runner configured with `pull_policy = ["if-not-present", "never"]` and `allowed_images`
- [ ] Custom CI image built (NX + UV + Ruff + Node.js + Python preinstalled) and pushed
- [ ] MinIO (or equivalent) deployed for NX remote cache, if a shared cache across runners is required
- [ ] Deploy token / PAT with `write_repository` scope created for release pushes

### Phase 1 — NX Workspace (see `02-nx-monorepo-management.md`)

- [ ] NX initialized, `nx.json` and `project.json` files in place for every project
- [ ] `nx graph` reflects the real dependency graph, including Python-to-Python edges
- [ ] `nx affected` verified locally against a known change

### Phase 2 — UV Migration (see `03-uv-package-manager-migration.md`)

- [ ] Workspace root `pyproject.toml` with `[tool.uv.workspace]` members defined
- [ ] Every package converted from Poetry to PEP 621 + UV
- [ ] `uv.lock` generated and committed; `poetry.lock` removed
- [ ] Private registry index configured (paired with §5 of this file for the CI side)

### Phase 3 — Ruff and Custom Formatter (see `04-ruff-and-custom-formatter.md`)

- [ ] `ruff.toml` (or `[tool.ruff]`) defined at workspace root, with per-project overrides where needed
- [ ] One-time `ruff check --fix --unsafe-fixes` + `ruff format` commit applied, team notified in advance of merge conflicts
- [ ] `flake8`/`black`/`isort`/`pylint` removed from dependencies
- [ ] Custom formatter script (`tools/format.py` or equivalent) wired as an NX target

### Phase 4 — Pipeline Wiring (this file)

- [ ] `.gitlab-ci.yml` (or template includes, §7) covers validate → test → build → publish → release
- [ ] Shared `NX_BASE`/`NX_HEAD` computation used consistently across every job (§2)
- [ ] Dynamic child pipeline (or `nx-gitlab-ci-filter-affected`) wired up and tested against an MR with zero, one, and many affected projects (§3)
- [ ] Cache layers (node/UV/NX) configured with MR-read / main-write policy (§8)
- [ ] Ruff Code Quality artifact appears correctly on MRs
- [ ] Vue.js jobs correctly scoped to `apps/frontend-*` / `libs/frontend/*` alongside Python jobs (§6)
- [ ] GitLab Package Registry publish job tested end to end (build → publish → reinstall from registry)

### Phase 5 — Release Automation (see `05-nx-release-and-reverse-dependencies.md`)

- [ ] Release strategy (independent vs. fixed versioning, reverse-dependency bumping) configured in `nx.json`
- [ ] Conventional Commits enforced (commit-msg hook)
- [ ] `nx release --dry-run` validated before enabling the real `release` CI job
- [ ] Release CI job (§2 of this file) uses a deploy token, runs manually on the default branch only

### Phase 6 — Production Hardening

- [ ] Pipeline duration measured before/after — target 60–80% reduction on unaffected changes
- [ ] Cache-warming job scheduled on the default branch
- [ ] Alerting configured on pipeline failures
- [ ] CVE-2025-36852 mitigation confirmed (remote cache write access restricted to protected branches)
- [ ] Internal documentation/runbook published for the team
