# מדריך מקיף: מודרניזציה של מונורפו Python/Vue עם NX, UV, RUFF ופורמטר מותאם

> **גרסה:** 2.0 | **תאריך:** מרץ 2026
> **סביבה:** GitLab CI/CD · רשת סגורה · Python + Vue.js · Monorepo

---

## תוכן עניינים

1. [רקע ומטרות](#1-רקע-ומטרות)
2. [מפת הפלאגינים — מה עובד ומה לא](#2-מפת-הפלאגינים--מה-עובד-ומה-לא)
3. [סקירת ארכיטקטורה](#3-סקירת-ארכיטקטורה)
4. [שלב 1 — NX + @nxlv/python: ניהול חכם של המונורפו](#4-שלב-1--nx--nxlvpython-ניהול-חכם-של-המונורפו)
5. [שלב 2 — UV: מיגרציה מ-Poetry](#5-שלב-2--uv-מיגרציה-מ-poetry)
6. [שלב 3 — RUFF: לינטינג ופורמטינג](#6-שלב-3--ruff-לינטינג-ופורמטינג)
7. [שלב 4 — פורמטר מותאם](#7-שלב-4--פורמטר-מותאם)
8. [אינטגרציה עם GitLab CI/CD](#8-אינטגרציה-עם-gitlab-cicd)
9. [עבודה ברשת סגורה](#9-עבודה-ברשת-סגורה)
10. [Release חכם עם NX ו-@nxlv/python](#10-release-חכם-עם-nx-ו-nxlvpython)
11. [מפת דרכים ותכנון מיגרציה](#11-מפת-דרכים-ותכנון-מיגרציה)
12. [בעיות ידועות וfootguns](#12-בעיות-ידועות-וfootguns)
13. [פתרון בעיות נפוצות](#13-פתרון-בעיות-נפוצות)

---

## 1. רקע ומטרות

### 1.1 המצב הנוכחי

הארגון מפעיל **מונורפו גדול** המשלב:
- עשרות ספריות Python ויישומים עם תלויות שונות בגרסאות שונות
- פרויקטים בVue.js
- Poetry כמנהל חבילות Python
- GitLab CI/CD עם pipelines קיימים
- רשת ארגונית סגורה ללא גישה ישירה לאינטרנט

### 1.2 בעיות קיימות שמובילות לשינוי

| בעיה | השפעה | פתרון מוצע |
|------|--------|------------|
| build כל המונורפו בכל PR | זמן CI ארוך, משאבים מבוזבזים | NX affected |
| Release ידני ולא מדויק | שגיאות אנושיות, גרסאות לא עקביות | NX Release + @nxlv/python |
| Poetry איטי, lockfile תלוי-פלטפורמה | CI איטי, בעיות הגדרה | UV |
| כלי לינטינג מרובים ולא אחידים | קונפיגורציה מורכבת, ביצועים ירודים | RUFF |
| פורמטינג לא אחיד בין צוותים | קוד לא עקבי, merge conflicts מיותרים | Custom Formatter + RUFF |

### 1.3 יעדי המיגרציה

1. **NX + @nxlv/python** — הפחתת זמן CI ב-70%+ על ידי build/test רק של מה שהשתנה
2. **UV** — הפחתת זמן install תלויות ב-10-100x
3. **RUFF** — לינטינג ופורמטינג מהיר, אחיד, קל לתחזוקה
4. **Custom Formatter** — אכיפת סגנון קוד ארגוני אחיד
5. כל השינויים — **תאימות מלאה עם GitLab CI/CD ורשת סגורה**

---

## 2. מפת הפלאגינים — מה עובד ומה לא

> ⚠️ **זה החלק הכי חשוב במסמך.** הבנה שגויה של מפת הפלאגינים תגרום לשבועות של עבודה לריק.

### 2.1 האמת על @nx/python

**`@nx/python` רשמי מחברת NX/Nrwl — לא קיים.**

NX היא framework שמצטיין בJS/TS. תמיכת Python היא קהילתית לחלוטין. כאשר תראו בתיעוד NX דוגמאות Python, הן מסתמכות על:

| פלאגין | מקור | מה הוא עושה |
|--------|------|-------------|
| **`@nxlv/python`** | קהילה (Lucas Vieira) | **הפלאגין המרכזי לPython** — generators, executors, UV/Poetry support, release |
| **`@nx/vue`** | רשמי (Nrwl) | Vue.js support |
| **`@nx/vite`** | רשמי (Nrwl) | Vite build/test |
| **`@jscutlery/semver`** | קהילה | Versioning ל-JS/TS בלבד — **לא Python** |

### 2.2 @nxlv/python — מה הוא מספק

`@nxlv/python` הוא הסיבה שNX + Python בכלל עובד. בלעדיו:
- NX לא יודע מה Python project
- NX לא יודע לבנות project graph מ-`pyproject.toml`
- NX לא יודע להריץ build/test/lint בצורה נכונה
- NX Release לא יודע לעדכן `pyproject.toml`

**מה הוא מספק:**

```
@nxlv/python
├── Generators (יצירת פרויקטים)
│   ├── uv-project          ← צור פרויקט Python חדש עם UV
│   ├── poetry-project      ← צור פרויקט Python חדש עם Poetry
│   └── pkg-sync            ← סנכרן imports לpyproject.toml (experimental)
│
├── Executors (הרצת targets)
│   ├── @nxlv/python:build          ← בנה wheel/sdist
│   ├── @nxlv/python:publish        ← פרסם לPyPI/private registry
│   ├── @nxlv/python:install        ← התקן תלויות
│   ├── @nxlv/python:lock           ← עדכן lockfile
│   ├── @nxlv/python:sync           ← סנכרן environment מlockfile
│   ├── @nxlv/python:ruff           ← הרץ RUFF lint
│   ├── @nxlv/python:flake8         ← הרץ flake8 (legacy)
│   └── @nxlv/python:run-commands   ← הרץ פקודה עם venv מופעל
│
├── Release Integration
│   └── @nxlv/python/release/version-actions  ← עדכן pyproject.toml בrelease
│
└── NX Sync
    └── inferDependencies: true  ← בנה project graph מpyproject.toml
```

### 2.3 מה @jscutlery/semver עושה (ולא עושה)

`@jscutlery/semver` הוא פלאגין versioning ל-JS/TS. הוא **לא** מעדכן `pyproject.toml`.

**מה הוא כן עושה:**
- Conventional Commits → SemVer bump ל-`package.json`
- יצירת CHANGELOG
- יצירת GitLab Release (יש `:gitlab` executor!)
- `trackDeps` — patch bump אוטומטי כאשר dependency מקבל גרסה

**לPython:** השתמשו ב-`@nxlv/python/release/version-actions` + NX Release המובנה.

### 2.4 ארכיטקטורת פלאגינים מלאה

```
monorepo/
├── package.json     ← @nxlv/python, @nx/vue, @nx/vite, @jscutlery/semver
├── nx.json          ← plugin config
│
│  Python projects — מנוהלים על ידי @nxlv/python
│  ├── executors: @nxlv/python:build, :ruff, :install, :publish
│  └── release: @nxlv/python/release/version-actions
│
│  Vue projects — מנוהלים על ידי @nx/vue + @nx/vite
│  ├── executors: @nx/vite:build, @nx/vite:test, @nx/vite:dev-server
│  └── release: @nx/js/src/release/version-actions (package.json)
│
│  GitLab Releases (tags + release notes)
│  └── @jscutlery/semver:gitlab (אופציונלי, ליצירת GitLab release objects)
```

---

## 3. סקירת ארכיטקטורה

### 3.1 מבנה המונורפו המוצע

```
monorepo/
├── nx.json                          ← הגדרות NX גלובליות + plugin config
├── package.json                     ← NX + Node.js כלים
├── node_modules/                    ← NX + plugins
├── pyproject.toml                   ← UV workspace root
├── uv.lock                          ← lockfile אחיד לכל הפרויקטים
├── .venv/                           ← venv משותף (כל ה-Python projects)
├── .python-version                  ← גרסת Python ברירת מחדל
├── ruff.toml                        ← RUFF גלובלי
├── .gitlab-ci.yml                   ← pipeline ראשי
│
├── packages/                        ← ספריות Python פנימיות
│   ├── shared-utils/
│   │   ├── project.json             ← NX project config
│   │   ├── pyproject.toml           ← UV member + @nxlv/python config
│   │   └── src/shared_utils/
│   ├── data-models/
│   └── ml-pipeline/
│
├── apps/                            ← יישומים Python
│   ├── api-service/
│   └── worker/
│
├── frontend/                        ← פרויקטי Vue.js
│   ├── web-app/
│   │   ├── project.json
│   │   └── src/
│   └── admin-panel/
│
└── tools/                           ← כלים פנימיים
    ├── format.py                    ← Custom Formatter
    └── scripts/
```

---

## 4. שלב 1 — NX + @nxlv/python: ניהול חכם של המונורפו

### 4.1 מהו NX?

NX הוא כלי לניהול monorepo שפותח על ידי חברת Nrwl. עבור Python, הוא פועל **יחד** עם `@nxlv/python` — כל אחד אחראי על חלק אחר:

| NX | @nxlv/python |
|----|--------------|
| Project graph engine | Python project detection |
| Affected algorithm | pyproject.toml parsing |
| Caching | build/test/lint executors |
| Release versioning engine | Python version bump logic |
| Task orchestration | UV/Poetry integration |

### 4.2 התקנה

#### דרישות מקדימות
```bash
node --version  # >= 18.0
npm --version   # >= 9.0
```

#### שלב א: אתחול NX במונורפו קיים

```bash
# בשורש המונורפו
npx nx@latest init

# תבחר: "Add NX to existing monorepo"
```

#### שלב ב: התקנת כל הפלאגינים

```bash
npm install -D \
  nx@latest \
  @nxlv/python@latest \
  @nx/vue@latest \
  @nx/vite@latest \
  @nx/eslint@latest
```

> **גרסאות נוכחיות (מרץ 2026):** `@nxlv/python@22.x`, `@nx/vue@21.x`

#### package.json בשורש

```json
{
  "name": "monorepo",
  "private": true,
  "devDependencies": {
    "nx": "^21.0.0",
    "@nxlv/python": "^22.0.0",
    "@nx/vue": "^21.0.0",
    "@nx/vite": "^21.0.0",
    "@nx/eslint": "^21.0.0"
  },
  "scripts": {
    "affected:build": "nx affected -t build",
    "affected:test": "nx affected -t test",
    "affected:lint": "nx affected -t lint",
    "graph": "nx graph",
    "sync": "nx sync"
  }
}
```

### 4.3 הגדרת nx.json

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",

  "defaultBase": "main",

  "plugins": [
    {
      "plugin": "@nxlv/python",
      "options": {
        "packageManager": "uv",
        "rootPyprojectDependencyGroup": "dev",
        "inferDependencies": true
      }
    },
    {
      "plugin": "@nx/vue/plugin",
      "options": {
        "buildTargetName": "build",
        "testTargetName": "test",
        "serveTargetName": "serve",
        "previewTargetName": "preview"
      }
    },
    {
      "plugin": "@nx/vite/plugin",
      "options": {
        "buildTargetName": "build",
        "testTargetName": "test",
        "devServerTargetName": "serve"
      }
    }
  ],

  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "sharedGlobals": [
      "{workspaceRoot}/nx.json",
      "{workspaceRoot}/pyproject.toml",
      "{workspaceRoot}/uv.lock",
      "{workspaceRoot}/ruff.toml"
    ],
    "python": [
      "{projectRoot}/**/*.py",
      "{projectRoot}/pyproject.toml",
      "!{projectRoot}/**/__pycache__/**",
      "!{projectRoot}/**/*.pyc",
      "!{projectRoot}/.pytest_cache/**"
    ],
    "tests": [
      "{projectRoot}/tests/**/*.py",
      "{projectRoot}/pyproject.toml"
    ],
    "frontend": [
      "{projectRoot}/src/**/*.{ts,vue,css,html}",
      "{projectRoot}/package.json",
      "{projectRoot}/vite.config.ts",
      "{projectRoot}/tsconfig.json"
    ]
  },

  "targetDefaults": {
    "build": {
      "cache": true,
      "dependsOn": ["^build"],
      "inputs": ["python", "sharedGlobals"],
      "outputs": ["{projectRoot}/dist"]
    },
    "test": {
      "cache": true,
      "inputs": ["python", "tests", "sharedGlobals"],
      "outputs": [
        "{projectRoot}/.coverage",
        "{projectRoot}/coverage.xml",
        "{projectRoot}/test-results.xml"
      ]
    },
    "lint": {
      "cache": true,
      "inputs": ["python", "sharedGlobals"],
      "outputs": []
    },
    "install": {
      "cache": false
    },
    "version": {
      "cache": false
    }
  },

  "generators": {
    "@nxlv/python:uv-project": {
      "linter": "ruff",
      "unitTestRunner": "pytest",
      "srcDir": true,
      "buildSystem": "hatch"
    }
  },

  "release": {
    "projects": ["packages/*"],
    "projectsRelationship": "independent",
    "releaseTagPattern": "{projectName}@{version}",
    "changelog": {
      "automaticFromRef": true,
      "projectChangelogs": {
        "file": "{projectRoot}/CHANGELOG.md"
      }
    },
    "version": {
      "conventionalCommits": true,
      "versionActions": "@nxlv/python/release/version-actions",
      "generatorOptions": {
        "fallbackCurrentVersionResolver": "registry"
      }
    },
    "git": {
      "commit": true,
      "commitMessage": "chore(release): {projectName} v{version}",
      "tag": true,
      "push": true
    }
  },

  "sync": {
    "globalGenerators": ["@nxlv/python:pkg-sync"],
    "applyChanges": false
  },

  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "test", "lint"]
      }
    }
  }
}
```

### 4.4 יצירת פרויקטים חדשים עם הgenerator

`@nxlv/python` מספק generator שיוצר את כל מבנה הפרויקט נכון:

```bash
# יצירת ספרייה חדשה
npx nx generate @nxlv/python:uv-project shared-utils \
  --directory=packages \
  --projectType=library \
  --linter=ruff \
  --unitTestRunner=pytest \
  --srcDir \
  --buildSystem=hatch

# יצירת אפליקציה חדשה
npx nx generate @nxlv/python:uv-project api-service \
  --directory=apps \
  --projectType=application \
  --linter=ruff \
  --unitTestRunner=pytest \
  --srcDir
```

הgenerator יצור:
- `packages/shared-utils/pyproject.toml` (מוגדר נכון)
- `packages/shared-utils/project.json` (עם כל ה-targets)
- `packages/shared-utils/src/shared_utils/__init__.py`
- `packages/shared-utils/tests/test_main.py`

### 4.5 project.json לפרויקט Python עם @nxlv/python

```json
{
  "name": "shared-utils",
  "$schema": "../../node_modules/nx/schemas/project-schema.json",
  "projectType": "library",
  "sourceRoot": "packages/shared-utils/src",
  "tags": ["python", "scope:shared", "type:lib"],

  "targets": {

    "install": {
      "executor": "@nxlv/python:install",
      "options": {}
    },

    "lock": {
      "executor": "@nxlv/python:lock",
      "options": {}
    },

    "build": {
      "executor": "@nxlv/python:build",
      "outputs": ["{projectRoot}/dist"],
      "options": {
        "outputPath": "packages/shared-utils/dist",
        "publish": false,
        "bundleLocalDependencies": true,
        "lockedVersions": true
      },
      "dependsOn": ["^build"]
    },

    "publish": {
      "executor": "@nxlv/python:publish",
      "options": {
        "buildTarget": "shared-utils:build",
        "publish": true
      },
      "dependsOn": ["build"]
    },

    "test": {
      "executor": "@nxlv/python:run-commands",
      "outputs": [
        "{projectRoot}/coverage.xml",
        "{projectRoot}/.coverage"
      ],
      "options": {
        "command": "pytest packages/shared-utils/tests -v --cov=src/shared_utils --cov-report=xml:coverage.xml --cov-report=term",
        "cwd": "{workspaceRoot}"
      },
      "inputs": ["python", "tests", "sharedGlobals"],
      "cache": true
    },

    "lint": {
      "executor": "@nxlv/python:ruff",
      "options": {
        "lintFilePatterns": ["packages/shared-utils/src", "packages/shared-utils/tests"]
      },
      "inputs": ["python", "sharedGlobals"],
      "cache": true
    },

    "format": {
      "executor": "@nxlv/python:run-commands",
      "options": {
        "command": "python tools/format.py packages/shared-utils",
        "cwd": "{workspaceRoot}"
      }
    },

    "format-check": {
      "executor": "@nxlv/python:run-commands",
      "options": {
        "command": "python tools/format.py packages/shared-utils --check",
        "cwd": "{workspaceRoot}"
      },
      "inputs": ["python"],
      "cache": true
    },

    "version": {
      "executor": "nx:run-commands",
      "options": {
        "command": "echo versioning handled by nx release"
      }
    }
  }
}
```

> **שימו לב:** `@nxlv/python:run-commands` (ולא `nx:run-commands`) — הexecutor הזה **מפעיל את ה-venv** לפני הרצת הפקודה. זה קריטי לצורה שבה Python packages נמצאים.

### 4.6 project.json לפרויקט Vue.js

```json
{
  "name": "web-app",
  "$schema": "../../node_modules/nx/schemas/project-schema.json",
  "projectType": "application",
  "sourceRoot": "frontend/web-app/src",
  "tags": ["vue", "scope:frontend", "type:app"],

  "targets": {
    "build": {
      "executor": "@nx/vite:build",
      "outputs": ["{workspaceRoot}/dist/frontend/web-app"],
      "options": {
        "outputPath": "dist/frontend/web-app",
        "configFile": "frontend/web-app/vite.config.ts"
      },
      "inputs": ["frontend", "sharedGlobals"],
      "cache": true
    },

    "serve": {
      "executor": "@nx/vite:dev-server",
      "options": {
        "buildTarget": "web-app:build",
        "configFile": "frontend/web-app/vite.config.ts"
      }
    },

    "test": {
      "executor": "@nx/vite:test",
      "outputs": ["{projectRoot}/coverage"],
      "options": {
        "configFile": "frontend/web-app/vite.config.ts"
      },
      "inputs": ["frontend", "sharedGlobals"],
      "cache": true
    },

    "lint": {
      "executor": "@nx/eslint:lint",
      "outputs": ["{options.outputFile}"],
      "options": {
        "lintFilePatterns": ["frontend/web-app/**/*.{ts,vue}"]
      },
      "inputs": ["frontend", "sharedGlobals"],
      "cache": true
    }
  }
}
```

### 4.7 הupdated pyproject.toml של member עבור @nxlv/python

```toml
[project]
name = "shared-utils"
version = "1.0.0"
description = "Shared utility library"
authors = [{ name = "Platform Team", email = "platform@company.com" }]
requires-python = ">=3.11"
dependencies = [
    "requests>=2.28",
    "pydantic>=2.0",
]

[project.optional-dependencies]
data = ["numpy>=1.24"]

[dependency-groups]
dev = [
    "pytest>=7.0",
    "pytest-cov>=4.0",
]
lint = [
    "ruff>=0.9",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/shared_utils"]

# NX Release יכתוב כאן את גרסאות ה-dependencies
[tool.uv.sources]
data-models = { workspace = true }   # workspace dependency
```

### 4.8 NX Affected — הלב של הפעולה

#### כיצד NX + @nxlv/python בונים את ה-Project Graph

1. **`@nxlv/python` סורק** כל `pyproject.toml` של workspace member
2. **מזהה** `[tool.uv.sources]` עם `{ workspace = true }` — אלו תלויות בין projects
3. **בונה edges** בProject Graph: אם `api-service` תלוי ב-`shared-utils`, קיים edge
4. **NX מפיץ** שינויים דרך הgraph: שינוי ב-`shared-utils` → גם `api-service` affected

#### הפעלת affected בפועל

```bash
# הצגת graph
npx nx graph

# הצגת affected projects בלבד
npx nx show projects --affected --base=main --head=HEAD

# build רק של affected
npx nx affected -t build --parallel=4

# test + lint במקביל על affected
npx nx affected -t test,lint --parallel=4

# dry run — הצג מה היה מריץ
npx nx affected -t build --dry-run
```

### 4.9 inferDependencies: true — אינטגרציה עמוקה

כאשר `inferDependencies: true` מוגדר בnx.json, הפלאגין:

1. **סורק קבצי Python** לאיתור `import` statements
2. **מזהה** imports של workspace members
3. **מתריע** כאשר import קיים אבל `pyproject.toml` לא מצהיר על התלות

עם **`@nxlv/python:pkg-sync`** generator:
```bash
# בדוק מה חסר
npx nx sync:check

# הוסף תלויות חסרות אוטומטית
npx nx sync
```

דוגמה: `api-service` מייבא `from shared_utils import helper` אבל `shared-utils` לא בrequirements → `nx sync` יוסיף אוטומטית.

### 4.10 NX Cache

#### Cache מקומי

Cache מאוחסן ב-`.nx/cache`. בכל הרצת target, NX:
1. מחשב hash של קלט (קבצים + env vars + פרמטרים)
2. בודק cache
3. אם hit — משחזר תוצאות
4. אם miss — מריץ ושומר

```bash
# ניקוי cache
npx nx reset
```

#### Cache מרחוק עם MinIO (on-premise)

```bash
npm install -D @nx/s3-cache
```

`.nx/s3cache.json` (לא ב-git):
```json
{
  "endpoint": "https://minio.internal.company.com",
  "bucket": "nx-cache",
  "region": "us-east-1",
  "forcePathStyle": true,
  "disableChecksum": true
}
```

`nx.json`:
```json
{
  "nxCloudOptions": {
    "runner": "@nx/s3-cache"
  }
}
```

> ⚠️ **CVE-2025-36852 (CREEP):** פגיעות cache poisoning. הגדירו write-only לmain branch.

### 4.11 namedInputs — שליטה מדויקת על cache invalidation

```json
{
  "namedInputs": {
    "python": [
      "{projectRoot}/**/*.py",
      "{projectRoot}/pyproject.toml",
      "!{projectRoot}/**/__pycache__/**",
      "!{projectRoot}/**/*.pyc",
      "!{projectRoot}/.pytest_cache/**",
      "!{projectRoot}/dist/**"
    ],
    "sharedGlobals": [
      "{workspaceRoot}/nx.json",
      "{workspaceRoot}/pyproject.toml",
      "{workspaceRoot}/uv.lock",
      "{workspaceRoot}/ruff.toml",
      "{workspaceRoot}/tools/format.py"
    ]
  }
}
```

> **טיפ:** שינוי ב-`uv.lock` invalidates cache לכולם (כי הוא ב-`sharedGlobals`). זה נכון — אם תלויות השתנו, כדאי לאמת מחדש.

### 4.12 Tags וגבולות ארכיטקטורה

```json
{
  "tags": ["python", "scope:shared", "type:lib"]
}
```

| מימד Tag | ערכים | משמעות |
|-----------|-------|---------|
| `scope:` | `shared`, `core`, `feature`, `app` | שכבת שייכות |
| `type:` | `lib`, `app`, `tool` | סוג פרויקט |
| `tech:` | `python`, `vue` | טכנולוגיה |

```bash
# הרצה לפי tag
npx nx run-many -t build --projects=tag:scope:shared
npx nx affected -t test --projects=tag:python
```

---

## 5. שלב 2 — UV: מיגרציה מ-Poetry

### 5.1 מהו UV ולמה לעבור?

UV הוא מנהל חבילות Python של חברת Astral (יוצרי RUFF), כתוב ב-Rust.

| מאפיין | Poetry | UV |
|--------|--------|-----|
| מהירות install (cold) | ~48 שניות | ~7 שניות |
| מהירות install (warm) | ~4 שניות | <1 שנייה |
| lockfile | תלוי פלטפורמה | Universal (כל OS + כל Python version) |
| Workspace support | אין | כן — Cargo-style |
| Python management | צריך pyenv | מובנה |
| תאימות PEP | `[tool.poetry]` | PEP 621 `[project]` |

### 5.2 UV Workspaces — איך הם עובדים עם @nxlv/python

UV workspace + @nxlv/python = כוח מלא:

- **UV** מנהל תלויות, lockfile, venv
- **@nxlv/python** מבין את מבנה ה-workspace ובונה את ה-Project Graph של NX
- **שורש אחד** `uv.lock` לכל הworkspace members
- **venv אחד** `{root}/.venv` משותף

**`pyproject.toml` בשורש:**
```toml
[project]
name = "monorepo-root"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = []

[tool.uv.workspace]
members = ["packages/*", "apps/*"]
exclude = ["packages/legacy-*"]

[tool.uv.sources]
shared-utils = { workspace = true }
data-models = { workspace = true }
ml-pipeline = { workspace = true }

[[tool.uv.index]]
name = "company-artifactory"
url = "https://artifactory.internal.company.com/artifactory/api/pypi/pypi-virtual/simple"
default = true

[tool.uv]
index-strategy = "first-index"
python-preference = "only-system"     # לרשת סגורה
python-downloads = "never"            # לרשת סגורה
```

### 5.3 מיגרציה מ-Poetry ל-UV — שלב אחר שלב

#### שלב א: inventory

```bash
# מצא כל pyproject.toml
find . -name "pyproject.toml" \
  -not -path "*/node_modules/*" \
  -not -path "*/.venv/*"

# בדוק groups
grep -r "\[tool.poetry.group" . --include="*.toml"

# בדוק private sources
grep -r "\[\[tool.poetry.source\]\]" . --include="*.toml"
```

#### שלב ב: המרת pyproject.toml

**לפני (Poetry):**
```toml
[tool.poetry]
name = "my-package"
version = "1.0.0"
description = "A package"
authors = ["Developer <dev@company.com>"]
packages = [{include = "my_package", from = "src"}]

[tool.poetry.dependencies]
python = "^3.11"
requests = {version = "^2.28", extras = ["security"]}

[tool.poetry.group.dev.dependencies]
pytest = "^7.0"
pytest-cov = "^4.0"

[tool.poetry.group.lint.dependencies]
ruff = "^0.9"

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

**אחרי (UV/PEP 621):**
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
    "pytest>=7.0",
    "pytest-cov>=4.0",
]
lint = [
    "ruff>=0.9",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_package"]
```

#### שלב ג: מחיקת lockfile ויצירה חדשה

```bash
find . -name "poetry.lock" -delete
find . -name ".venv" -type d -exec rm -rf {} + 2>/dev/null || true

uv lock
uv sync --all-groups
```

#### שלב ד: בדיקת תאימות

```bash
uv run python -c "import my_package; print('OK')"
uv run pytest
uv tree
```

### 5.4 טבלת המרת פקודות

| Poetry | UV |
|--------|-----|
| `poetry install` | `uv sync` |
| `poetry install --no-dev` | `uv sync --no-dev` |
| `poetry add requests` | `uv add requests` |
| `poetry add --dev pytest` | `uv add --dev pytest` |
| `poetry add --group lint ruff` | `uv add --group lint ruff` |
| `poetry remove requests` | `uv remove requests` |
| `poetry update` | `uv lock --upgrade` |
| `poetry run python script.py` | `uv run python script.py` |
| `poetry build` | `uv build` |
| `poetry publish` | `uv publish` |
| `poetry lock` | `uv lock` |
| `poetry show --tree` | `uv tree` |
| `poetry export -f requirements.txt` | `uv export --format requirements.txt` |
| `poetry shell` | `source .venv/bin/activate` |

### 5.5 Private Registry לרשת סגורה

```toml
[[tool.uv.index]]
name = "company-artifactory"
url = "https://artifactory.internal.company.com/artifactory/api/pypi/pypi-virtual/simple"
publish-url = "https://artifactory.internal.company.com/artifactory/api/pypi/pypi-local"
default = true

[tool.uv]
no-index = false
index-strategy = "first-index"
```

**Authentication (לעולם לא בקוד):**
```bash
# CI/CD variables
export UV_INDEX_COMPANY_ARTIFACTORY_USERNAME="ci-user"
export UV_INDEX_COMPANY_ARTIFACTORY_PASSWORD="${ARTIFACTORY_TOKEN}"
```

**לסביבה ללא גישה לPyPI בכלל:**
```toml
[tool.uv]
no-index = true

[[tool.uv.index]]
name = "internal"
url = "https://pypi.internal.company.com/simple/"
default = true
```

### 5.6 uv.lock — Universal Lockfile

```bash
uv lock             # יצירה/עדכון
uv lock --check     # בדיקה שh-lockfile עדכני (לCI)
uv sync --frozen    # install ללא שינוי lockfile (לCI)
uv lock --upgrade-package requests  # upgrade ספציפי
```

---

## 6. שלב 3 — RUFF: לינטינג ופורמטינג

### 6.1 מהו RUFF

RUFF הוא לינטר Python כתוב ב-Rust, פי 10-100 מהיר מהכלים הקיימים. הוא מחליף:

| כלי ישן | RUFF שקול |
|---------|-----------|
| flake8 | `ruff check` |
| isort | `ruff check --select I` |
| black | `ruff format` |
| pyupgrade | `ruff check --select UP` |
| bandit | `ruff check --select S` |
| pydocstyle | `ruff check --select D` |

### 6.2 הgenerator מוסיף RUFF אוטומטית

כאשר יוצרים פרויקט עם:
```bash
npx nx generate @nxlv/python:uv-project mylib --linter=ruff
```

הgenerator מוסיף אוטומטית:
- `ruff.toml` לפרויקט
- `lint` target עם `@nxlv/python:ruff` executor בproject.json
- `ruff` ל-`[dependency-groups] lint`

### 6.3 הגדרת RUFF גלובלי

**`ruff.toml` בשורש המונורפו:**
```toml
# ruff.toml — גלובלי לכל המונורפו
target-version = "py311"
line-length = 100

[lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "B",    # flake8-bugbear
    "C4",   # flake8-comprehensions
    "UP",   # pyupgrade
    "N",    # pep8-naming
    "S",    # flake8-bandit (security)
    "PTH",  # flake8-use-pathlib
    "SIM",  # flake8-simplify
    "RUF",  # ruff-specific
]
ignore = [
    "E501",    # line too long (מטופל על ידי formatter)
    "ANN101",  # missing self annotation
    "ANN102",  # missing cls annotation
    "S101",    # use of assert
]

[lint.per-file-ignores]
"tests/**/*.py" = ["S101", "ANN", "D"]
"**/migrations/**/*.py" = ["ALL"]

[lint.isort]
known-first-party = [
    "shared_utils",
    "data_models",
    "ml_pipeline",
    "api_service"
]
force-sort-within-sections = true

[lint.pydocstyle]
convention = "google"

[format]
quote-style = "double"
indent-style = "space"
skip-magic-trailing-comma = false
line-ending = "lf"
```

### 6.4 @nxlv/python:ruff executor — הגדרה בproject.json

```json
{
  "targets": {
    "lint": {
      "executor": "@nxlv/python:ruff",
      "options": {
        "lintFilePatterns": [
          "packages/shared-utils/src",
          "packages/shared-utils/tests"
        ]
      },
      "inputs": ["python", "sharedGlobals"],
      "cache": true
    }
  }
}
```

> **חשוב:** השתמשו ב-`@nxlv/python:ruff` ולא ב-`nx:run-commands` עם `ruff check`. הexecutor הייעודי:
> - מפעיל ה-venv לפני הריצה
> - מכיר את מבנה הפרויקט
> - מוסיף תמיכה ב-NX caching נכונה

### 6.5 RUFF per-project override

```toml
# packages/my-lib/ruff.toml
extend = "../../ruff.toml"

[lint]
extend-select = ["D"]
extend-ignore = ["UP007"]

[lint.per-file-ignores]
"src/my_lib/internal/**" = ["D", "ANN"]
```

### 6.6 מיגרציה מ-flake8/black/isort

```bash
# הרצת auto-fix ראשוני — commit נפרד!
uv run ruff check . --fix --unsafe-fixes
uv run ruff format .

git add -A
git commit -m "chore: apply RUFF formatting (initial migration)"
```

> ⚠️ **קריטי:** עשו commit זה לפני כל שינוי אחר. זה יגרום ל-merge conflicts בכל ה-PRs הפתוחים — הודיעו לצוות מראש!

### 6.7 false positives נפוצים

| שגיאה | סיבה | פתרון |
|-------|-------|--------|
| `B008` ב-FastAPI | `Depends()` כdefault arg | `per-file-ignores: "routers/**" = ["B008"]` |
| `S101` | assert בtests | `per-file-ignores: "tests/**" = ["S101"]` |
| `ISC001` | conflict עם formatter | `ignore = ["ISC001"]` |
| `ANN101` | self annotation | `ignore = ["ANN101", "ANN102"]` |
| `UP007` | Optional[X] | ב-Python < 3.10, ignore |
| `D401` | imperative mood | `ignore = ["D401"]` אם אינכם אוכפים |

### 6.8 pre-commit hooks

```yaml
# .pre-commit-config.yaml לרשת פתוחה
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.9.10
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format
```

**לרשת סגורה (local hooks):**
```yaml
repos:
  - repo: local
    hooks:
      - id: ruff-check
        name: RUFF lint
        language: system
        entry: uv run ruff check --fix
        types: [python]
        pass_filenames: true

      - id: ruff-format
        name: RUFF format
        language: system
        entry: uv run ruff format
        types: [python]
        pass_filenames: true
```

---

## 7. שלב 4 — פורמטר מותאם

### 7.1 הסקריפט

**`tools/format.py`:**
```python
#!/usr/bin/env python3
"""Custom formatter script — applies RUFF formatting to a directory.

Usage:
    python tools/format.py <directory>
    python tools/format.py packages/my-lib
    python tools/format.py packages/my-lib --check
    python tools/format.py packages/my-lib --diff
"""

import argparse
import subprocess
import sys
from pathlib import Path


def parse_args() -> argparse.Namespace:
    parser = argparse.ArgumentParser(
        description="Apply RUFF formatting to a directory",
        formatter_class=argparse.RawDescriptionHelpFormatter,
    )
    parser.add_argument("directory", type=Path, help="Directory to format")
    parser.add_argument(
        "--check",
        action="store_true",
        help="Check only, do not modify files (exit 1 if changes needed)",
    )
    parser.add_argument(
        "--diff",
        action="store_true",
        help="Show diff without modifying files",
    )
    parser.add_argument("--verbose", "-v", action="store_true")
    return parser.parse_args()


def validate_directory(directory: Path) -> None:
    if not directory.exists():
        print(f"Error: Directory '{directory}' does not exist.", file=sys.stderr)
        sys.exit(1)
    if not directory.is_dir():
        print(f"Error: '{directory}' is not a directory.", file=sys.stderr)
        sys.exit(1)


def run(cmd: list[str], verbose: bool) -> int:
    if verbose:
        print(f"Running: {' '.join(cmd)}")
    return subprocess.run(cmd, capture_output=False).returncode


def main() -> None:
    args = parse_args()
    validate_directory(args.directory)

    print(f"{'Checking' if args.check else 'Formatting'} '{args.directory}'...")

    # שלב 1: תיקון imports (isort-style) — רק כאשר לא בmode בדיקה
    if not args.check and not args.diff:
        rc = run(
            ["ruff", "check", str(args.directory), "--select", "I,F401,UP", "--fix"],
            args.verbose,
        )
        if rc != 0:
            print("Warning: Some lint fixes could not be applied automatically.")

    # שלב 2: פורמוט (black-style)
    cmd = ["ruff", "format", str(args.directory)]
    if args.check:
        cmd.append("--check")
    if args.diff:
        cmd.append("--diff")
    if args.verbose:
        cmd.append("--verbose")

    rc = run(cmd, args.verbose)

    if rc == 0:
        print(f"✓ '{args.directory}' {'is properly formatted' if args.check else 'formatted successfully'}.")
    else:
        if args.check:
            print(f"✗ '{args.directory}' needs formatting. Run without --check to fix.")
        else:
            print(f"✗ Formatting failed for '{args.directory}'.")

    sys.exit(rc)


if __name__ == "__main__":
    main()
```

### 7.2 שימוש

```bash
uv run python tools/format.py packages/shared-utils
uv run python tools/format.py packages/shared-utils --check   # לCI
uv run python tools/format.py packages/shared-utils --diff    # לreview
uv run python tools/format.py .                               # כל המונורפו
```

### 7.3 אינטגרציה ב-NX

ב-`project.json` של כל Python project:
```json
{
  "targets": {
    "format": {
      "executor": "@nxlv/python:run-commands",
      "options": {
        "command": "python tools/format.py {projectRoot}",
        "cwd": "{workspaceRoot}"
      }
    },
    "format-check": {
      "executor": "@nxlv/python:run-commands",
      "options": {
        "command": "python tools/format.py {projectRoot} --check",
        "cwd": "{workspaceRoot}"
      },
      "inputs": ["python"],
      "cache": true
    }
  }
}
```

> שימו לב: שוב `@nxlv/python:run-commands` — הvenv מופעל לפני הריצה, כך `python` ו-`ruff` שבvenv זמינים.

```bash
npx nx affected -t format
npx nx affected -t format-check
```

---

## 8. אינטגרציה עם GitLab CI/CD

### 8.1 משתני סביבה קריטיים

```yaml
variables:
  # UV — חובות
  UV_LINK_MODE: "copy"            # ⚠️ חובה! GitLab אינו תומך ב-hardlinks
  UV_CACHE_DIR: ".uv-cache"
  UV_FROZEN: "true"               # לCI: אל תשנה lockfile
  UV_NO_MANAGED_PYTHON: "1"       # רשת סגורה: Python מהמערכת
  UV_PYTHON_DOWNLOADS: "never"    # רשת סגורה: אל תנסה להוריד Python

  # NX
  NX_BASE: "${CI_MERGE_REQUEST_DIFF_BASE_SHA:-${CI_COMMIT_BEFORE_SHA:-$(git rev-parse HEAD~1 2>/dev/null || echo HEAD)}}"
  NX_HEAD: "$CI_COMMIT_SHA"
  NX_NO_CLOUD: "true"             # רשת סגורה: ללא NX Cloud

  # Credentials
  UV_INDEX_COMPANY_ARTIFACTORY_USERNAME: "${ARTIFACTORY_USER}"
  UV_INDEX_COMPANY_ARTIFACTORY_PASSWORD: "${ARTIFACTORY_TOKEN}"
```

> ⚠️ **`UV_LINK_MODE: "copy"` הוא קריטי!** GitLab CI יוצר mountpoint נפרד לתיקיית הbuild. hard links אינם עובדים בין mountpoints שונים — UV ייכשל בשקט.

### 8.2 `.gitlab-ci.yml` מלא

```yaml
# .gitlab-ci.yml

variables:
  PYTHON_VERSION: "3.12"
  UV_LINK_MODE: "copy"
  UV_CACHE_DIR: ".uv-cache"
  UV_FROZEN: "true"
  UV_NO_MANAGED_PYTHON: "1"
  UV_PYTHON_DOWNLOADS: "never"
  UV_INDEX_COMPANY_ARTIFACTORY_USERNAME: "${ARTIFACTORY_USER}"
  UV_INDEX_COMPANY_ARTIFACTORY_PASSWORD: "${ARTIFACTORY_TOKEN}"
  NX_NO_CLOUD: "true"

default:
  image: registry.internal.company.com/ci/python-nx:latest
  tags:
    - docker
    - internal
  before_script:
    - |
      NX_BASE="${CI_MERGE_REQUEST_DIFF_BASE_SHA:-${CI_COMMIT_BEFORE_SHA}}"
      if [ -z "$NX_BASE" ]; then
        NX_BASE=$(git rev-parse HEAD~1 2>/dev/null || echo "HEAD")
      fi
      export NX_BASE
      export NX_HEAD="$CI_COMMIT_SHA"
      echo "NX_BASE=$NX_BASE"
      echo "NX_HEAD=$NX_HEAD"
  cache:
    - key:
        files: [uv.lock]
        prefix: "uv-v1"
      paths: [${UV_CACHE_DIR}]
      policy: pull
    - key: "nx-cache-${CI_COMMIT_REF_SLUG}"
      paths: [.nx/cache/]
      policy: pull

stages:
  - setup
  - validate
  - test
  - build
  - release

# ─────────────────────────────────────────────────────
# SETUP — install + warm cache
# ─────────────────────────────────────────────────────
setup:deps:
  stage: setup
  cache:
    - key:
        files: [uv.lock]
        prefix: "uv-v1"
      paths: [${UV_CACHE_DIR}]
      policy: pull-push
    - key: "nx-cache-${CI_COMMIT_REF_SLUG}"
      paths: [.nx/cache/]
      policy: pull-push
  script:
    - uv sync --all-groups --frozen
    - npm ci --prefer-offline
  after_script:
    - uv cache prune --ci

# ─────────────────────────────────────────────────────
# VALIDATE
# ─────────────────────────────────────────────────────
validate:lockfile:
  stage: validate
  needs: []
  script:
    - uv lock --check
  rules:
    - changes: ["**/pyproject.toml", "pyproject.toml"]

validate:lint:
  stage: validate
  needs: [setup:deps]
  script:
    - |
      AFFECTED=$(npx nx show projects --affected --base=$NX_BASE --head=$NX_HEAD 2>/dev/null | tr '\n' ',')
      if [ -z "$AFFECTED" ]; then
        echo "No affected projects."
        exit 0
      fi
    - npx nx affected -t lint --base=$NX_BASE --head=$NX_HEAD --parallel=4
  artifacts:
    reports:
      codequality: ruff-code-quality.json
    when: always
    expire_in: 1 week

validate:format:
  stage: validate
  needs: [setup:deps]
  script:
    - npx nx affected -t format-check --base=$NX_BASE --head=$NX_HEAD --parallel=4

# ─────────────────────────────────────────────────────
# TEST
# ─────────────────────────────────────────────────────
test:python:
  stage: test
  needs: [validate:lint, validate:format]
  script:
    - npx nx affected -t test --base=$NX_BASE --head=$NX_HEAD --parallel=2
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: "**/coverage.xml"
      junit: "**/test-results.xml"
    when: always

test:frontend:
  stage: test
  needs: [setup:deps]
  script:
    - npx nx affected -t test --base=$NX_BASE --head=$NX_HEAD --parallel=4
  rules:
    - changes: ["frontend/**/*", "package.json"]

# ─────────────────────────────────────────────────────
# BUILD
# ─────────────────────────────────────────────────────
build:packages:
  stage: build
  needs: [test:python]
  script:
    - npx nx affected -t build --base=$NX_BASE --head=$NX_HEAD --parallel=4
  artifacts:
    paths: ["**/dist/"]
    expire_in: 1 week
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG

# ─────────────────────────────────────────────────────
# RELEASE
# ─────────────────────────────────────────────────────
release:version:
  stage: release
  needs: [build:packages]
  script:
    - git config user.email "ci-bot@company.com"
    - git config user.name "GitLab CI Bot"
    - |
      # מצא רק ספריות שהשתנו (type:lib בלבד)
      AFFECTED_LIBS=$(npx nx show projects --affected \
        --base=$NX_BASE --head=$NX_HEAD \
        --filter-tag=type:lib 2>/dev/null | tr '\n' ',')
      if [ -z "$AFFECTED_LIBS" ]; then
        echo "No libraries affected. Skipping release."
        exit 0
      fi
      echo "Releasing: $AFFECTED_LIBS"
    - npx nx release --projects=$AFFECTED_LIBS --yes
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
  environment:
    name: production
  variables:
    GL_TOKEN: "${GITLAB_RELEASE_TOKEN}"
    GIT_PUSH_TOKEN: "${CI_PUSH_TOKEN}"

release:publish:
  stage: release
  needs: [release:version]
  script:
    - npx nx affected -t publish --base=$NX_BASE --head=$NX_HEAD
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
```

### 8.3 RUFF Code Quality ב-GitLab

```bash
# יצירת report בפורמט GitLab
uv run ruff check . \
  --output-format=gitlab \
  --output-file=ruff-code-quality.json
```

תוצאות מופיעות בדף ה-MR ישירות.

### 8.4 Dockerfile לCI Runner פנימי

```dockerfile
FROM python:3.12-slim-bookworm

# UV
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/

# Node.js + npm (לNX)
RUN apt-get update && apt-get install -y curl git \
    && curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g pnpm \
    && rm -rf /var/lib/apt/lists/*

# פרטי חברה — pip pointing to Artifactory
RUN pip config set global.index-url \
    https://artifactory.internal.company.com/artifactory/api/pypi/pypi-virtual/simple

# npm pointing to internal Verdaccio
RUN npm config set registry https://verdaccio.internal.company.com/

ENV UV_LINK_MODE=copy \
    UV_NO_MANAGED_PYTHON=1 \
    UV_PYTHON_DOWNLOADS=never \
    NODE_ENV=production

WORKDIR /workspace
```

---

## 9. עבודה ברשת סגורה

### 9.1 אתגרי רשת סגורה

| רכיב | בעיה | פתרון |
|------|-------|--------|
| UV binary | לא ניתן להוריד | Docker / העברה ידנית |
| npm packages | registry חיצוני | Verdaccio/Nexus |
| Python packages | PyPI לא נגיש | Artifactory/Nexus/devpi |
| Docker images | Docker Hub לא נגיש | GitLab Container Registry |
| Python downloads | UV מנסה להוריד | `UV_PYTHON_DOWNLOADS=never` |
| Node.js | nodejs.org לא נגיש | package manager הפצה |

### 9.2 הגדרת Artifactory כ-PyPI Mirror

```toml
[[tool.uv.index]]
name = "company-artifactory"
url = "https://artifactory.internal.company.com/artifactory/api/pypi/pypi-virtual/simple"
publish-url = "https://artifactory.internal.company.com/artifactory/api/pypi/pypi-local"
default = true

[tool.uv]
no-index = true    # לאבטחה מרבית — רק Artifactory
index-strategy = "first-index"
```

> ⚠️ **קריטי ב-Artifactory:** ב-Admin → Repositories → pypi-virtual → כבה **"Forward PyPI package requests to PyPI.org"**. ללא זה, packages שאינם ב-Artifactory נופלים ל-PyPI.org בשקט!

### 9.3 UV ב-air-gapped

```bash
# על מכונה עם אינטרנט:
# הורד מ: https://github.com/astral-sh/uv/releases
# e.g.: uv-x86_64-unknown-linux-gnu.tar.gz

# העבר לשרת הפנימי והתקן:
tar -xzf uv-x86_64-unknown-linux-gnu.tar.gz
sudo cp uv /usr/local/bin/
sudo chmod +x /usr/local/bin/uv
```

**Environment variables לairgap:**
```bash
export UV_NO_MANAGED_PYTHON=1
export UV_PYTHON_DOWNLOADS=never
export UV_INDEX_URL=https://artifactory.internal.company.com/...
```

### 9.4 npm packages לרשת סגורה (Verdaccio)

```yaml
# verdaccio-config.yml
storage: /verdaccio/storage
packages:
  '@*/*':
    access: $all
    publish: $authenticated
    proxy: npmjs
  '**':
    access: $all
    publish: $authenticated
    proxy: npmjs
uplinks:
  npmjs:
    url: https://registry.npmjs.org/
    cache: true
```

```bash
# הורד packages הנדרשים לNX מבחוץ וסנכרן:
npm pack nx @nxlv/python @nx/vue @nx/vite
# העבר tar.gz לשרת הפנימי
npm publish --registry https://verdaccio.internal.company.com/
```

### 9.5 GitLab Runner לרשת סגורה

```toml
# /etc/gitlab-runner/config.toml
[[runners]]
  name = "internal-runner"
  url = "https://gitlab.internal.company.com/"
  executor = "docker"

  [runners.docker]
    image = "registry.internal.company.com/ci/python-nx:latest"
    pull_policy = ["if-not-present", "never"]  # אל תנסה Docker Hub!
    volumes = ["/cache"]
```

---

## 10. Release חכם עם NX ו-@nxlv/python

### 10.1 הארכיטקטורה הנכונה לRelease Python

כאן נמצאת אחת ההבנות הכי חשובות:

```
NX Release Engine (מובנה)
    ↓
    קורא commit history (Conventional Commits)
    מחשב bump (major/minor/patch)
    קורא VersionActions implementation
    ↓
@nxlv/python/release/version-actions
    ↓
    עדכן גרסה ב-pyproject.toml
    עדכן uv.lock
    צור CHANGELOG.md
    ↓
NX Release Engine (המשך)
    ↓
    git commit + git tag
    push
```

**לא** `@jscutlery/semver` לPython. הוא לJS/TS בלבד.

### 10.2 הגדרה בnx.json

```json
{
  "release": {
    "projects": ["packages/*"],
    "projectsRelationship": "independent",
    "releaseTagPattern": "{projectName}@{version}",

    "changelog": {
      "automaticFromRef": true,
      "projectChangelogs": {
        "file": "{projectRoot}/CHANGELOG.md"
      }
    },

    "version": {
      "conventionalCommits": true,
      "versionActions": "@nxlv/python/release/version-actions",
      "generatorOptions": {
        "fallbackCurrentVersionResolver": "registry"
      }
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

### 10.3 הגדרה בproject.json

```json
{
  "name": "shared-utils",
  "targets": {
    "publish": {
      "executor": "@nxlv/python:publish",
      "options": {
        "buildTarget": "shared-utils:build",
        "publish": true
      },
      "dependsOn": ["build"]
    }
  },

  "release": {
    "version": {
      "versionActions": "@nxlv/python/release/version-actions"
    }
  }
}
```

### 10.4 Conventional Commits — הבסיס לautomation

NX Release קורא commit messages ומחשב bump:

| Commit | גרסה |
|--------|------|
| `feat(shared-utils): add pagination` | minor bump |
| `fix(shared-utils): handle None values` | patch bump |
| `feat!: remove deprecated auth` | major bump |
| `chore: update deps` | ללא bump |
| `docs: update README` | ללא bump |

> **קריטי:** ה-scope בcommit message (`feat(shared-utils):`) מאפשר לNX לדעת **לאיזה project** שייך ה-bump. ללא scope, כל affected projects יקבלו bump.

### 10.5 Release של ספריות ספציפיות

```bash
# Release ספרייה אחת
npx nx release --projects=shared-utils

# Release מספר ספריות
npx nx release --projects=shared-utils,data-models

# Release לפי tag (כל הlib scope)
npx nx release --projects=tag:type:lib

# Dry run — preview בלבד
npx nx release --projects=shared-utils --dry-run

# גרסה ידנית
npx nx release version --projects=shared-utils --specifier=2.0.0
```

### 10.6 @jscutlery/semver לGitLab Releases (Vue/JS)

לפרויקטי Vue.js, אם רוצים GitLab Release objects (לא רק tags):

```bash
npm install -D @jscutlery/semver
```

**project.json לVue:**
```json
{
  "targets": {
    "version": {
      "executor": "@jscutlery/semver:version",
      "options": {
        "preset": "conventional-commits",
        "trackDeps": true,
        "commitMessageFormat": "chore(release): ${projectName} v${version}",
        "postTargets": ["web-app:gitlab-release"]
      }
    },
    "gitlab-release": {
      "executor": "@jscutlery/semver:gitlab",
      "options": {
        "tag": "${version}",
        "notes": "${notes}"
      }
    }
  }
}
```

> **שימו לב:** `@jscutlery/semver:gitlab` דורש GitLab Release CLI מותקן על ה-runner.

### 10.7 bundleLocalDependencies — לdeployments

כאשר מפרסמים ל-Lambda/serverless, צריך wheel שמכיל את כל התלויות:

```json
{
  "build": {
    "executor": "@nxlv/python:build",
    "options": {
      "outputPath": "apps/lambda-func/dist",
      "bundleLocalDependencies": true,
      "lockedVersions": true
    }
  }
}
```

הexecutor יעתיק את כל workspace members שה-app תלוי בהם לתוך ה-wheel.

### 10.8 GitLab Package Registry כ-PyPI

```bash
# publish לGitLab Package Registry
uv publish \
  --publish-url https://gitlab.internal.company.com/api/v4/projects/${CI_PROJECT_ID}/packages/pypi \
  --username gitlab-ci-token \
  --password ${CI_JOB_TOKEN}
```

**Group-level index:**
```toml
[[tool.uv.index]]
name = "gitlab-group"
url = "https://gitlab.internal.company.com/api/v4/groups/{group_id}/-/packages/pypi/simple"
default = true
```

> ⚠️ כבו "Forward PyPI package requests to PyPI.org" בהגדרות הGroup!

---

## 11. מפת דרכים ותכנון מיגרציה

### 11.1 ציר זמן (8 שבועות)

```
שבוע 1-2: תשתית ו-spike
    ├── בניית Docker image עם UV + NX + Node.js
    ├── הגדרת Verdaccio לnpm
    ├── הגדרת Artifactory לPython
    ├── spike: הרצת @nxlv/python על פרויקט pilot בסביבת dev
    └── אימות שכל הtools זמינים ברשת הסגורה

שבוע 3-4: NX + @nxlv/python
    ├── npm install + nx.json עם plugins
    ├── nx generate לכל פרויקט (או המרה ידנית)
    ├── בדיקת project graph
    ├── בדיקת affected
    ├── הגדרת MinIO לcache
    └── CI pipeline ראשוני

שבוע 5: UV
    ├── בחירת pilot project
    ├── המרת pyproject.toml
    ├── בדיקה עם uv sync
    ├── המרת שאר הפרויקטים
    └── עדכון CI לUV

שבוע 6: RUFF
    ├── ruff.toml גלובלי
    ├── auto-fix commit
    ├── per-project overrides
    └── CI integration + Code Quality

שבוע 7: Custom Formatter
    ├── tools/format.py
    ├── NX targets
    └── CI integration

שבוע 8: Release + polish
    ├── הגדרת NX Release + @nxlv/python/release/version-actions
    ├── בדיקת Conventional Commits
    ├── release pipeline
    └── הדרכת צוותים
```

### 11.2 רשימת משימות מפורטת

#### NX + @nxlv/python (שבוע 3-4)

- [ ] `npm install -D nx @nxlv/python @nx/vue @nx/vite`
- [ ] יצירת `nx.json` עם plugin config (`packageManager: "uv"`, `inferDependencies: true`)
- [ ] סריקת כל הפרויקטים, מיפוי ל-lib/app
- [ ] יצירת `project.json` לכל Python project (עם `@nxlv/python` executors)
- [ ] יצירת `project.json` לכל Vue project
- [ ] הוספת `tags` לכל project
- [ ] הרצת `npx nx graph` — אימות graph
- [ ] הרצת `npx nx show projects --affected --base=main` — אימות affected
- [ ] הגדרת MinIO + `@nx/s3-cache`
- [ ] עדכון `.gitlab-ci.yml` לNX affected
- [ ] הגדרת `NX_BASE`/`NX_HEAD` variables
- [ ] בדיקת cache בCI
- [ ] אכיפת module boundaries עם ESLint

#### UV (שבוע 5)

- [ ] התקנת UV binary (ראה סעיף 9.3 לרשת סגורה)
- [ ] בחירת pilot project
- [ ] המרת `pyproject.toml` (Poetry → PEP 621)
- [ ] יצירת `uv.lock` ראשון
- [ ] בדיקת tests
- [ ] המרת כל שאר הפרויקטים
- [ ] הוספת workspace root `pyproject.toml`
- [ ] הגדרת Artifactory index
- [ ] עדכון Dockerfile לCI
- [ ] עדכון `.gitlab-ci.yml` (UV_LINK_MODE=copy!)
- [ ] בדיקת `uv lock --check` job בCI
- [ ] עדכון onboarding docs

#### RUFF (שבוע 6)

- [ ] `uv add --dev ruff` בשורש
- [ ] `ruff.toml` גלובלי
- [ ] `ruff check . --select ALL --statistics` — מצב ראשוני
- [ ] הגדרת rules + ignores
- [ ] `ruff check . --fix --unsafe-fixes` + `ruff format .`
- [ ] **commit גדול עצמאי**: `"chore: apply RUFF formatting (initial)"`
- [ ] **הודעה לצוות** על merge conflicts צפויים בPRs פתוחים
- [ ] per-project `ruff.toml` overrides לפרויקטים שצריכים
- [ ] pre-commit hooks (local לרשת סגורה)
- [ ] הסרת flake8, black, isort
- [ ] `validate:lint` job בCI עם Code Quality artifacts
- [ ] `validate:format` job בCI

#### Custom Formatter (שבוע 7)

- [ ] כתיבת `tools/format.py`
- [ ] כתיבת tests ל-`tools/format.py`
- [ ] הוספת `format` + `format-check` targets לכל `project.json`
- [ ] בדיקה: `npx nx affected -t format-check`
- [ ] אינטגרציה בCI

#### Release (שבוע 8)

- [ ] הגדרת `release` בnx.json עם `versionActions: "@nxlv/python/release/version-actions"`
- [ ] הגדרת `release` ב-project.json של כל lib
- [ ] בדיקת `npx nx release --dry-run`
- [ ] הגדרת `GL_TOKEN` + `GIT_PUSH_TOKEN` ב-GitLab CI variables
- [ ] release pipeline ב-CI (with manual trigger)
- [ ] הדרכת כל הצוותים על Conventional Commits
- [ ] commitlint + pre-commit hook לאכיפת commit format
- [ ] תיעוד תהליך release

---

## 12. בעיות ידועות וfootguns

### 12.1 NX + @nxlv/python

#### plugin לא מזהה UV workspace
**תסמין:** `npx nx graph` לא מציג תלויות בין Python projects.
**סיבה:** `inferDependencies: false` (ברירת מחדל) או `packageManager` לא מוגדר.
**פתרון:**
```json
{
  "plugins": [{
    "plugin": "@nxlv/python",
    "options": {
      "packageManager": "uv",
      "inferDependencies": true
    }
  }]
}
```

#### שימוש ב-`nx:run-commands` במקום `@nxlv/python:run-commands`
**תסמין:** פקודות Python נכשלות עם `command not found` או import errors.
**סיבה:** `nx:run-commands` לא מפעיל את ה-venv. `@nxlv/python:run-commands` כן.
**פתרון:** תמיד השתמשו ב-`@nxlv/python:run-commands` לפקודות Python.

#### `@nxlv/python:build` לא מוצא dependencies
**תסמין:** build נכשל עם missing module.
**סיבה:** `bundleLocalDependencies: false` ו-workspace members לא installed.
**פתרון:** `bundleLocalDependencies: true` או ודאו `uv sync` רץ לפני.

#### circular dependencies בProject Graph
**תסמין:** NX מסרב לרוץ / timeout.
**פתרון:** יצירת `shared-core` package שמכיל קוד משותף.

### 12.2 UV

#### `UV_LINK_MODE` שגוי בCI
**תסמין:** "cross-device link" / "Invalid cross-device link".
**פתרון:** `UV_LINK_MODE=copy` — **חובה** בGitLab CI.

#### conflict בגרסאות בין workspace members
**תסמין:** `uv lock` נכשל עם "conflicting versions".
**פתרון:** packages סותרים לא יכולים לשתף workspace. השתמשו ב-path dependencies.

#### `uv.lock` outdated בCI
**תסמין:** `uv sync --frozen` נכשל.
**סיבה:** מישהו שינה `pyproject.toml` בלי לעדכן `uv.lock`.
**פתרון:** הוסיפו `uv lock --check` job שרץ לפני הsync.

### 12.3 RUFF

#### אלפי שגיאות בהרצה ראשונה
**זה נורמלי.** הרצו `ruff check . --fix --unsafe-fixes` + `ruff format .` בcommit נפרד.

#### merge conflicts אחרי auto-fix
**זה בלתי נמנע.** הודיעו לצוות לפני, בקשו לmerge/rebase PRs פתוחים לאחר ה-commit הגדול.

#### `ISC001` conflict עם formatter
**פתרון:** `ignore = ["ISC001"]` ב-ruff.toml.

### 12.4 Release

#### `@jscutlery/semver` לא עדכן pyproject.toml
**סיבה:** `@jscutlery/semver` הוא לJS/TS בלבד. לPython צריך `@nxlv/python/release/version-actions`.

#### NX Release לא מוצא גרסה נוכחית
**תסמין:** "fallbackCurrentVersionResolver failed".
**פתרון:** הגדירו `fallbackCurrentVersionResolver: "registry"` או וודאו שגרסה בpyproject.toml תקינה.

#### push נכשל בCI (no permissions)
**סיבה:** CI_JOB_TOKEN אין הרשאת push.
**פתרון:** הגדירו `GIT_PUSH_TOKEN` עם deploy token בעל הרשאת `write_repository`.

---

## 13. פתרון בעיות נפוצות

### 13.1 Debug NX

```bash
# הצגת כל הprojects
npx nx show projects

# project ספציפי
npx nx show project shared-utils --json

# graph
npx nx graph

# affected עם output מפורט
npx nx show projects --affected --base=main --head=HEAD --verbose

# ניקוי cache
npx nx reset

# הרצה ללא cache (לdebugging)
npx nx run shared-utils:test --skip-nx-cache

# verbose run
npx nx run shared-utils:build --verbose

# בדיקת target
npx nx run shared-utils:lint --dry-run
```

### 13.2 Debug @nxlv/python

```bash
# בדיקת graph שנבנה מהPlugin
npx nx show project shared-utils --json | jq '.targets'

# בדיקת workspace sync
npx nx sync:check

# apply sync
npx nx sync
```

### 13.3 Debug UV

```bash
uv tree                              # עץ תלויות
uv tree --package api-service        # עץ לpackage ספציפי
uv lock --check                      # lockfile עדכני?
uv sync --verbose                    # verbose sync
uv python list                       # גרסאות Python
uv cache dir                         # מיקום cache
uv cache prune                       # ניקוי cache
uv export --format requirements.txt  # diagnostics
```

### 13.4 Debug RUFF

```bash
uv run ruff check --show-settings          # settings פעילים
uv run ruff check . --statistics           # violations statistics
uv run ruff rule E501                      # הסבר rule
uv run ruff check --select E501 .          # rule ספציפי
uv run ruff check . --show-files           # קבצים שנבדקים
```

### 13.5 Debug GitLab CI

```yaml
# הוסף job לdebugging
debug:env:
  stage: .pre
  script:
    - echo "NX_BASE=$NX_BASE"
    - echo "NX_HEAD=$NX_HEAD"
    - uv --version
    - node --version
    - npx nx --version
    - uv lock --check
    - npx nx show projects
```

---

## סיכום — עצות הזהב

1. **`@nxlv/python` הוא הבסיס** — בלעדיו NX לא מכיר Python. התקינו אותו ראשון, הגדירו `packageManager: "uv"` ו-`inferDependencies: true`.

2. **`@nxlv/python:run-commands` ולא `nx:run-commands`** — תמיד השתמשו בexecutor הייעודי לPython כדי שה-venv יופעל.

3. **`@nxlv/python/release/version-actions` לPython release** — לא `@jscutlery/semver`. זה הכלי הנכון לעדכן `pyproject.toml`.

4. **`UV_LINK_MODE=copy` חובה בGitLab CI** — אל תשכחו. זה הbug הכי נפוץ.

5. **commit גדול לformatting** — RUFF migration דורש commit עצמאי גדול. הודיעו לצוות מראש.

6. **כבו "Forward to PyPI"** ב-Artifactory — לביטחון מלא ברשת סגורה.

7. **pilot project קודם** — אל תמגרו הכל בבת אחת. ספרייה אחת → אמת → הרחב.

8. **Conventional Commits הדרכה** — Release automation מסתמך לחלוטין על commit messages נכונים. אכפו עם pre-commit hook.

---

### מדדי הצלחה

| מדד | מצב נוכחי | יעד |
|-----|-----------|-----|
| זמן CI pipeline (full build) | X דקות | <30% בזכות NX affected |
| זמן dependency install | X שניות | <10% בזכות UV warm cache |
| זמן lint | X שניות | <5% בזכות RUFF |
| Release — זמן + שגיאות | ידני, שגיאות | אוטומטי, סלקטיבי |

---

*מסמך זה נכתב בעזרת מחקר מעמיק בתיעוד הרשמי של NX, @nxlv/python, UV, RUFF ו-GitLab CI. מרץ 2026.*

**מקורות:**
- [@nxlv/python — npm](https://www.npmjs.com/package/@nxlv/python)
- [nx-plugins GitHub (lucasvieirasilva)](https://github.com/lucasvieirasilva/nx-plugins)
- [UV Documentation](https://docs.astral.sh/uv)
- [RUFF Documentation](https://docs.astral.sh/ruff)
- [NX Release Documentation](https://nx.dev/features/manage-releases)
- [@jscutlery/semver](https://github.com/jscutlery/semver)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
