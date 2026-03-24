# מדריך מקיף: GitLab CI/CD עם NX, UV, RUFF ו-Python Monorepo בסביבה ארגונית סגורה

> **קהל יעד**: מפתחים וצוותי DevOps בחברות גדולות המשתמשות ב-GitLab Self-Hosted בתוך רשת ארגונית סגורה (air-gapped / closed corporate network).
> **מוקד**: Python Monorepo גדול + Vue.js + NX + UV + RUFF + שחרורים אוטומטיים.

---

## תוכן עניינים

1. [ארכיטקטורת Monorepo עם NX ו-UV](#1-ארכיטקטורת-monorepo-עם-nx-ו-uv)
2. [NX ב-GitLab CI/CD — יסודות](#2-nx-ב-gitlab-cicd--יסודות)
3. [NX Affected — בניה חכמה של רק מה שהשתנה](#3-nx-affected--בניה-חכמה-של-רק-מה-שהשתנה)
4. [Dynamic Child Pipelines עם NX Affected](#4-dynamic-child-pipelines-עם-nx-affected)
5. [NX Remote Cache — ללא NX Cloud בסביבה סגורה](#5-nx-remote-cache--ללא-nx-cloud-בסביבה-סגורה)
6. [NX Distributed Task Execution (Agents) ב-GitLab CI](#6-nx-distributed-task-execution-agents-ב-gitlab-ci)
7. [UV ב-GitLab CI/CD — ניהול תלויות Python](#7-uv-ב-gitlab-cicd--ניהול-תלויות-python)
8. [UV Workspaces — Python Monorepo אמיתי](#8-uv-workspaces--python-monorepo-אמיתי)
9. [RUFF ב-GitLab CI/CD — Linting ו-Formatting](#9-ruff-ב-gitlab-cicd--linting-ו-formatting)
10. [@nxlv/python — הגשר בין NX ל-Python/UV](#10-nxlvpython--הגשר-בין-nx-ל-pythonuv)
11. [GitLab Package Registry כ-Private PyPI](#11-gitlab-package-registry-כ-private-pypi)
12. [Vue.js עם NX ב-GitLab CI](#12-vuejs-עם-nx-ב-gitlab-ci)
13. [Release Automation עם NX ו-@jscutlery/semver](#13-release-automation-עם-nx-ו-jscutlerysemver)
14. [Docker Images מותאמות ל-CI: NX + UV + RUFF](#14-docker-images-מותאמות-ל-ci-nx--uv--ruff)
15. [סביבה ארגונית סגורה — שיקולים וקונפיגורציות מיוחדות](#15-סביבה-ארגונית-סגורה--שיקולים-וקונפיגורציות-מיוחדות)
16. [אסטרטגיית Caching מלאה ל-NX + UV ב-GitLab](#16-אסטרטגיית-caching-מלאה-ל-nx--uv-ב-gitlab)
17. [Template Library ב-GitLab CI לפרויקט Monorepo](#17-template-library-ב-gitlab-ci-לפרויקט-monorepo)
18. [בעיות נפוצות, פערים ואזהרות](#18-בעיות-נפוצות-פערים-ואזהרות)
19. [checklist יישום מלא](#19-checklist-יישום-מלא)

---

## 1. ארכיטקטורת Monorepo עם NX ו-UV

### מבנה תיקיות מומלץ

```
my-company-monorepo/
├── .gitlab-ci.yml                    # Pipeline ראשי
├── nx.json                           # קונפיגורציית NX
├── pyproject.toml                    # UV Workspace root
├── uv.lock                           # Lockfile משותף לכל ה-workspace
├── package.json                      # node dependencies לכלי NX
├── package-lock.json
│
├── apps/
│   ├── frontend-dashboard/           # Vue.js application
│   │   ├── project.json              # NX project config
│   │   ├── package.json
│   │   ├── vite.config.ts
│   │   └── src/
│   ├── api-service/                  # Python FastAPI application
│   │   ├── project.json
│   │   ├── pyproject.toml            # UV member package
│   │   └── src/
│   └── data-pipeline/               # Python data pipeline
│       ├── project.json
│       ├── pyproject.toml
│       └── src/
│
├── libs/
│   ├── python/
│   │   ├── shared-utils/             # Python shared library
│   │   │   ├── project.json
│   │   │   ├── pyproject.toml
│   │   │   └── src/
│   │   ├── db-models/
│   │   │   ├── project.json
│   │   │   ├── pyproject.toml
│   │   │   └── src/
│   │   └── auth-helpers/
│   │       ├── project.json
│   │       ├── pyproject.toml
│   │       └── src/
│   └── frontend/
│       ├── ui-components/            # Vue.js shared components
│       │   ├── project.json
│       │   └── src/
│       └── design-tokens/
│           ├── project.json
│           └── src/
│
├── tools/
│   ├── scripts/                      # כלי build מותאמים
│   │   ├── generate-child-pipeline.js
│   │   └── format_code.py            # הפורמטר המותאם שלכם
│   └── executors/                    # NX executors מותאמים
│
├── .ci/                              # CI/CD templates
│   ├── python-jobs.gitlab-ci.yml
│   ├── frontend-jobs.gitlab-ci.yml
│   ├── release-jobs.gitlab-ci.yml
│   └── cache-config.gitlab-ci.yml
│
└── docker/
    ├── ci-runner/
    │   └── Dockerfile                # תמונת Docker מותאמת ל-CI
    └── base-python/
        └── Dockerfile
```

### קונפיגורציית UV Workspace Root

**`pyproject.toml` (שורש ה-monorepo):**

```toml
[tool.uv.workspace]
members = [
    "apps/api-service",
    "apps/data-pipeline",
    "libs/python/shared-utils",
    "libs/python/db-models",
    "libs/python/auth-helpers",
]
# אם יש packages שאתם לא רוצים כחלק מה-workspace:
# exclude = ["apps/legacy-service"]

[tool.uv]
# Python version מינימלי עבור כל ה-workspace
requires-python = ">=3.11"

# Index ראשי — GitLab Package Registry הפנימי שלכם
[[tool.uv.index]]
name = "corporate-pypi"
url = "https://gitlab.company.internal/api/v4/groups/MY_GROUP_ID/-/packages/pypi/simple"
default = true

# PyPI ציבורי כ-fallback (אם הרשת מאפשרת — אם לא, להסיר)
[[tool.uv.index]]
name = "pypi"
url = "https://pypi.org/simple"

[tool.uv.sources]
# תלויות פנימיות — מה-workspace עצמו
shared-utils = { workspace = true }
db-models = { workspace = true }
auth-helpers = { workspace = true }
```

### קונפיגורציית NX ראשית

**`nx.json`:**

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "npmScope": "mycompany",
  "defaultBase": "main",
  "plugins": [
    {
      "plugin": "@nxlv/python",
      "options": {
        "packageManager": "uv"
      }
    }
  ],
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/**/?(*.)+(spec|test).[jt]s?(x)",
      "!{projectRoot}/test/**/*",
      "!{projectRoot}/**/*.md"
    ],
    "sharedGlobals": [
      "{workspaceRoot}/nx.json",
      "{workspaceRoot}/uv.lock",
      "{workspaceRoot}/package-lock.json"
    ]
  },
  "targetDefaults": {
    "build": {
      "cache": true,
      "inputs": ["production", "^production"],
      "outputs": ["{projectRoot}/dist", "{projectRoot}/build"]
    },
    "test": {
      "cache": true,
      "inputs": ["default", "^production"],
      "outputs": ["{projectRoot}/coverage", "{projectRoot}/.pytest_cache"]
    },
    "lint": {
      "cache": true,
      "inputs": [
        "default",
        "{workspaceRoot}/.ruff.toml",
        "{workspaceRoot}/ruff.toml"
      ]
    },
    "format": {
      "cache": false
    },
    "e2e": {
      "cache": false
    },
    "@nx/vite:build": {
      "cache": true,
      "inputs": ["production", "^production"],
      "outputs": ["{projectRoot}/dist"]
    },
    "@nx/vite:test": {
      "cache": true,
      "inputs": ["default", "^production"]
    }
  },
  "release": {
    "projects": ["libs/python/*", "libs/frontend/*"],
    "projectsRelationship": "independent",
    "version": {
      "conventionalCommits": true
    },
    "changelog": {
      "projectChangelogs": {
        "createRelease": "gitlab"
      }
    },
    "git": {
      "commitMessage": "chore(release): publish {version}",
      "tagPattern": "{projectName}@{version}"
    }
  }
}
```

---

## 2. NX ב-GitLab CI/CD — יסודות

### `.gitlab-ci.yml` בסיסי עם NX

```yaml
# .gitlab-ci.yml

# ========================
# משתנים גלובליים
# ========================
variables:
  # השתמשו בתמונת Docker מותאמת שלכם (ראו פרק 14)
  CI_IMAGE: "registry.company.internal/ci/nx-uv-ruff:latest"

  # ספר ל-NX שאנחנו ב-CI
  CI: "true"
  NX_DAEMON: "false"          # בסביבת CI עדיף בלי daemon
  NX_VERBOSE_LOGGING: "false"

  # UV configuration
  UV_CACHE_DIR: ".uv-cache"
  UV_LINK_MODE: "copy"        # חובה ב-Docker — אין hard links
  UV_PYTHON_DOWNLOADS: "never" # אל תוריד Python — השתמש בזה שב-image
  UV_SYSTEM_PYTHON: "0"

  # Cache key base
  CACHE_KEY_BASE: "v1"

# ========================
# הגדרת stages
# ========================
stages:
  - validate
  - affected
  - build
  - test
  - lint
  - publish
  - release

# ========================
# Cache Templates
# ========================
.node-cache: &node-cache
  cache:
    - key:
        prefix: "${CACHE_KEY_BASE}-node"
        files:
          - package-lock.json
      paths:
        - node_modules/
        - .npm/
      policy: pull-push
      when: always

.uv-cache: &uv-cache
  cache:
    - key:
        prefix: "${CACHE_KEY_BASE}-uv"
        files:
          - uv.lock
      paths:
        - $UV_CACHE_DIR
      policy: pull-push
      when: always

.nx-cache: &nx-cache
  cache:
    - key:
        prefix: "${CACHE_KEY_BASE}-nx"
        files:
          - nx.json
          - package-lock.json
      paths:
        - .nx/cache/
      policy: pull-push
      when: always

# ========================
# Job Template בסיסי
# ========================
.base:
  image: $CI_IMAGE
  interruptible: true
  before_script:
    # חישוב base/head commits עבור NX affected
    - export NX_HEAD=$CI_COMMIT_SHA
    - |
      if [ -n "$CI_MERGE_REQUEST_DIFF_BASE_SHA" ]; then
        export NX_BASE=$CI_MERGE_REQUEST_DIFF_BASE_SHA
      else
        export NX_BASE=${CI_COMMIT_BEFORE_SHA:-HEAD~1}
      fi
    - echo "NX_BASE=$NX_BASE, NX_HEAD=$NX_HEAD"
    # התקנת node dependencies
    - npm ci --cache .npm --prefer-offline
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
    - if: '$CI_COMMIT_BRANCH == "develop"'

# ========================
# Job: אימות workspace
# ========================
validate:workspace:
  extends: .base
  stage: validate
  <<: *node-cache
  script:
    - npx nx format:check --base=$NX_BASE --head=$NX_HEAD
    - npx nx graph --file=graph.json
  artifacts:
    paths:
      - graph.json
    expire_in: 1 day

# ========================
# Job: הרצת כל הבדיקות על הפרוייקטים המושפעים
# ========================
affected:lint-test-build:
  extends: .base
  stage: affected
  cache:
    - <<: *node-cache
    - <<: *uv-cache
    - <<: *nx-cache
  script:
    - npx nx affected -t lint test build --base=$NX_BASE --head=$NX_HEAD --parallel=3
  after_script:
    # Self-healing CI (רק עם NX Cloud)
    # - npx nx fix-ci
    - echo "Pipeline completed"
  artifacts:
    paths:
      - "**/.pytest_cache/"
      - "**/coverage/"
      - "**/dist/"
    reports:
      junit: "**/test-results.xml"
    expire_in: 7 days
```

---

## 3. NX Affected — בניה חכמה של רק מה שהשתנה

### כיצד NX Affected עובד

NX מנתח את ה-Git diff ומשווה אותו עם ה-**Project Graph** שלו. הוא מוצא את כל הפרויקטים שהשתנו — ואת כל הפרויקטים שמושפעים ממנו דרך תלויות.

```
שינוי ב: libs/python/shared-utils
    ↓
NX מזהה שמושפעים:
  - apps/api-service (תלוי ב-shared-utils)
  - apps/data-pipeline (תלוי ב-shared-utils)
  - libs/python/auth-helpers (תלוי ב-shared-utils)
```

### משתני סביבה קריטיים ב-GitLab

| משתנה GitLab | מה הוא מכיל | שימוש ב-NX |
|---|---|---|
| `CI_COMMIT_SHA` | ה-commit הנוכחי | `NX_HEAD` |
| `CI_MERGE_REQUEST_DIFF_BASE_SHA` | ה-base של ה-MR | `NX_BASE` בסביבת MR |
| `CI_COMMIT_BEFORE_SHA` | ה-commit לפני ה-push | `NX_BASE` בסביבת branch push |
| `CI_DEFAULT_BRANCH` | branch ראשי (main) | fallback ל-NX_BASE |

### Script מלא לחישוב NX_BASE/NX_HEAD

```bash
#!/bin/bash
# tools/scripts/compute-nx-refs.sh
# להשתמש ב-before_script של כל job

set -e

export NX_HEAD="$CI_COMMIT_SHA"

if [ -n "$CI_MERGE_REQUEST_DIFF_BASE_SHA" ]; then
  # Merge Request pipeline
  export NX_BASE="$CI_MERGE_REQUEST_DIFF_BASE_SHA"
  echo "MR pipeline: NX_BASE=$NX_BASE"
elif [ "$CI_COMMIT_BEFORE_SHA" = "0000000000000000000000000000000000000000" ]; then
  # Branch חדש — השווה עם ה-default branch
  export NX_BASE="$(git merge-base HEAD origin/$CI_DEFAULT_BRANCH || echo HEAD~1)"
  echo "New branch: NX_BASE=$NX_BASE"
elif [ -n "$CI_COMMIT_BEFORE_SHA" ]; then
  # Push רגיל
  export NX_BASE="$CI_COMMIT_BEFORE_SHA"
  echo "Branch push: NX_BASE=$NX_BASE"
else
  # Fallback
  export NX_BASE="HEAD~1"
  echo "Fallback: NX_BASE=$NX_BASE"
fi

echo "Final: NX_BASE=$NX_BASE, NX_HEAD=$NX_HEAD"
```

### בדיקת affected בזמן פיתוח

```bash
# הצג אילו פרויקטים מושפעים מהשינויים שלך
npx nx affected:graph

# הצג רשימה בלבד
npx nx show projects --affected --base=main --head=HEAD

# הרץ build רק על המושפעים
npx nx affected -t build --base=main --head=HEAD

# הרץ מספר tasks במקביל
npx nx affected -t lint test build --parallel=3 --base=main --head=HEAD
```

---

## 4. Dynamic Child Pipelines עם NX Affected

### הגישה: Pipeline דינמי המבוסס על NX Affected

GitLab תומכת ב-"Dynamic Child Pipelines" — יצירת קובץ `.gitlab-ci.yml` בזמן ריצה ואז הפעלתו כ-child pipeline. זה מאפשר ליצור jobs בדיוק לפרויקטים המושפעים.

### `.gitlab-ci.yml` ראשי — שלב יצירת ה-Pipeline הדינמי

```yaml
# .gitlab-ci.yml (חלק מה-pipeline הראשי)

stages:
  - generate
  - trigger

# ========================
# שלב 1: יצירת ה-YAML הדינמי
# ========================
generate:child-pipeline:
  stage: generate
  image: $CI_IMAGE
  before_script:
    - npm ci --cache .npm --prefer-offline
    - export NX_HEAD=$CI_COMMIT_SHA
    - |
      if [ -n "$CI_MERGE_REQUEST_DIFF_BASE_SHA" ]; then
        export NX_BASE=$CI_MERGE_REQUEST_DIFF_BASE_SHA
      else
        export NX_BASE=${CI_COMMIT_BEFORE_SHA:-HEAD~1}
      fi
  script:
    - node tools/scripts/generate-child-pipeline.js
  artifacts:
    paths:
      - dynamic-child-pipeline.yml
    expire_in: 1 hour
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

# ========================
# שלב 2: הפעלת ה-Pipeline הדינמי
# ========================
trigger:child-pipeline:
  stage: trigger
  trigger:
    include:
      - artifact: dynamic-child-pipeline.yml
        job: generate:child-pipeline
    strategy: depend
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'
```

### הסקריפט שיוצר את ה-Pipeline הדינמי

**`tools/scripts/generate-child-pipeline.js`:**

```javascript
#!/usr/bin/env node
/**
 * יוצר GitLab CI YAML דינמי בהתבסס על nx affected
 *
 * השימוש: node tools/scripts/generate-child-pipeline.js
 * פלט: dynamic-child-pipeline.yml
 */

const { execSync } = require('child_process');
const fs = require('fs');
const yaml = require('js-yaml'); // npm i js-yaml -D

const NX_BASE = process.env.NX_BASE || 'HEAD~1';
const NX_HEAD = process.env.NX_HEAD || 'HEAD';
const CI_IMAGE = process.env.CI_IMAGE || 'registry.company.internal/ci/nx-uv-ruff:latest';

// ========================
// 1. קבלת הפרויקטים המושפעים מ-NX
// ========================
function getAffectedProjects(target) {
  try {
    const result = execSync(
      `npx nx show projects --affected --base=${NX_BASE} --head=${NX_HEAD} --withTarget=${target}`,
      { encoding: 'utf8', stdio: ['pipe', 'pipe', 'pipe'] }
    ).trim();

    if (!result) return [];
    return result.split('\n').filter(Boolean);
  } catch (e) {
    console.warn(`Could not get affected projects for target ${target}:`, e.message);
    return [];
  }
}

// ========================
// 2. קבלת metadata על פרויקט
// ========================
function getProjectType(projectName) {
  try {
    const info = JSON.parse(
      execSync(`npx nx show project ${projectName} --json`, { encoding: 'utf8' })
    );
    return info.projectType || 'library'; // application / library
  } catch {
    return 'library';
  }
}

// ========================
// 3. יצירת job לפרויקט Python
// ========================
function createPythonJob(projectName, target, stage) {
  return {
    stage,
    image: CI_IMAGE,
    interruptible: true,
    variables: {
      UV_CACHE_DIR: '.uv-cache',
      UV_LINK_MODE: 'copy',
    },
    cache: [
      {
        key: { prefix: 'v1-node', files: ['package-lock.json'] },
        paths: ['node_modules/', '.npm/'],
        policy: 'pull',
      },
      {
        key: { prefix: 'v1-uv', files: ['uv.lock'] },
        paths: ['.uv-cache'],
        policy: 'pull',
      },
      {
        key: { prefix: 'v1-nx', files: ['nx.json', 'package-lock.json'] },
        paths: ['.nx/cache/'],
        policy: 'pull-push',
      },
    ],
    before_script: [
      'npm ci --cache .npm --prefer-offline',
    ],
    script: [
      `npx nx run ${projectName}:${target}`,
    ],
    artifacts: target === 'build' ? {
      paths: [`${getProjectRoot(projectName)}/dist`],
      expire_in: '7 days',
    } : undefined,
  };
}

// ========================
// 4. יצירת job לפרויקט Frontend (Vue.js)
// ========================
function createFrontendJob(projectName, target, stage) {
  return {
    stage,
    image: CI_IMAGE,
    interruptible: true,
    cache: [
      {
        key: { prefix: 'v1-node', files: ['package-lock.json'] },
        paths: ['node_modules/', '.npm/'],
        policy: 'pull',
      },
      {
        key: { prefix: 'v1-nx', files: ['nx.json', 'package-lock.json'] },
        paths: ['.nx/cache/'],
        policy: 'pull-push',
      },
    ],
    before_script: [
      'npm ci --cache .npm --prefer-offline',
    ],
    script: [
      `npx nx run ${projectName}:${target}`,
    ],
    artifacts: target === 'build' ? {
      paths: [`${getProjectRoot(projectName)}/dist`],
      expire_in: '7 days',
    } : undefined,
  };
}

function getProjectRoot(projectName) {
  try {
    const info = JSON.parse(
      execSync(`npx nx show project ${projectName} --json`, { encoding: 'utf8' })
    );
    return info.root || projectName;
  } catch {
    return projectName;
  }
}

// ========================
// 5. בניית ה-Pipeline הדינמי
// ========================
function generatePipeline() {
  console.log(`Generating dynamic pipeline for NX_BASE=${NX_BASE}, NX_HEAD=${NX_HEAD}`);

  const affectedBuild = getAffectedProjects('build');
  const affectedTest = getAffectedProjects('test');
  const affectedLint = getAffectedProjects('lint');

  console.log(`Affected build: ${affectedBuild.join(', ') || 'none'}`);
  console.log(`Affected test: ${affectedTest.join(', ') || 'none'}`);
  console.log(`Affected lint: ${affectedLint.join(', ') || 'none'}`);

  // אם לא מושפע שום דבר — צור pipeline ריק עם job placeholder
  const allAffected = new Set([...affectedBuild, ...affectedTest, ...affectedLint]);

  if (allAffected.size === 0) {
    const emptyPipeline = {
      stages: ['noop'],
      'nothing-affected': {
        stage: 'noop',
        image: 'alpine:latest',
        script: ['echo "No affected projects found"'],
      },
    };
    fs.writeFileSync('dynamic-child-pipeline.yml', yaml.dump(emptyPipeline));
    console.log('No affected projects. Generated empty pipeline.');
    return;
  }

  const pipeline = {
    stages: ['lint', 'build', 'test'],
    variables: {
      CI_IMAGE,
      UV_CACHE_DIR: '.uv-cache',
      UV_LINK_MODE: 'copy',
    },
  };

  // יצירת jobs לכל פרויקט מושפע
  for (const project of affectedLint) {
    const jobKey = `lint:${project}`.replace(/[^a-zA-Z0-9:_-]/g, '-');
    pipeline[jobKey] = createPythonJob(project, 'lint', 'lint');
  }

  for (const project of affectedBuild) {
    const jobKey = `build:${project}`.replace(/[^a-zA-Z0-9:_-]/g, '-');
    pipeline[jobKey] = createPythonJob(project, 'build', 'build');
  }

  for (const project of affectedTest) {
    const jobKey = `test:${project}`.replace(/[^a-zA-Z0-9:_-]/g, '-');
    const job = createPythonJob(project, 'test', 'test');
    // test תלוי ב-build
    if (affectedBuild.includes(project)) {
      job.needs = [`build:${project}`.replace(/[^a-zA-Z0-9:_-]/g, '-')];
    }
    pipeline[jobKey] = job;
  }

  const yamlContent = yaml.dump(pipeline, { lineWidth: 120 });
  fs.writeFileSync('dynamic-child-pipeline.yml', yamlContent);
  console.log(`Generated dynamic-child-pipeline.yml with ${Object.keys(pipeline).length - 2} jobs`);
}

generatePipeline();
```

### גישה חלופית: `nx-gitlab-ci-filter-affected`

כלי CLI מוכן שמפשט את התהליך:

```yaml
# .gitlab-ci.yml

generate:affected-pipeline:
  stage: generate
  image: node:20-alpine
  script:
    - npm ci --cache .npm --prefer-offline
    - |
      export NX_HEAD=$CI_COMMIT_SHA
      export NX_BASE=${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}
    # הכלי קורא pipeline.yml ומייצר affected.yml
    - npx nx-gitlab-ci-filter-affected --input .ci/pipeline-template.yml --output affected-pipeline.yml
  artifacts:
    paths:
      - affected-pipeline.yml
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

## 5. NX Remote Cache — ללא NX Cloud בסביבה סגורה

### אפשרויות ל-Remote Cache ב-Corporate Environment

בסביבה ארגונית ללא גישה לאינטרנט, יש כמה אפשרויות:

#### אפשרות A: MinIO כ-S3-Compatible Storage (מומלץ)

MinIO הוא S3-compatible object storage שאפשר להריץ **on-premises** ולחבר ל-`@nx/s3-cache`.

**התקנת MinIO (Docker Compose לסביבת הארגון):**

```yaml
# docker-compose.minio.yml
version: '3.8'
services:
  minio:
    image: minio/minio:latest   # יש לשמור image זה ב-registry הפנימי
    ports:
      - "9000:9000"
      - "9001:9001"
    volumes:
      - minio-data:/data
    environment:
      MINIO_ROOT_USER: nx-cache-admin
      MINIO_ROOT_PASSWORD: "CHANGE_ME_STRONG_PASSWORD"
    command: server /data --console-address ":9001"
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:9000/minio/health/live"]
      interval: 30s
      timeout: 20s
      retries: 3

volumes:
  minio-data:
```

**`nx.json` — קונפיגורציית `@nx/s3-cache` עם MinIO:**

```json
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

**`package.json` — הוספת ה-plugin:**

```json
{
  "nx": {
    "includedScripts": []
  },
  "devDependencies": {
    "@nx/s3-cache": "^21.0.0"
  }
}
```

**`.gitlab-ci.yml` — משתני סביבה:**

```yaml
variables:
  # MinIO credentials — שמרו אלה ב-GitLab CI Variables (masked)
  AWS_ACCESS_KEY_ID: "$MINIO_ACCESS_KEY"          # הגדירו ב-GitLab CI Settings
  AWS_SECRET_ACCESS_KEY: "$MINIO_SECRET_KEY"      # הגדירו ב-GitLab CI Settings

  # הפעלת NX Key (נדרש לאחר Nx 20.8)
  NX_KEY: "$NX_ACTIVATION_KEY"                    # הגדירו ב-GitLab CI Settings
```

**אזהרת אבטחה — CVE-2025-36852 (CREEP):**
> Cache poisoning vulnerability קיים ב-bucket-based caches. MR שנפתח על-ידי עובד יכול לכתוב cache זדוני שישפיע על production builds. הגבילו write access ל-CI בלבד (לא ל-MR pipelines).

```yaml
# הגנה: read-only ב-MRs, read-write ב-main
nx-cache-config:
  variables:
    NX_CACHE_MODE: >-
      ${{ CI_PIPELINE_SOURCE == 'merge_request_event' ? 'read-only' : 'read-write' }}
```

הגדירו בהתאם ב-`nx.json`:
```json
{
  "s3": {
    "localMode": "read-only",
    "ciMode": "read-write"
  }
}
```

#### אפשרות B: `portable-nx-cache` — ב-GitLab CI Filesystem Cache

פתרון זה מנצל את ה-GitLab CI cache הרגיל ומריץ שרת cache מקומי:

```yaml
# .gitlab-ci.yml

.nx-portable-cache:
  variables:
    PORTABLE_NX_CACHE_PORT: "8080"
    PORTABLE_NX_CACHE_TOKEN: "$NX_CACHE_TOKEN"   # סוד ב-GitLab Variables
    NX_SELF_HOSTED_REMOTE_CACHE_SERVER: "http://localhost:8080"
    NX_SELF_HOSTED_REMOTE_CACHE_ACCESS_TOKEN: "$NX_CACHE_TOKEN"
  cache:
    - key:
        prefix: "v1-portable-nx-cache"
        files:
          - nx.json
          - package-lock.json
          - uv.lock
      paths:
        - .portable-nx-cache/
      policy: pull-push
  before_script:
    # הורדת ה-binary (מ-registry פנימי בסביבה סגורה)
    - curl -fsSL https://artifactory.company.internal/tools/portable-nx-cache-linux-amd64 -o /tmp/portable-nx-cache
    - chmod +x /tmp/portable-nx-cache
    # הפעלת השרת ברקע
    - |
      PORTABLE_NX_CACHE_DIR=.portable-nx-cache \
      PORTABLE_NX_CACHE_PORT=$PORTABLE_NX_CACHE_PORT \
      PORTABLE_NX_CACHE_TOKEN=$PORTABLE_NX_CACHE_TOKEN \
      /tmp/portable-nx-cache &
    # המתנה שהשרת יעלה
    - |
      until curl --output /dev/null --silent --head --fail http://localhost:$PORTABLE_NX_CACHE_PORT/ready; do
        echo "Waiting for cache server..."
        sleep 1
      done
    - echo "Cache server is ready"
```

---

## 6. NX Distributed Task Execution (Agents) ב-GitLab CI

### ארכיטקטורת DTE

NX DTE מחלק tasks בין מספר runners במקביל. בסביבת NX Cloud ניתן להשתמש ב-`nx-cloud start-agent`, ובסביבה ללא NX Cloud יש לממש זאת ידנית.

### `.gitlab-ci.yml` עם DTE (דורש NX Cloud או Self-Hosted NX Cloud)

```yaml
# .gitlab-ci.yml — DTE עם NX Cloud

image: $CI_IMAGE

variables:
  NX_CLOUD_ACCESS_TOKEN: "$NX_CLOUD_TOKEN"  # ב-GitLab CI Variables
  CI: "true"

stages:
  - affected

# ========================
# Template לכל ה-DTE agents
# ========================
.dte-agent:
  interruptible: true
  stage: affected
  cache:
    - key:
        prefix: "v1-node"
        files:
          - package-lock.json
      paths:
        - node_modules/
        - .npm/
      policy: pull
    - key:
        prefix: "v1-uv"
        files:
          - uv.lock
      paths:
        - .uv-cache/
      policy: pull
  before_script:
    - npm ci --cache .npm --prefer-offline
  script:
    - export NX_AGENT_NAME=$CI_JOB_ID
    - npx nx-cloud start-agent
  after_script:
    - export NX_AGENT_NAME=$CI_JOB_ID
    # אם משתמשים ב-NX Cloud — Self-Healing CI
    # - npx nx fix-ci

# ========================
# Template לאורכסטרטור
# ========================
.base-pipeline:
  interruptible: true
  stage: affected
  only:
    - main
    - merge_requests
  cache:
    - key:
        prefix: "v1-node"
        files:
          - package-lock.json
      paths:
        - node_modules/
        - .npm/
      policy: pull-push
    - key:
        prefix: "v1-uv"
        files:
          - uv.lock
      paths:
        - .uv-cache/
      policy: pull-push
  before_script:
    - npm ci --cache .npm --prefer-offline
    - export NX_HEAD=$CI_COMMIT_SHA
    - |
      if [ -n "$CI_MERGE_REQUEST_DIFF_BASE_SHA" ]; then
        export NX_BASE=$CI_MERGE_REQUEST_DIFF_BASE_SHA
      else
        export NX_BASE=${CI_COMMIT_BEFORE_SHA:-HEAD~1}
      fi
  artifacts:
    expire_in: 5 days
    paths:
      - dist/

# ========================
# Job ראשי — Orchestrator
# ========================
nx-dte:
  extends: .base-pipeline
  script:
    # התחלת DTE run — distribute על 3 agents
    - npx nx-cloud start-ci-run --distribute-on="3 linux-medium-js" --stop-agents-after="build"
    # הרצת format check (לא מחולק)
    - npx nx-cloud record -- npx nx format:check --base=$NX_BASE --head=$NX_HEAD
    # הרצת tasks על המושפעים (מחולק בין agents)
    - npx nx affected --base=$NX_BASE --head=$NX_HEAD -t lint test build --parallel=2
  after_script:
    # Self-healing CI
    - npx nx fix-ci

# ========================
# Agents — הוסיפו כמה שצריך
# ========================
nx-dte-agent-1:
  extends: .dte-agent

nx-dte-agent-2:
  extends: .dte-agent

nx-dte-agent-3:
  extends: .dte-agent
```

### DTE ללא NX Cloud (Manual Distribution)

בסביבה ללא NX Cloud, ניתן לחלק ידנית:

```yaml
# חלוקה ידנית לפי project type

build:python-affected:
  extends: .base
  script:
    - npx nx affected -t build --base=$NX_BASE --head=$NX_HEAD --projects='libs/python/*,apps/api-*'

build:frontend-affected:
  extends: .base
  script:
    - npx nx affected -t build --base=$NX_BASE --head=$NX_HEAD --projects='apps/frontend-*,libs/frontend/*'

test:python-affected:
  extends: .base
  needs: [build:python-affected]
  script:
    - npx nx affected -t test --base=$NX_BASE --head=$NX_HEAD --projects='libs/python/*,apps/api-*'
```

---

## 7. UV ב-GitLab CI/CD — ניהול תלויות Python

### קונפיגורציית UV בסיסית ב-GitLab CI

```yaml
# .gitlab-ci.yml — UV jobs

variables:
  UV_CACHE_DIR: ".uv-cache"
  UV_LINK_MODE: "copy"        # חובה — Docker יוצר mountpoint נפרד
  UV_PYTHON_DOWNLOADS: "never" # אל תוריד Python בזמן CI
  UV_SYSTEM_PYTHON: "0"        # אל תשתמש ב-system Python

  # אם ה-private registry דורש auth:
  UV_INDEX_CORPORATE_PYPI_USERNAME: "gitlab-ci-token"
  UV_INDEX_CORPORATE_PYPI_PASSWORD: "$CI_JOB_TOKEN"

# ========================
# Cache Template ל-UV
# ========================
.uv-cache-full: &uv-cache-full
  cache:
    key:
      prefix: "v1-uv-${CI_JOB_NAME}"
      files:
        - uv.lock
    paths:
      - $UV_CACHE_DIR
    policy: pull-push
    when: always
    fallback_keys:
      - "v1-uv-"

# ========================
# Job: התקנת תלויות
# ========================
python:install:
  stage: .pre
  image: ghcr.io/astral-sh/uv:0.7.0-python3.12-bookworm-slim
  <<: *uv-cache-full
  script:
    - uv sync --frozen --all-packages
  after_script:
    - uv cache prune --ci  # הקטנת ה-cache

# ========================
# Job: הרצת tests
# ========================
python:test:
  stage: test
  image: ghcr.io/astral-sh/uv:0.7.0-python3.12-bookworm-slim
  <<: *uv-cache-full
  script:
    - uv run --frozen pytest apps/api-service/tests/ -v --junitxml=test-results.xml
  artifacts:
    reports:
      junit: test-results.xml
    expire_in: 7 days

# ========================
# Job: build של package ספציפי
# ========================
python:build:
  stage: build
  image: ghcr.io/astral-sh/uv:0.7.0-python3.12-bookworm-slim
  <<: *uv-cache-full
  script:
    - uv build --package my-library --out-dir dist/
  artifacts:
    paths:
      - dist/*.whl
      - dist/*.tar.gz
    expire_in: 7 days
```

### שימוש ב-UV עם Python גרסאות שונות

```yaml
# Matrix build על גרסאות Python שונות
python:test-matrix:
  stage: test
  parallel:
    matrix:
      - PYTHON_VERSION: ["3.11", "3.12", "3.13"]
  image: "ghcr.io/astral-sh/uv:latest-python${PYTHON_VERSION}-bookworm-slim"
  script:
    - uv run --python $PYTHON_VERSION pytest tests/
```

### UV Cache Optimization

```yaml
# אסטרטגיית Cache מתקדמת ל-UV

# Job שרק ממלא cache (pull-push) — ירוץ אחת ל-push ל-main
uv:warm-cache:
  stage: .pre
  image: ghcr.io/astral-sh/uv:latest-python3.12-bookworm-slim
  cache:
    key:
      prefix: "v1-uv-warm"
      files:
        - uv.lock
    paths:
      - .uv-cache/
    policy: push  # רק כותב, לא קורא
  script:
    - uv sync --frozen --all-packages
    - uv cache prune --ci
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# Jobs אחרים רק קוראים מה-cache
python:test:
  cache:
    key:
      prefix: "v1-uv-warm"
      files:
        - uv.lock
    paths:
      - .uv-cache/
    policy: pull  # רק קורא
```

---

## 8. UV Workspaces — Python Monorepo אמיתי

### מבנה `pyproject.toml` לכל member

**`apps/api-service/pyproject.toml`:**

```toml
[project]
name = "api-service"
version = "0.1.0"
description = "Company API Service"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.34.0",
    "pydantic>=2.10.0",
    # תלות פנימית מה-workspace
    "shared-utils",
    "db-models",
    "auth-helpers",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-asyncio>=0.24.0",
    "pytest-cov>=6.0.0",
    "httpx>=0.28.0",  # לבדיקות HTTP
]

[tool.uv.sources]
# אלה יסופקו מה-workspace
shared-utils = { workspace = true }
db-models = { workspace = true }
auth-helpers = { workspace = true }

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/api_service"]
```

**`libs/python/shared-utils/pyproject.toml`:**

```toml
[project]
name = "shared-utils"
version = "1.2.0"
description = "Shared utilities for company Python projects"
requires-python = ">=3.11"
dependencies = [
    "python-dateutil>=2.9.0",
    "structlog>=24.0.0",
    "pydantic>=2.10.0",
]

[project.optional-dependencies]
dev = [
    "pytest>=8.0.0",
    "pytest-cov>=6.0.0",
    "ruff>=0.9.0",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/shared_utils"]

[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "UP", "B", "SIM"]
ignore = ["E501"]
```

### פקודות UV Workspace שימושיות

```bash
# התקנת כל ה-workspace
uv sync

# התקנת package ספציפי (כולל תלויות שלו מה-workspace)
uv sync --package api-service

# הרצת command בתוך package ספציפי
uv run --package api-service python -m pytest

# הוספת תלות חיצונית ל-package ספציפי
uv add httpx --package api-service

# הוספת תלות פנימית מה-workspace
uv add shared-utils --package api-service

# Build של package ספציפי
uv build --package shared-utils --out-dir dist/

# עדכון lockfile לכל ה-workspace
uv lock

# בדיקת lockfile בלי לשנות
uv lock --check
```

---

## 9. RUFF ב-GitLab CI/CD — Linting ו-Formatting

### קונפיגורציית Ruff ב-`pyproject.toml` / `ruff.toml`

**`ruff.toml` (בשורש ה-monorepo):**

```toml
# ruff.toml

# גרסת Python
target-version = "py311"

# אורך שורה
line-length = 100

# תיקיות להתעלם מהן
exclude = [
    ".git",
    ".nx",
    ".uv-cache",
    "node_modules",
    "dist",
    "build",
    "__pycache__",
    "*.egg-info",
    "migrations",     # אם משתמשים ב-Alembic
]

[lint]
# כללים מופעלים
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "SIM",  # flake8-simplify
    "C4",   # flake8-comprehensions
    "DTZ",  # flake8-datetimez
    "T20",  # flake8-print
    "ANN",  # flake8-annotations (type hints)
]

# כללים מושתקים
ignore = [
    "E501",   # line too long (מטופל על-ידי formatter)
    "ANN101", # self type annotation
    "ANN102", # cls type annotation
    "T201",   # print statements (יש projects שצריכים)
]

# אפשר auto-fix בכלים אלה
fixable = ["I", "UP", "C4"]

[lint.per-file-ignores]
"tests/**/*.py" = ["ANN", "S"]
"**/__init__.py" = ["F401"]   # unused imports בקבצי __init__

[lint.isort]
known-first-party = ["shared_utils", "db_models", "auth_helpers"]

[format]
# סגנון quotes
quote-style = "double"
# indent
indent-style = "space"
indent-width = 4
```

### `.gitlab-ci.yml` — RUFF Jobs

```yaml
# .ci/python-lint.gitlab-ci.yml

# ========================
# RUFF Lint — עם GitLab Code Quality Report
# ========================
ruff:lint:
  stage: lint
  image: ghcr.io/astral-sh/ruff:0.9.10-alpine
  interruptible: true
  variables:
    GIT_STRATEGY: fetch
  before_script:
    - ruff --version
  script:
    # הרצת linter עם פורמט GitLab
    - |
      ruff check \
        --output-format=gitlab \
        --output-file=$CI_PROJECT_DIR/ruff-code-quality.json \
        .
  artifacts:
    reports:
      codequality: $CI_PROJECT_DIR/ruff-code-quality.json
    paths:
      - ruff-code-quality.json
    expire_in: 7 days
    when: always  # שמור גם אם הJob נכשל
  allow_failure: false  # שנו ל-true להתחלה
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

# ========================
# RUFF Format Check
# ========================
ruff:format-check:
  stage: lint
  image: ghcr.io/astral-sh/ruff:0.9.10-alpine
  interruptible: true
  script:
    - ruff format --check --diff .
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

# ========================
# RUFF על Affected בלבד (עם NX)
# ========================
ruff:lint-affected:
  stage: lint
  image: $CI_IMAGE
  interruptible: true
  before_script:
    - npm ci --cache .npm --prefer-offline
    - export NX_HEAD=$CI_COMMIT_SHA
    - export NX_BASE=${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}
  script:
    # מצא את הפרויקטים המושפעים
    - AFFECTED=$(npx nx show projects --affected --base=$NX_BASE --head=$NX_HEAD --withTarget=lint 2>/dev/null || echo "")
    - |
      if [ -z "$AFFECTED" ]; then
        echo "No affected Python projects to lint"
        exit 0
      fi
    - echo "Running ruff on affected projects: $AFFECTED"
    # הרץ lint על כל project מושפע
    - |
      for project in $AFFECTED; do
        PROJECT_ROOT=$(npx nx show project $project --json | python3 -c "import sys,json; print(json.load(sys.stdin)['root'])")
        echo "Linting $project ($PROJECT_ROOT)..."
        ruff check --output-format=gitlab "$PROJECT_ROOT" >> $CI_PROJECT_DIR/ruff-code-quality.json || true
      done
  artifacts:
    reports:
      codequality: $CI_PROJECT_DIR/ruff-code-quality.json
    when: always
```

### שילוב RUFF עם הפורמטר המותאם שלכם

```yaml
# ========================
# הפורמטר המותאם
# ========================
format:custom:
  stage: lint
  image: $CI_IMAGE
  interruptible: true
  script:
    # הריצו את הפורמטר המותאם שלכם
    - uv run python tools/scripts/format_code.py --check .
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# ========================
# Job שמחיל את כל הפורמטים
# (ידני — לא ב-MR)
# ========================
format:apply:
  stage: lint
  image: $CI_IMAGE
  script:
    - ruff format .
    - ruff check --fix .
    - uv run python tools/scripts/format_code.py .
    - |
      git config user.email "ci@company.internal"
      git config user.name "GitLab CI"
      git add -A
      git diff --cached --exit-code || git commit -m "style: auto-format code"
      git push "https://gitlab-ci-token:${CI_JOB_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git" HEAD:${CI_COMMIT_REF_NAME}
  when: manual
  rules:
    - if: '$CI_COMMIT_BRANCH != "main"'
```

---

## 10. @nxlv/python — הגשר בין NX ל-Python/UV

### התקנה והגדרה

```bash
# התקנת ה-plugin
npm install @nxlv/python --save-dev

# הוספה ל-nx.json
```

**`nx.json` עם @nxlv/python:**

```json
{
  "plugins": [
    {
      "plugin": "@nxlv/python",
      "options": {
        "packageManager": "uv",
        "autoActivate": true
      }
    }
  ],
  "sync": {
    "globalGenerators": ["@nxlv/python:pkg-sync"]
  }
}
```

### יצירת Python Project חדש

```bash
# יצירת library חדש
npx nx generate @nxlv/python:uv-project \
  --name=my-new-lib \
  --directory=libs/python/my-new-lib \
  --projectType=library \
  --linter=ruff \
  --unitTestRunner=pytest \
  --publishable \
  --buildBundleLocalDependencies=true

# יצירת application חדש
npx nx generate @nxlv/python:uv-project \
  --name=my-new-app \
  --directory=apps/my-new-app \
  --projectType=application \
  --linter=ruff \
  --unitTestRunner=pytest
```

### `project.json` לפרויקט Python עם @nxlv/python

**`libs/python/shared-utils/project.json`:**

```json
{
  "$schema": "../../../../node_modules/nx/schemas/project-schema.json",
  "name": "shared-utils",
  "projectType": "library",
  "sourceRoot": "libs/python/shared-utils/src",
  "targets": {
    "lint": {
      "executor": "@nxlv/python:ruff",
      "options": {
        "lintFilePatterns": ["libs/python/shared-utils"]
      },
      "cache": true,
      "inputs": [
        "default",
        "{workspaceRoot}/ruff.toml",
        "{workspaceRoot}/.ruff.toml"
      ]
    },
    "test": {
      "executor": "@nxlv/python:run-commands",
      "options": {
        "command": "uv run --frozen pytest src/ tests/ -v --cov=src --cov-report=xml:coverage.xml --junitxml=test-results.xml",
        "cwd": "libs/python/shared-utils"
      },
      "cache": true,
      "inputs": ["default", "^production"],
      "outputs": [
        "{projectRoot}/coverage.xml",
        "{projectRoot}/test-results.xml",
        "{projectRoot}/.pytest_cache"
      ]
    },
    "build": {
      "executor": "@nxlv/python:build",
      "options": {
        "outputPath": "libs/python/shared-utils/dist",
        "publish": false,
        "lockedVersions": true,
        "bundleLocalDependencies": true
      },
      "cache": true,
      "inputs": ["production", "^production"],
      "outputs": ["{projectRoot}/dist"]
    },
    "publish": {
      "executor": "@nxlv/python:publish",
      "options": {
        "buildTarget": "shared-utils:build",
        "registry": "https://gitlab.company.internal/api/v4/projects/{PROJECT_ID}/packages/pypi"
      },
      "dependsOn": ["build"]
    },
    "format": {
      "executor": "@nxlv/python:ruff",
      "options": {
        "lintFilePatterns": ["libs/python/shared-utils"],
        "format": true
      },
      "cache": false
    }
  }
}
```

### Implicit Dependencies — NX מזהה תלויות Python אוטומטית

החל מגרסה 21.3.0 של @nxlv/python, ניתן להפעיל `inferDependencies` שסורק imports ומגלה תלויות:

```json
{
  "plugins": [
    {
      "plugin": "@nxlv/python",
      "options": {
        "packageManager": "uv",
        "inferDependencies": true
      }
    }
  ]
}
```

כעת אם `api-service` עושה `from shared_utils import something`, NX יבין אוטומטית שיש תלות ויכלול `shared-utils` ב-affected כאשר הוא משתנה.

---

## 11. GitLab Package Registry כ-Private PyPI

### קונפיגורציית GitLab Package Registry

#### הגדרת הפניה ב-UV

**`pyproject.toml` (שורש ה-monorepo):**

```toml
[[tool.uv.index]]
name = "corporate-pypi"
url = "https://gitlab.company.internal/api/v4/groups/MY_GROUP_ID/-/packages/pypi/simple"
default = true

# חשוב: אל תשמרו credentials כאן בגלל אבטחה
# אלא השתמשו ב-environment variables
```

**`uv.toml` (אלטרנטיבה):**

```toml
[[index]]
name = "corporate-pypi"
url = "https://gitlab.company.internal/api/v4/groups/MY_GROUP_ID/-/packages/pypi/simple"
default = true
# authenticate = "always"  # תמיד שלח credentials
```

#### Authentication ב-UV

**Authentication עם Environment Variables (מומלץ):**

```bash
# ב-GitLab CI — מוגדר כ-CI Variables (masked)
export UV_INDEX_CORPORATE_PYPI_USERNAME="gitlab-ci-token"
export UV_INDEX_CORPORATE_PYPI_PASSWORD="$CI_JOB_TOKEN"
```

**Authentication עם `.netrc`:**

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

#### העלאת Packages ל-GitLab Registry

```yaml
# .gitlab-ci.yml

publish:python-package:
  stage: publish
  image: $CI_IMAGE
  script:
    # Build
    - uv build --package $PACKAGE_NAME --out-dir dist/
    # Upload — UV תומך ב-twine
    - |
      uv run twine upload \
        --repository-url "${CI_API_V4_URL}/projects/${CI_PROJECT_ID}/packages/pypi" \
        --username "gitlab-ci-token" \
        --password "${CI_JOB_TOKEN}" \
        dist/*.whl dist/*.tar.gz
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

#### מניעת PyPI Forwarding (חשוב לסביבות סגורות)

ב-GitLab Admin:
1. עברו ל-**Group Settings > Packages and registries > Package registry**
2. כבו את **Forward PyPI package requests to PyPI.org**

זה קריטי! ללא זה, כאשר package לא נמצא ב-registry הפנימי, GitLab יעביר את הבקשה ל-PyPI הציבורי — דבר שאסור ברשת סגורה.

#### pip.conf לחיבור ל-Registry

**`pip.conf` (לפיתוח מקומי):**

```ini
[global]
index-url = https://gitlab-ci-token:MYTOKEN@gitlab.company.internal/api/v4/groups/MY_GROUP_ID/-/packages/pypi/simple
no-index = false
trusted-host = gitlab.company.internal
```

**`~/.config/uv/uv.toml` (לפיתוח מקומי):**

```toml
[[index]]
name = "corporate-pypi"
url = "https://gitlab.company.internal/api/v4/groups/MY_GROUP_ID/-/packages/pypi/simple"
default = true
```

---

## 12. Vue.js עם NX ב-GitLab CI

### הגדרת Vue.js Project ב-NX

**`apps/frontend-dashboard/project.json`:**

```json
{
  "$schema": "../../node_modules/nx/schemas/project-schema.json",
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
      "configurations": {
        "production": {
          "mode": "production"
        },
        "development": {
          "mode": "development"
        }
      },
      "cache": true,
      "inputs": ["production", "^production"],
      "outputs": ["{projectRoot}/dist"]
    },
    "serve": {
      "executor": "@nx/vite:dev-server",
      "options": {
        "buildTarget": "frontend-dashboard:build"
      },
      "cache": false
    },
    "test": {
      "executor": "@nx/vite:test",
      "options": {
        "config": "apps/frontend-dashboard/vite.config.ts",
        "reportsDirectory": "apps/frontend-dashboard/coverage"
      },
      "cache": true,
      "inputs": ["default", "^production"],
      "outputs": ["{projectRoot}/coverage"]
    },
    "lint": {
      "executor": "@nx/eslint:lint",
      "options": {
        "lintFilePatterns": ["apps/frontend-dashboard/**/*.{ts,vue}"]
      },
      "cache": true
    }
  }
}
```

### `.gitlab-ci.yml` ל-Vue.js עם NX Affected

```yaml
# .ci/frontend-jobs.gitlab-ci.yml

# ========================
# Build Vue.js apps מושפעות
# ========================
frontend:build-affected:
  stage: build
  image: $CI_IMAGE
  interruptible: true
  cache:
    - key:
        prefix: "v1-node"
        files:
          - package-lock.json
      paths:
        - node_modules/
        - .npm/
      policy: pull
    - key:
        prefix: "v1-nx"
        files:
          - nx.json
          - package-lock.json
      paths:
        - .nx/cache/
      policy: pull-push
  before_script:
    - npm ci --cache .npm --prefer-offline
    - export NX_HEAD=$CI_COMMIT_SHA
    - export NX_BASE=${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}
  script:
    - npx nx affected -t build --base=$NX_BASE --head=$NX_HEAD --projects='apps/frontend-*,libs/frontend/*' --parallel=2
  artifacts:
    paths:
      - dist/apps/
    expire_in: 7 days
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH == "main"'

# ========================
# Test Vue.js components מושפעים
# ========================
frontend:test-affected:
  stage: test
  image: $CI_IMAGE
  interruptible: true
  needs: [frontend:build-affected]
  cache:
    - key:
        prefix: "v1-node"
        files:
          - package-lock.json
      paths:
        - node_modules/
        - .npm/
      policy: pull
    - key:
        prefix: "v1-nx"
        files:
          - nx.json
          - package-lock.json
      paths:
        - .nx/cache/
      policy: pull-push
  before_script:
    - npm ci --cache .npm --prefer-offline
    - export NX_HEAD=$CI_COMMIT_SHA
    - export NX_BASE=${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}
  script:
    - npx nx affected -t test --base=$NX_BASE --head=$NX_HEAD --projects='apps/frontend-*,libs/frontend/*'
  artifacts:
    reports:
      junit: "**/test-results.xml"
      coverage_report:
        coverage_format: cobertura
        path: "**/coverage/cobertura-coverage.xml"
    expire_in: 7 days

# ========================
# Deploy frontend (דוגמה)
# ========================
frontend:deploy-affected:
  stage: publish
  image: $CI_IMAGE
  needs: [frontend:build-affected, frontend:test-affected]
  before_script:
    - npm ci --cache .npm --prefer-offline
    - export NX_HEAD=$CI_COMMIT_SHA
    - export NX_BASE=${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}
  script:
    # מצא apps מושפעות
    - |
      AFFECTED_APPS=$(npx nx show projects --affected --base=$NX_BASE --head=$NX_HEAD \
        --withTarget=build --projects='apps/frontend-*' 2>/dev/null || echo "")
    - |
      for app in $AFFECTED_APPS; do
        echo "Deploying $app..."
        # לוגיקת ה-deploy שלכם (Kubernetes, S3, Nginx, etc.)
      done
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
```

---

## 13. Release Automation עם NX ו-@jscutlery/semver

### קונפיגורציית @jscutlery/semver

**התקנה:**

```bash
npm install -D @jscutlery/semver
npx nx generate @jscutlery/semver:install
```

**הגדרת versioning target ב-`project.json`:**

```json
{
  "targets": {
    "version": {
      "executor": "@jscutlery/semver:version",
      "options": {
        "preset": "conventionalcommits",
        "tagPrefix": "{projectName}@",
        "commitMessageFormat": "chore({projectName}): release version {version}",
        "push": true,
        "enforceBumpPolicy": "all",
        "trackDeps": true,
        "postTargets": [
          "shared-utils:build",
          "shared-utils:publish",
          "shared-utils:gitlab-release"
        ]
      }
    },
    "gitlab-release": {
      "executor": "@jscutlery/semver:gitlab",
      "options": {
        "tag": "{tag}",
        "ref": "main",
        "releaseNotes": "{notes}"
      }
    }
  }
}
```

### `.gitlab-ci.yml` ל-Release

```yaml
# .ci/release-jobs.gitlab-ci.yml

# ========================
# Release — manual trigger ל-main
# ========================
release:version:
  stage: release
  image: $CI_IMAGE
  interruptible: false
  variables:
    # חובה: token עם הרשאות push ו-create release
    GITLAB_TOKEN: "$SEMANTIC_RELEASE_TOKEN"  # PAT עם api scope
    GIT_AUTHOR_EMAIL: "ci-release@company.internal"
    GIT_AUTHOR_NAME: "GitLab CI Release Bot"
  before_script:
    - npm ci --cache .npm --prefer-offline
    # הגדרת git identity
    - git config user.email "$GIT_AUTHOR_EMAIL"
    - git config user.name "$GIT_AUTHOR_NAME"
    # הגדרת remote עם token
    - git remote set-url origin "https://gitlab-ci-token:${CI_JOB_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git"
  script:
    # Version + Changelog + Tag לכל library מושפעת
    - npx nx affected -t version --base=$(git describe --tags --abbrev=0 || echo "HEAD~10") --head=HEAD
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual

# ========================
# Release עם NX Release (חלופה)
# ========================
release:nx-release:
  stage: release
  image: $CI_IMAGE
  variables:
    GITLAB_TOKEN: "$SEMANTIC_RELEASE_TOKEN"
  before_script:
    - npm ci --cache .npm --prefer-offline
    - git config user.email "ci@company.internal"
    - git config user.name "CI Release Bot"
    - git remote set-url origin "https://ci-release:${SEMANTIC_RELEASE_TOKEN}@${CI_SERVER_HOST}/${CI_PROJECT_PATH}.git"
  script:
    # Dry run לבדיקה
    - npx nx release --dry-run
    # Release אמיתי (uncomment בפרודקשן)
    # - npx nx release
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
      when: manual
  allow_failure: false

# ========================
# Publish Python packages אחרי Release
# ========================
publish:python-after-release:
  stage: release
  image: $CI_IMAGE
  needs: [release:nx-release]
  script:
    # מצא packages עם גרסאות חדשות
    - |
      CHANGED_PACKAGES=$(git diff HEAD~1 HEAD --name-only | \
        grep 'pyproject.toml' | \
        xargs -I{} dirname {} | \
        xargs -I{} sh -c 'echo $(basename {})')
    - |
      for pkg in $CHANGED_PACKAGES; do
        echo "Building and publishing $pkg..."
        uv build --package $pkg --out-dir dist/$pkg/
        uv run twine upload \
          --repository-url "${CI_API_V4_URL}/groups/MY_GROUP_ID/packages/pypi" \
          --username "gitlab-ci-token" \
          --password "${CI_JOB_TOKEN}" \
          dist/$pkg/*.whl dist/$pkg/*.tar.gz
      done
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

### Conventional Commits — כיצד עובד

```
# גורם ל-patch version bump (1.0.0 → 1.0.1)
fix(shared-utils): handle None values in date parser

# גורם ל-minor version bump (1.0.0 → 1.1.0)
feat(db-models): add new User.last_login field

# גורם ל-major version bump (1.0.0 → 2.0.0)
feat(auth-helpers)!: rename authenticate() to verify_token()

BREAKING CHANGE: The authenticate() function has been renamed

# לא גורם לשינוי גרסה
chore: update dependencies
docs: improve README
test: add unit tests for edge cases
```

---

## 14. Docker Images מותאמות ל-CI: NX + UV + RUFF

### Dockerfile לתמונת CI מותאמת

```dockerfile
# docker/ci-runner/Dockerfile
# תמונת CI המשלבת: Node.js (לNX) + UV + RUFF + Python

# ========================
# שלב 1: Base image
# ========================
FROM ubuntu:22.04 AS base

ENV DEBIAN_FRONTEND=noninteractive \
    PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1

RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    git \
    ca-certificates \
    build-essential \
    libssl-dev \
    libffi-dev \
    && rm -rf /var/lib/apt/lists/*

# ========================
# שלב 2: Node.js (לNX)
# ========================
FROM base AS node-stage

ARG NODE_VERSION=20.18.0

# בסביבה סגורה — הורידו מ-mirror פנימי
RUN curl -fsSL "https://nodejs.org/dist/v${NODE_VERSION}/node-v${NODE_VERSION}-linux-x64.tar.gz" \
    | tar -xz -C /usr/local --strip-components=1

# ========================
# שלב 3: Python + UV
# ========================
FROM node-stage AS python-stage

ARG PYTHON_VERSION=3.12.9
ARG UV_VERSION=0.7.0

# התקנת Python
RUN curl -fsSL "https://www.python.org/ftp/python/${PYTHON_VERSION}/Python-${PYTHON_VERSION}.tgz" \
    | tar -xz -C /tmp && \
    cd /tmp/Python-${PYTHON_VERSION} && \
    ./configure --enable-optimizations --with-lto && \
    make -j$(nproc) && \
    make install && \
    rm -rf /tmp/Python-${PYTHON_VERSION}

# symlinks
RUN ln -sf /usr/local/bin/python3 /usr/local/bin/python && \
    ln -sf /usr/local/bin/pip3 /usr/local/bin/pip

# התקנת UV
RUN curl -LsSf "https://github.com/astral-sh/uv/releases/download/${UV_VERSION}/uv-installer.sh" | sh
ENV PATH="/root/.cargo/bin:/root/.local/bin:$PATH"

# אלטרנטיבה — מ-pip (יותר פשוט לסביבות סגורות)
# RUN pip install uv==${UV_VERSION}

# ========================
# שלב 4: RUFF
# ========================
FROM python-stage AS ruff-stage

ARG RUFF_VERSION=0.9.10

# התקנת RUFF
RUN pip install ruff==${RUFF_VERSION}

# או מ-binary ישיר:
# RUN curl -LsSf "https://github.com/astral-sh/ruff/releases/download/v${RUFF_VERSION}/ruff-x86_64-unknown-linux-musl.tar.gz" \
#     | tar -xz -C /usr/local/bin

# ========================
# שלב 5: כלים נוספים
# ========================
FROM ruff-stage AS tools-stage

# twine לפרסום packages
RUN pip install twine

# kubectl, helm, aws-cli וכו' לפי הצורך
# RUN ...

# ========================
# תמונה סופית
# ========================
FROM tools-stage AS final

# הגדרות UV
ENV UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never \
    UV_CACHE_DIR=/tmp/.uv-cache

# הגדרות NX
ENV NX_DAEMON=false

# בדיקות גרסאות
RUN node --version && \
    npm --version && \
    python --version && \
    uv --version && \
    ruff --version

WORKDIR /workspace

CMD ["bash"]
```

### Dockerfile מהיר יותר — Multi-stage עם UV base image

```dockerfile
# docker/ci-runner/Dockerfile.uv-base
# גישה מהירה יותר — בנויה על UV's official image

FROM ghcr.io/astral-sh/uv:0.7.0-python3.12-bookworm-slim AS uv-base

# הוספת Node.js
RUN apt-get update && apt-get install -y --no-install-recommends \
    curl \
    git \
    ca-certificates \
    && rm -rf /var/lib/apt/lists/*

ARG NODE_VERSION=20
RUN curl -fsSL https://deb.nodesource.com/setup_${NODE_VERSION}.x | bash - && \
    apt-get install -y nodejs && \
    rm -rf /var/lib/apt/lists/*

# RUFF (כבר מותקן ב-UV image אם יש uv tool install, אחרת:)
RUN pip install ruff twine

ENV UV_LINK_MODE=copy \
    UV_PYTHON_DOWNLOADS=never \
    NX_DAEMON=false

WORKDIR /workspace

RUN node --version && uv --version && ruff --version
```

### Build ו-Push לרגיסטרי הפנימי

```yaml
# .ci/build-ci-image.gitlab-ci.yml

build:ci-docker-image:
  stage: validate
  image: docker:24
  services:
    - docker:24-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
    CI_IMAGE_TAG: "${CI_REGISTRY_IMAGE}/ci-runner:${CI_COMMIT_SHORT_SHA}"
  script:
    - docker login -u "$CI_REGISTRY_USER" -p "$CI_REGISTRY_PASSWORD" "$CI_REGISTRY"
    - docker build -t "$CI_IMAGE_TAG" -t "${CI_REGISTRY_IMAGE}/ci-runner:latest" docker/ci-runner/
    - docker push "$CI_IMAGE_TAG"
    - docker push "${CI_REGISTRY_IMAGE}/ci-runner:latest"
  rules:
    # בנה image חדש רק כאשר ה-Dockerfile משתנה
    - changes:
        - docker/ci-runner/Dockerfile
        - docker/ci-runner/Dockerfile.uv-base
      if: '$CI_COMMIT_BRANCH == "main"'
```

---

## 15. סביבה ארגונית סגורה — שיקולים וקונפיגורציות מיוחדות

### ארכיטקטורת הסביבה הסגורה

```
╔══════════════════════════════════════════════════════════════╗
║                   Corporate Network                          ║
║                                                              ║
║  ┌─────────────────┐    ┌──────────────────────────────┐    ║
║  │   GitLab         │    │   Internal Registries        │    ║
║  │   Self-Hosted    │    │                              │    ║
║  │                  │    │  ┌──────────────────────┐    │    ║
║  │  - Source Code   │    │  │ Docker Registry      │    │    ║
║  │  - CI Pipelines  │◄──►│  │ (GitLab Container    │    │    ║
║  │  - Package Reg.  │    │  │  Registry or Harbor) │    │    ║
║  │  - NX Cache (?)  │    │  └──────────────────────┘    │    ║
║  └─────────────────┘    │  ┌──────────────────────┐    │    ║
║                          │  │ PyPI Mirror          │    │    ║
║  ┌─────────────────┐    │  │ (GitLab Package Reg  │    │    ║
║  │   MinIO         │    │  │  or Devpi/Nexus)     │    │    ║
║  │   (NX Cache)    │    │  └──────────────────────┘    │    ║
║  └─────────────────┘    │  ┌──────────────────────┐    │    ║
║                          │  │ npm Mirror           │    │    ║
║  ┌─────────────────┐    │  │ (Verdaccio/Nexus)    │    │    ║
║  │   GitLab        │    │  └──────────────────────┘    │    ║
║  │   Runners       │    └──────────────────────────────┘    ║
║  │   (On-Prem)     │                                         ║
║  └─────────────────┘                                         ║
╚══════════════════════════════════════════════════════════════╝
```

### הגדרת npm Registry פנימי (Verdaccio)

```yaml
# docker-compose.verdaccio.yml
version: '3.8'
services:
  verdaccio:
    image: verdaccio/verdaccio:5
    ports:
      - "4873:4873"
    volumes:
      - verdaccio-storage:/verdaccio/storage
      - ./verdaccio-config:/verdaccio/conf
    environment:
      VERDACCIO_PORT: "4873"

volumes:
  verdaccio-storage:
```

**`verdaccio-config/config.yaml`:**

```yaml
storage: /verdaccio/storage/data
auth:
  htpasswd:
    file: /verdaccio/storage/htpasswd
    max_users: -1  # unlimited

uplinks:
  npmjs:
    url: https://registry.npmjs.org/
    timeout: 30s
    max_fails: 2
    fail_timeout: 5m

packages:
  '@mycompany/*':
    access: $authenticated
    publish: $authenticated
    unpublish: $authenticated

  '**':
    access: $anonymous
    publish: $authenticated
    proxy: npmjs

server:
  keepAliveTimeout: 60

middlewares:
  audit:
    enabled: true

log: {type: stdout, format: pretty, level: http}
```

**.npmrc בשורש המונורפו (לסביבה הסגורה):**

```
registry=http://verdaccio.company.internal:4873/
//verdaccio.company.internal:4873/:_authToken=${NPM_TOKEN}
```

**`.gitlab-ci.yml` — שימוש ב-npm registry פנימי:**

```yaml
variables:
  NPM_CONFIG_REGISTRY: "http://verdaccio.company.internal:4873/"
  NPM_TOKEN: "$VERDACCIO_TOKEN"  # CI Variable

before_script:
  - |
    echo "//verdaccio.company.internal:4873/:_authToken=${NPM_TOKEN}" >> ~/.npmrc
  - npm ci --cache .npm --prefer-offline
```

### הגדרת GitLab Runner לסביבה סגורה

**`/etc/gitlab-runner/config.toml`:**

```toml
concurrent = 4
check_interval = 0

[session_server]
  session_timeout = 1800

[[runners]]
  name = "corporate-runner"
  url = "https://gitlab.company.internal"
  token = "RUNNER_TOKEN"
  executor = "docker"

  [runners.docker]
    tls_verify = false
    image = "registry.company.internal/ci/nx-uv-ruff:latest"
    privileged = false
    disable_entrypoint_overwrite = false
    oom_kill_disable = false
    disable_cache = false

    # Registry פנימי בלבד
    pull_policy = ["if-not-present", "never"]
    # זה מונע ניסיון להוריד images מהאינטרנט

    allowed_images = [
      "registry.company.internal/*",
      "node:*",           # אם יש mirror פנימי
    ]

    volumes = [
      "/cache",
      "/var/run/docker.sock:/var/run/docker.sock"
    ]

    shm_size = 0

    [runners.docker.dns]
    # DNS פנימי
    dns = ["10.0.0.53"]

  [runners.cache]
    Type = "s3"
    Path = "gitlab-runner-cache"
    Shared = true

    [runners.cache.s3]
      ServerAddress = "minio.company.internal:9000"
      AccessKey = "RUNNER_CACHE_ACCESS_KEY"
      SecretKey = "RUNNER_CACHE_SECRET_KEY"
      BucketName = "runner-cache"
      Insecure = false  # השתמשו ב-true רק ל-dev
```

### Mirror של Docker Images לסביבה סגורה

**סקריפט לסנכרון images נדרשים:**

```bash
#!/bin/bash
# tools/scripts/sync-docker-images.sh
# הרץ על מכונה עם גישה לאינטרנט, אחר כך העבר

INTERNAL_REGISTRY="registry.company.internal"
IMAGES=(
  "ghcr.io/astral-sh/uv:0.7.0-python3.12-bookworm-slim"
  "ghcr.io/astral-sh/ruff:0.9.10-alpine"
  "node:20-alpine"
  "node:20-bookworm-slim"
  "alpine:3.19"
  "minio/minio:latest"
  "verdaccio/verdaccio:5"
)

for image in "${IMAGES[@]}"; do
  # חלץ שם הimage ללא registry prefix
  name=$(echo $image | sed 's|.*/||')
  tag=$(echo $name | cut -d':' -f2)
  short_name=$(echo $name | cut -d':' -f1)

  echo "Syncing $image -> $INTERNAL_REGISTRY/mirrors/$short_name:$tag"

  docker pull "$image"
  docker tag "$image" "$INTERNAL_REGISTRY/mirrors/$short_name:$tag"
  docker push "$INTERNAL_REGISTRY/mirrors/$short_name:$tag"
done
```

### אחסון binaries של UV, RUFF ב-Artifactory/GitLab

בסביבה סגורה, אתם צריכים לאחסן את ה-binaries של UV ו-RUFF ב-registry פנימי:

```yaml
# .gitlab-ci.yml — הורדת binaries מ-registry פנימי
variables:
  UV_VERSION: "0.7.0"
  RUFF_VERSION: "0.9.10"
  INTERNAL_TOOLS_URL: "https://artifactory.company.internal/tools"

install:tools:
  stage: .pre
  script:
    # UV
    - |
      curl -fsSL "${INTERNAL_TOOLS_URL}/uv/${UV_VERSION}/uv-linux-x64.tar.gz" \
        | tar -xz -C /usr/local/bin
    # RUFF
    - |
      curl -fsSL "${INTERNAL_TOOLS_URL}/ruff/${RUFF_VERSION}/ruff-linux-x64" \
        -o /usr/local/bin/ruff && chmod +x /usr/local/bin/ruff
    # אחסן ב-artifacts לשימוש ב-jobs הבאים
    - mkdir -p tools-bin && cp /usr/local/bin/uv /usr/local/bin/ruff tools-bin/
  artifacts:
    paths:
      - tools-bin/
    expire_in: 1 day
```

---

## 16. אסטרטגיית Caching מלאה ל-NX + UV ב-GitLab

### שכבות Cache ב-GitLab CI

| שכבה | מה נשמר | key | policy |
|---|---|---|---|
| npm/node_modules | תלויות JavaScript לNX | `hash(package-lock.json)` | pull-push |
| .uv-cache | packages Python שהורדו | `hash(uv.lock)` | pull-push |
| .nx/cache | תוצאות tasks מ-NX | `hash(nx.json + package-lock.json)` | pull-push |
| .portable-nx-cache | NX remote cache (filesystem) | `hash(nx.json + uv.lock)` | pull-push |
| dist/ | artifacts של build | artifact (לא cache) | — |

### `.gitlab-ci.yml` — קונפיגורציית Cache מלאה ומיטבית

```yaml
# קונפיגורציית cache מלאה ל-monorepo

variables:
  CACHE_VERSION: "v3"  # שנו לאיפוס כל ה-caches
  UV_CACHE_DIR: ".uv-cache"

# ========================
# Cache anchors
# ========================

# node_modules — פעולות NX
.cache-node: &cache-node
  key:
    prefix: "${CACHE_VERSION}-node-${CI_JOB_NAME}"
    files:
      - package-lock.json
  paths:
    - node_modules/
    - .npm/
  policy: pull-push
  when: always
  fallback_keys:
    - "${CACHE_VERSION}-node-"

# UV packages cache
.cache-uv: &cache-uv
  key:
    prefix: "${CACHE_VERSION}-uv"
    files:
      - uv.lock
  paths:
    - $UV_CACHE_DIR
  policy: pull-push
  when: always
  fallback_keys:
    - "${CACHE_VERSION}-uv-"

# NX computation cache
.cache-nx: &cache-nx
  key:
    prefix: "${CACHE_VERSION}-nx"
    files:
      - nx.json
      - package-lock.json
      - uv.lock
  paths:
    - .nx/cache/
  policy: pull-push
  when: always
  fallback_keys:
    - "${CACHE_VERSION}-nx-"

# ========================
# Job שממלא cache (ל-main branch)
# ========================
cache:warm-all:
  stage: validate
  image: $CI_IMAGE
  cache:
    - <<: *cache-node
      policy: push
    - <<: *cache-uv
      policy: push
    - <<: *cache-nx
      policy: push
  script:
    - npm ci --cache .npm --prefer-offline
    - uv sync --frozen --all-packages
    - uv cache prune --ci
    - echo "Cache warmed successfully"
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'

# ========================
# Job בדיקה רגיל — קורא cache
# ========================
python:test:
  stage: test
  image: $CI_IMAGE
  cache:
    - <<: *cache-node
      policy: pull      # רק קריאה, מהיר יותר
    - <<: *cache-uv
      policy: pull
    - <<: *cache-nx
      policy: pull-push # NX cache מתעדכן
  before_script:
    - npm ci --cache .npm --prefer-offline
  script:
    - |
      export NX_HEAD=$CI_COMMIT_SHA
      export NX_BASE=${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}
    - npx nx affected -t test --base=$NX_BASE --head=$NX_HEAD

# ========================
# ניקוי cache (ידני)
# ========================
cache:clear:
  stage: validate
  image: alpine:3.19
  script:
    - echo "Cache will be cleared by changing CACHE_VERSION variable"
    - echo "Current CACHE_VERSION: $CACHE_VERSION"
  when: manual
```

### אסטרטגיית cache לתהליכי CI שונים

```yaml
# אסטרטגיה שונה ל-MR vs main

# MR — רק קריאה מ-cache (מהיר, לא מזהם)
.mr-cache:
  cache:
    - key:
        prefix: "${CACHE_VERSION}-node"
        files: [package-lock.json]
      paths: [node_modules/, .npm/]
      policy: pull
    - key:
        prefix: "${CACHE_VERSION}-uv"
        files: [uv.lock]
      paths: [$UV_CACHE_DIR]
      policy: pull
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'

# main — קריאה וכתיבה (מעדכן cache)
.main-cache:
  cache:
    - key:
        prefix: "${CACHE_VERSION}-node"
        files: [package-lock.json]
      paths: [node_modules/, .npm/]
      policy: pull-push
    - key:
        prefix: "${CACHE_VERSION}-uv"
        files: [uv.lock]
      paths: [$UV_CACHE_DIR]
      policy: pull-push
  rules:
    - if: '$CI_COMMIT_BRANCH == "main"'
```

---

## 17. Template Library ב-GitLab CI לפרויקט Monorepo

### מבנה ה-Template Library

```yaml
# .gitlab-ci.yml — ה-pipeline הראשי שמשלב הכל

include:
  - local: .ci/variables.gitlab-ci.yml
  - local: .ci/cache-config.gitlab-ci.yml
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

# Pipeline-level variables
variables:
  CI_IMAGE: "registry.company.internal/ci/nx-uv-ruff:latest"
  CACHE_VERSION: "v4"
  UV_CACHE_DIR: ".uv-cache"
  UV_LINK_MODE: "copy"
  UV_PYTHON_DOWNLOADS: "never"
  NX_DAEMON: "false"
```

**`.ci/variables.gitlab-ci.yml`:**

```yaml
# .ci/variables.gitlab-ci.yml
# משתנים גלובליים

variables:
  # ========================
  # CI Image
  # ========================
  CI_IMAGE: "registry.company.internal/ci/nx-uv-ruff:latest"

  # ========================
  # UV
  # ========================
  UV_CACHE_DIR: ".uv-cache"
  UV_LINK_MODE: "copy"
  UV_PYTHON_DOWNLOADS: "never"
  UV_INDEX_CORPORATE_PYPI_USERNAME: "gitlab-ci-token"
  UV_INDEX_CORPORATE_PYPI_PASSWORD: "$CI_JOB_TOKEN"

  # ========================
  # NX
  # ========================
  NX_DAEMON: "false"
  NX_VERBOSE_LOGGING: "false"
  # NX_KEY: "$NX_ACTIVATION_KEY"  # נדרש לself-hosted cache

  # ========================
  # Cache
  # ========================
  CACHE_VERSION: "v4"

  # ========================
  # GitLab
  # ========================
  GIT_DEPTH: "50"  # מינימום commits לanalysis
  GIT_STRATEGY: "fetch"
```

**`.ci/python-jobs.gitlab-ci.yml`:**

```yaml
# .ci/python-jobs.gitlab-ci.yml

# Template בסיסי לכל Python jobs
.python-base:
  image: $CI_IMAGE
  interruptible: true
  before_script:
    - npm ci --cache .npm --prefer-offline --quiet
    - export NX_HEAD=$CI_COMMIT_SHA
    - |
      if [ -n "$CI_MERGE_REQUEST_DIFF_BASE_SHA" ]; then
        export NX_BASE=$CI_MERGE_REQUEST_DIFF_BASE_SHA
      else
        export NX_BASE=${CI_COMMIT_BEFORE_SHA:-HEAD~1}
      fi
    - echo "NX: base=$NX_BASE head=$NX_HEAD"

# Lint Python
python:lint:
  extends: .python-base
  stage: lint
  cache:
    - key:
        prefix: "${CACHE_VERSION}-node"
        files: [package-lock.json]
      paths: [node_modules/, .npm/]
      policy: pull
    - key:
        prefix: "${CACHE_VERSION}-nx"
        files: [nx.json, package-lock.json]
      paths: [.nx/cache/]
      policy: pull-push
  script:
    - npx nx affected -t lint --base=$NX_BASE --head=$NX_HEAD --parallel=4
  artifacts:
    reports:
      codequality: ruff-code-quality.json
    when: always
    expire_in: 7 days
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

# Test Python
python:test:
  extends: .python-base
  stage: test
  cache:
    - key:
        prefix: "${CACHE_VERSION}-node"
        files: [package-lock.json]
      paths: [node_modules/, .npm/]
      policy: pull
    - key:
        prefix: "${CACHE_VERSION}-uv"
        files: [uv.lock]
      paths: [$UV_CACHE_DIR]
      policy: pull
    - key:
        prefix: "${CACHE_VERSION}-nx"
        files: [nx.json, package-lock.json, uv.lock]
      paths: [.nx/cache/]
      policy: pull-push
  script:
    - npx nx affected -t test --base=$NX_BASE --head=$NX_HEAD --parallel=2
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    reports:
      junit: "**/test-results.xml"
      coverage_report:
        coverage_format: cobertura
        path: "**/coverage.xml"
    expire_in: 7 days
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'

# Build Python packages
python:build:
  extends: .python-base
  stage: build
  cache:
    - key:
        prefix: "${CACHE_VERSION}-node"
        files: [package-lock.json]
      paths: [node_modules/, .npm/]
      policy: pull
    - key:
        prefix: "${CACHE_VERSION}-uv"
        files: [uv.lock]
      paths: [$UV_CACHE_DIR]
      policy: pull
    - key:
        prefix: "${CACHE_VERSION}-nx"
        files: [nx.json, package-lock.json, uv.lock]
      paths: [.nx/cache/]
      policy: pull-push
  script:
    - npx nx affected -t build --base=$NX_BASE --head=$NX_HEAD
  artifacts:
    paths:
      - "**/dist/*.whl"
      - "**/dist/*.tar.gz"
    expire_in: 7 days
  rules:
    - if: '$CI_COMMIT_BRANCH =~ /^(main|develop)$/'
```

---

## 18. בעיות נפוצות, פערים ואזהרות

### בעיות NX

#### בעיה 1: NX_BASE שגוי בסביבת CI

**תסמין**: NX מריץ את כל הפרויקטים גם כשמשהו קטן השתנה.

**פתרון**: וודאו שה-git history מספיק עמוק:

```yaml
variables:
  GIT_DEPTH: "0"  # Clone מלא (איטי) — אם יש בעיות
  # או
  GIT_DEPTH: "100"  # 100 commits לאחור
```

#### בעיה 2: NX לא מוצא שינויים ב-MR

**תסמין**: `CI_MERGE_REQUEST_DIFF_BASE_SHA` ריק.

**פתרון**: הגדירו `only: merge_requests` בנכון:

```yaml
job:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
  # אל תשתמשו ב-only/except אם אתם משתמשים ב-rules
```

#### בעיה 3: NX Daemon בעיות ב-CI

**תסמין**: שגיאות connection ל-NX daemon.

**פתרון**: כבו daemon ב-CI:

```yaml
variables:
  NX_DAEMON: "false"
```

#### בעיה 4: NX cache corrupted

**תסמין**: Jobs נכשלים עם שגיאות cache.

**פתרון**: שנו את `CACHE_VERSION` ב-GitLab CI Variables כדי לאפס.

### בעיות UV

#### בעיה 5: `UV_LINK_MODE` שגיאה ב-Docker

**תסמין**: `Failed to hardlink ... to ...`

**פתרון**: הגדירו `UV_LINK_MODE: "copy"` — חובה ב-Docker כי mountpoints שונים.

```yaml
variables:
  UV_LINK_MODE: "copy"
```

#### בעיה 6: uv לא מוצא Python

**תסמין**: `error: No interpreter found for Python`

**פתרון**: ציינו Python path מפורש:

```yaml
variables:
  UV_PYTHON: "/usr/local/bin/python3.12"
  UV_PYTHON_DOWNLOADS: "never"
```

#### בעיה 7: Lock file לא מעודכן

**תסמין**: `error: The lockfile at 'uv.lock' needs to be updated`

**פתרון**: הריצו `uv lock` מקומית לפני commit:

```bash
uv lock
git add uv.lock
git commit -m "chore: update uv.lock"
```

#### בעיה 8: Authentication fails לGitLab registry

**תסמין**: `401 Unauthorized` בהורדת packages פנימיים.

**פתרון**: וודאו שה-CI Variables מוגדרים נכון:

```yaml
variables:
  UV_INDEX_CORPORATE_PYPI_USERNAME: "gitlab-ci-token"
  UV_INDEX_CORPORATE_PYPI_PASSWORD: "$CI_JOB_TOKEN"
```

### בעיות RUFF

#### בעיה 9: RUFF מדלג על קבצים

**תסמין**: RUFF לא בודק קבצים מסוימים.

**פתרון**: בדקו `exclude` ב-`ruff.toml`:

```toml
[lint]
# force-include יגרום ל-ruff לבדוק קבצים גם אם הם ב-exclude
force-include = ["libs/python/generated/**/*.py"]
```

#### בעיה 10: RUFF Code Quality report לא מופיע ב-MR

**תסמין**: ה-codequality artifact קיים אבל לא מופיע ב-MR.

**פתרון**: וודאו פורמט JSON תקין:

```bash
# בדיקת הפורמט
ruff check --output-format=gitlab . | python3 -m json.tool > /dev/null
```

### בעיות GitLab CI

#### בעיה 11: Dynamic child pipeline לא מופעל

**תסמין**: `trigger` job נכשל עם "artifact not found".

**פתרון**: ה-artifact חייב להיות מה-job בדיוק כפי שהוא מוגדר ב-`include`:

```yaml
trigger:child:
  trigger:
    include:
      - artifact: dynamic-child-pipeline.yml
        job: generate:child-pipeline  # חייב להתאים בדיוק!
```

#### בעיה 12: Cache לא נשמר בין pipelines

**תסמין**: כל pipeline מתחיל מאפס.

**פתרון**:
1. וודאו שה-runner מוגדר עם `shared_cache` או שהוא אותו runner
2. הגדירו cache ב-runner level (S3 backend)
3. בדקו שהkey לא משתנה לא צפוי

### פערים ידועים בכלים

| כלי | פער | עקיפה |
|---|---|---|
| NX | לא תומך ב-Python natively | @nxlv/python plugin |
| NX Release | לא תומך ב-Python packages | @nxlv/python publish + @jscutlery/semver |
| NX Release | לא תומך ב-GitLab releases natively | @jscutlery/semver:gitlab executor |
| UV | credentials ב-uv.lock — לא נשמרים | צריך לספק env vars בכל pipeline |
| UV | לא תומך ב-"affected" | @nxlv/python מטפל בזה דרך NX |
| @nxlv/python | documentation חסרה | פנו ל-GitHub issues |
| @nx/s3-cache | CVE-2025-36852 cache poisoning | הגבילו write access |

---

## 19. Checklist יישום מלא

### שלב 1: הכנת הסביבה הארגונית

- [ ] הגדרת GitLab Self-Hosted עם Container Registry פעיל
- [ ] הגדרת MinIO instance לNX remote cache (או portable-nx-cache)
- [ ] הגדרת Verdaccio/Nexus כ-npm registry פנימי
- [ ] כיבוי "Forward PyPI package requests" ב-GitLab Group settings
- [ ] אחסון Docker images נדרשים ב-registry הפנימי:
  - `ghcr.io/astral-sh/uv:*`
  - `ghcr.io/astral-sh/ruff:*`
  - `node:20-*`
  - `alpine:3.*`
- [ ] הגדרת GitLab Runner עם `pull_policy = ["if-not-present"]`

### שלב 2: אתחול ה-Monorepo

- [ ] יצירת מבנה תיקיות (apps/, libs/, tools/, .ci/)
- [ ] אתחול NX: `npx nx@latest init`
- [ ] התקנת @nxlv/python: `npm install @nxlv/python --save-dev`
- [ ] הגדרת `nx.json` עם plugin ו-targetDefaults
- [ ] יצירת root `pyproject.toml` עם UV workspace
- [ ] הגדרת private PyPI index ב-`pyproject.toml`
- [ ] יצירת `ruff.toml` בשורש

### שלב 3: הגדרת Pipeline ב-GitLab

- [ ] יצירת `.gitlab-ci.yml` ראשי
- [ ] יצירת template files ב-`.ci/`
- [ ] הגדרת CI Variables ב-GitLab:
  - `MINIO_ACCESS_KEY` (masked)
  - `MINIO_SECRET_KEY` (masked)
  - `NX_CACHE_TOKEN` (masked)
  - `NPM_TOKEN` (masked, לVerdaccio)
  - `SEMANTIC_RELEASE_TOKEN` (masked, PAT עם api scope)
- [ ] בדיקת pipeline בסיסי על branch ניסיוני

### שלב 4: הגדרת NX Affected

- [ ] בדיקת `nx show projects --affected` מקומית
- [ ] וידוא `NX_BASE`/`NX_HEAD` מחושבים נכון ב-CI
- [ ] יצירת `tools/scripts/generate-child-pipeline.js`
- [ ] בדיקת dynamic child pipeline על MR

### שלב 5: הגדרת UV Workspace

- [ ] הגדרת כל packages עם `pyproject.toml` תקין
- [ ] הרצת `uv sync` — וידוא `uv.lock` נוצר
- [ ] הגדרת authentication לGitLab registry
- [ ] בדיקת `uv build --package PACKAGE_NAME`
- [ ] בדיקת `uv run pytest` על כל package

### שלב 6: הגדרת RUFF

- [ ] יצירת `ruff.toml` מותאם לארגון
- [ ] הרצת `ruff check .` מקומית
- [ ] הגדרת Job ב-CI עם GitLab Code Quality output
- [ ] אינטגרציה עם הפורמטר המותאם שלכם
- [ ] הגדרת pre-commit hook מקומי (אופציונלי)

### שלב 7: Release Automation

- [ ] התקנת `@jscutlery/semver`
- [ ] הגדרת `version` target בכל library
- [ ] הגדרת `SEMANTIC_RELEASE_TOKEN` ב-CI
- [ ] הגדרת Conventional Commits בצוות
- [ ] בדיקת `--dry-run` לפני production

### שלב 8: אופטימיזציה ו-Production Hardening

- [ ] בדיקת זמני pipeline — ציפייה: חיסכון 60-80% בזמן
- [ ] הגדרת cache warming job ל-main branch
- [ ] הגדרת alerting על pipeline failures
- [ ] בדיקת CVE-2025-36852 — הגבלת write access ל-MR pipelines
- [ ] תיעוד פנימי לצוות

---

## סיכום טכני

| רכיב | כלי | אינטגרציית GitLab CI |
|---|---|---|
| Build Orchestration | NX | `nx affected`, dynamic pipelines |
| Python Package Manager | UV | `uv sync --frozen`, `uv build` |
| Python Linter/Formatter | RUFF | `ruff check --output-format=gitlab` |
| NX Python Integration | @nxlv/python | project.json executors |
| Remote Cache | MinIO + @nx/s3-cache | S3-compatible, on-prem |
| Release Automation | @jscutlery/semver | `nx affected -t version` |
| Private PyPI | GitLab Package Registry | `UV_INDEX_*` env vars |
| npm Mirror | Verdaccio | `.npmrc` + CI variables |
| CI Image | Custom Dockerfile | Node + UV + RUFF + Python |

השילוב של כלים אלה מספק:
- **Build חכם**: רק מה שהשתנה נבנה ונבדק
- **Cache אגרסיבי**: NX cache + UV cache + GitLab CI cache
- **Release מדויק**: גרסאות עצמאיות לכל library, changelog אוטומטי
- **סביבה סגורה**: ללא תלות בשירותים חיצוניים
- **מהירות**: 60-80% ירידה בזמן pipeline על codebase גדול

---

## מקורות

- [NX GitLab CI Configuration](https://nx.dev/ci/recipes/set-up/monorepo-ci-gitlab)
- [NX Affected Commands](https://nx.dev/docs/features/ci-features/affected)
- [NX GitLab DTE](https://nx.dev/ci/recipes/dte/gitlab-dte)
- [NX Self-Hosted Caching](https://nx.dev/docs/guides/tasks--caching/self-hosted-caching)
- [NX S3 Cache Plugin](https://nx.dev/docs/reference/remote-cache-plugins/s3-cache/overview)
- [NX Release Guide](https://nx.dev/docs/guides/nx-release)
- [UV GitLab CI Integration](https://docs.astral.sh/uv/guides/integration/gitlab/)
- [UV Package Indexes](https://docs.astral.sh/uv/concepts/indexes/)
- [UV Caching](https://docs.astral.sh/uv/concepts/cache/)
- [UV Workspaces](https://docs.astral.sh/uv/concepts/projects/workspaces/)
- [RUFF Integrations](https://docs.astral.sh/ruff/integrations/)
- [GitLab PyPI Package Registry](https://docs.gitlab.com/user/packages/pypi_repository/)
- [GitLab CI Caching](https://docs.gitlab.com/ci/caching/)
- [GitLab CI Monorepo Guide](https://about.gitlab.com/blog/building-a-gitlab-ci-cd-pipeline-for-a-monorepo-the-easy-way/)
- [GitLab Offline Environments](https://docs.gitlab.com/topics/offline/quick_start_guide/)
- [@nxlv/python Plugin](https://www.npmjs.com/package/@nxlv/python)
- [@jscutlery/semver Plugin](https://github.com/jscutlery/semver)
- [nx-gitlab-ci-filter-affected](https://github.com/jase88/nx-gitlab-ci-filter-affected)
- [portable-nx-cache](https://salvozappa.com/how-nx-pulled-the-rug-on-us.html)
- [NX Custom Cache Server](https://github.com/IKatsuba/nx-cache-server)
- [CI/CD Dockerfile for UV+RUFF](https://code.mendhak.com/ci-cd-dockerfile-for-python-uv-ruff-pytest/)
- [GitLab CI Air-Gapped Security Scanning](https://about.gitlab.com/blog/2025/02/05/tutorial-security-scanning-in-air-gapped-environments/)
