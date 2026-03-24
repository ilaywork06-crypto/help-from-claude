# מדריך מקיף: מודרניזציה של מונורפו Python/Vue עם NX, UV, RUFF ופורמטר מותאם

> **גרסה:** 1.0 | **תאריך:** מרץ 2026 | **עורך:** Claude Sonnet 4.6
> **סביבה:** GitLab CI/CD · רשת סגורה · Python + Vue.js · Monorepo

---

## תוכן עניינים

1. [רקע ומטרות](#1-רקע-ומטרות)
2. [סקירת ארכיטקטורה](#2-סקירת-ארכיטקטורה)
3. [שלב 1 — NX: ניהול חכם של המונורפו](#3-שלב-1--nx-ניהול-חכם-של-המונורפו)
4. [שלב 2 — UV: מיגרציה מ-Poetry](#4-שלב-2--uv-מיגרציה-מ-poetry)
5. [שלב 3 — RUFF: לינטינג ופורמטינג](#5-שלב-3--ruff-לינטינג-ופורמטינג)
6. [שלב 4 — פורמטר מותאם](#6-שלב-4--פורמטר-מותאם)
7. [אינטגרציה עם GitLab CI/CD](#7-אינטגרציה-עם-gitlab-cicd)
8. [עבודה ברשת סגורה](#8-עבודה-ברשת-סגורה)
9. [Release חכם עם NX](#9-release-חכם-עם-nx)
10. [מפת דרכים ותכנון מיגרציה](#10-מפת-דרכים-ותכנון-מיגרציה)
11. [בעיות ידועות וfootguns](#11-בעיות-ידועות-וfootguns)
12. [פתרון בעיות נפוצות](#12-פתרון-בעיות-נפוצות)

---

## 1. רקע ומטרות

### 1.1 המצב הנוכחי

הארגון מפעיל **מונורפו גדול** המשלב:
- עשרות ספריות Python ויישומים
- פרויקטים בVue.js
- תלויות שונות בגרסאות שונות בין הפרויקטים
- Poetry כמנהל חבילות Python
- GitLab CI/CD עם pipelines קיימים
- רשת ארגונית סגורה ללא גישה ישירה לאינטרנט

### 1.2 בעיות קיימות שמובילות לשינוי

| בעיה | השפעה | פתרון מוצע |
|------|--------|------------|
| build כל המונורפו בכל PR | זמן CI ארוך, משאבים מבוזבזים | NX affected |
| Release ידני ולא מדויק | שגיאות אנושיות, גרסאות לא עקביות | NX Release |
| Poetry איטי, lockfile תלוי-פלטפורמה | CI איטי, בעיות הגדרה | UV |
| כלי לינטינג מרובים ולא אחידים | קונפיגורציה מורכבת, ביצועים ירודים | RUFF |
| פורמטינג לא אחיד בין צוותים | קוד לא עקבי, merge conflicts מיותרים | Custom Formatter + RUFF |

### 1.3 יעדי המיגרציה

1. **NX** — הפחתת זמן CI ב-70%+ על ידי build/test רק של מה שהשתנה
2. **UV** — הפחתת זמן install תלויות ב-10-100x
3. **RUFF** — לינטינג ופורמטינג מהיר, אחיד, קל לתחזוקה
4. **Custom Formatter** — אכיפת סגנון קוד ארגוני אחיד
5. כל השינויים — **תאימות מלאה עם GitLab CI/CD ורשת סגורה**

---

## 2. סקירת ארכיטקטורה

### 2.1 מבנה המונורפו המוצע

```
monorepo/
├── nx.json                          ← הגדרות NX גלובליות
├── package.json                     ← NX + Node.js כלים
├── pyproject.toml                   ← UV workspace root
├── uv.lock                          ← lockfile אחיד לכל הפרויקטים
├── .python-version                  ← גרסת Python ברירת מחדל
├── ruff.toml                        ← RUFF גלובלי
├── .gitlab-ci.yml                   ← pipeline ראשי
│
├── packages/                        ← ספריות Python פנימיות
│   ├── shared-utils/
│   │   ├── project.json             ← NX project config
│   │   ├── pyproject.toml           ← UV member
│   │   ├── ruff.toml                ← RUFF override (אופציונלי)
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
    └── release-scripts/
```

### 2.2 שכבות האחריות

```
┌─────────────────────────────────────────────────────┐
│                    GitLab CI/CD                      │
│  (pipelines, artifact management, PyPI registry)    │
└──────────────────────┬──────────────────────────────┘
                       │
┌──────────────────────▼──────────────────────────────┐
│                      NX                              │
│  (project graph, affected, caching, release)        │
└──────┬────────────────────────┬───────────────────┘
       │                        │
┌──────▼──────┐         ┌───────▼──────┐
│     UV      │         │   Vite/Vue   │
│ (Python     │         │ (frontend    │
│  deps mgmt) │         │  toolchain)  │
└──────┬──────┘         └───────┬──────┘
       │                        │
┌──────▼─────────────────────────▼──────┐
│              RUFF + Formatter         │
│    (linting, formatting, code style)  │
└───────────────────────────────────────┘
```

---

## 3. שלב 1 — NX: ניהול חכם של המונורפו

### 3.1 מהו NX?

NX הוא כלי לניהול monorepo שפותח על ידי חברת Nrwl. הוא מציע:

- **Project Graph** — גרף תלויות בין כל הפרויקטים במונורפו
- **Affected Algorithm** — זיהוי אוטומטי אילו פרויקטים הושפעו מכל שינוי
- **Computation Cache** — שמירת תוצאות builds ומניעת עבודה כפולה
- **Release Management** — שחרור גרסאות סלקטיבי עם versioning אוטומטי
- **Executors & Generators** — קונפיגורציה אחידה לכל project types

### 3.2 מושגי יסוד

| מושג | הסבר |
|------|-------|
| **Workspace** | שורש המונורפו כולו |
| **Project** | ספרייה, אפליקציה, או כל יחידה ב-monorepo |
| **Target** | פעולה שניתן להריץ על project (build, test, lint...) |
| **Executor** | הממשה של target (כמו `nx:run-commands`, `@nx/vite:build`) |
| **Generator** | תבנית ליצירת project חדש |
| **Plugin** | חבילה שמוסיפה executors/generators לNX |
| **namedInputs** | קבוצות קבצים שקובעות מתי cache invalidation קורה |
| **Affected** | פרויקטים שהשתנו (ישירות או דרך תלויות) |

### 3.3 התקנת NX

#### דרישות מקדימות
```bash
# Node.js נדרש (LTS מומלץ)
node --version  # >= 18.0

# npm/pnpm (pnpm מומלץ)
npm install -g pnpm
```

#### הוספת NX למונורפו קיים
```bash
# בשורש המונורפו
npx nx@latest init

# או עם pnpm
pnpx nx@latest init
```

הפקודה תנתח את המונורפו הקיים ותיצור:
- `nx.json` — קונפיגורציה גלובלית
- `package.json` — אם לא קיים

#### package.json בשורש
```json
{
  "name": "monorepo",
  "private": true,
  "devDependencies": {
    "nx": "^21.0.0",
    "@nx/python": "^21.0.0",
    "@nxlv/python": "^21.0.0",
    "@nx/vue": "^21.0.0",
    "@nx/vite": "^21.0.0",
    "@jscutlery/semver": "^5.0.0"
  },
  "scripts": {
    "build": "nx run-many -t build",
    "test": "nx run-many -t test",
    "lint": "nx run-many -t lint"
  }
}
```

### 3.4 הגדרת nx.json

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "nxCloudId": "",

  "defaultBase": "main",

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
      "!{projectRoot}/**/*.pyc"
    ],
    "tests": [
      "{projectRoot}/tests/**/*.py",
      "{projectRoot}/pyproject.toml"
    ],
    "frontend": [
      "{projectRoot}/src/**/*.{ts,vue,css,html}",
      "{projectRoot}/package.json",
      "{projectRoot}/vite.config.ts"
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
      "outputs": ["{projectRoot}/.coverage", "{projectRoot}/coverage.xml"]
    },
    "lint": {
      "cache": true,
      "inputs": ["python", "sharedGlobals"],
      "outputs": []
    },
    "format": {
      "cache": false
    }
  },

  "release": {
    "projects": ["packages/*"],
    "projectsRelationship": "independent",
    "releaseTagPattern": "{projectName}@{version}",
    "changelog": {
      "automaticFromRef": true,
      "projectChangelogs": true
    },
    "version": {
      "conventionalCommits": true,
      "generatorOptions": {
        "fallbackCurrentVersionResolver": "registry"
      }
    }
  },

  "plugins": [
    {
      "plugin": "@nxlv/python",
      "options": {
        "buildTargetName": "build",
        "testTargetName": "test",
        "lintTargetName": "lint",
        "publishTargetName": "publish",
        "pyprojectTomlConfigFilePath": "{projectRoot}/pyproject.toml",
        "inferDependencies": true
      }
    },
    {
      "plugin": "@nx/vue/plugin",
      "options": {
        "buildTargetName": "build",
        "testTargetName": "test",
        "serveTargetName": "serve"
      }
    }
  ],

  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "test", "lint"],
        "remoteCache": {
          "enabled": true,
          "url": "https://minio.internal.company.com",
          "bucket": "nx-cache"
        }
      }
    }
  }
}
```

### 3.5 project.json לפרויקט Python

```json
{
  "name": "shared-utils",
  "$schema": "../../node_modules/nx/schemas/project-schema.json",
  "projectType": "library",
  "sourceRoot": "packages/shared-utils/src",
  "tags": ["python", "scope:shared", "type:lib"],

  "targets": {
    "install": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv sync --package shared-utils --frozen",
        "cwd": "{workspaceRoot}"
      }
    },

    "build": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv build --package shared-utils --out-dir dist",
        "cwd": "{workspaceRoot}"
      },
      "dependsOn": ["install"],
      "inputs": ["python", "sharedGlobals"],
      "outputs": ["{projectRoot}/dist"]
    },

    "test": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run --package shared-utils pytest packages/shared-utils/tests -v --cov=src --cov-report=xml",
        "cwd": "{workspaceRoot}"
      },
      "dependsOn": ["install"],
      "inputs": ["python", "tests", "sharedGlobals"],
      "outputs": ["{projectRoot}/coverage.xml", "{projectRoot}/.coverage"]
    },

    "lint": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run ruff check packages/shared-utils/src --output-format=concise",
        "cwd": "{workspaceRoot}"
      },
      "inputs": ["python", "sharedGlobals"],
      "cache": true
    },

    "format": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run python tools/format.py packages/shared-utils",
        "cwd": "{workspaceRoot}"
      }
    },

    "format-check": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run ruff format packages/shared-utils --check",
        "cwd": "{workspaceRoot}"
      },
      "cache": true
    },

    "publish": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv publish --index company-pypi",
        "cwd": "{workspaceRoot}"
      },
      "dependsOn": ["build"]
    },

    "version": {
      "executor": "@jscutlery/semver:version",
      "options": {
        "preset": "conventional-commits",
        "trackDeps": true,
        "commitMessageFormat": "chore(release): {projectName} v{version}"
      }
    }
  }
}
```

### 3.6 project.json לפרויקט Vue.js

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
      "options": {
        "outputPath": "dist/frontend/web-app",
        "configFile": "frontend/web-app/vite.config.ts"
      },
      "inputs": ["frontend", "sharedGlobals"],
      "outputs": ["{workspaceRoot}/dist/frontend/web-app"]
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
      "options": {
        "configFile": "frontend/web-app/vite.config.ts"
      }
    },

    "lint": {
      "executor": "nx:run-commands",
      "options": {
        "command": "npx eslint frontend/web-app/src --ext .ts,.vue",
        "cwd": "{workspaceRoot}"
      }
    }
  }
}
```

### 3.7 NX Affected — הלב של הפעולה

#### כיצד NX מחשב מה "הושפע"

NX בונה **Project Graph** — גרף של כל הפרויקטים ותלויותיהם. לאחר מכן:

1. **מוצא שינויים** בין `NX_BASE` ל-`NX_HEAD` (git diff)
2. **מזהה פרויקטים שנגעו** בקבצים שהשתנו
3. **מפיץ שינויים** דרך הגרף — אם `api-service` תלוי ב-`shared-utils` וזה השתנה, גם `api-service` "הושפע"
4. **מחזיר רשימה** של רק הפרויקטים שצריך לעבד

```bash
# הצגת פרויקטים מושפעים
nx affected:graph

# build רק של מה שהשתנה
nx affected -t build

# test רק של מה שהשתנה
nx affected -t test

# lint רק של מה שהשתנה
nx affected -t lint

# הרצה מרובה במקביל
nx affected -t build,test,lint --parallel=4

# הצגה בלבד
nx show projects --affected --base=main --head=HEAD
```

#### הגדרת namedInputs לדיוק מירבי

```json
{
  "namedInputs": {
    "python": [
      "{projectRoot}/**/*.py",
      "{projectRoot}/pyproject.toml",
      "!{projectRoot}/**/__pycache__/**",
      "!{projectRoot}/**/*.pyc",
      "!{projectRoot}/.pytest_cache/**"
    ],
    "sharedGlobals": [
      "{workspaceRoot}/nx.json",
      "{workspaceRoot}/pyproject.toml",
      "{workspaceRoot}/uv.lock",
      "{workspaceRoot}/ruff.toml",
      "{workspaceRoot}/pyproject.toml"
    ]
  }
}
```

> ⚠️ **אזהרה חשובה:** שינוי ב-`uv.lock` גורם ל-cache invalidation של **כל** הפרויקטים כי הוא ב-`sharedGlobals`. זה נכון — אם תלויות השתנו, כדאי לבדוק מחדש. ניתן לדקדק יותר על ידי העברת `uv.lock` מחוץ ל-`sharedGlobals` ואכיפה per-project, אך זה מורכב יותר.

### 3.8 NX Cache — מניעת עבודה כפולה

#### Cache מקומי (ברירת מחדל)

Cache מאוחסן ב-`.nx/cache` בשורש המונורפו. בכל הרצת target, NX:
1. מחשב hash של הקלט (קבצים, env vars, פרמטרים)
2. בודק אם קיים cache לhash זה
3. אם כן — משחזר תוצאות מהcache (ללא הרצה)
4. אם לא — מריץ ושומר תוצאות לcache

```bash
# ניקוי cache מקומי
nx reset

# הצגת cache stats
nx show projects --with-target=build
```

#### Cache משותף עם MinIO (on-premise)

```bash
npm install -D @nx/s3-cache
```

`nx.json`:
```json
{
  "s3Cache": {
    "endpoint": "https://minio.internal.company.com",
    "bucket": "nx-cache",
    "region": "us-east-1",
    "forcePathStyle": true,
    "disableChecksum": true
  }
}
```

> ⚠️ **CVE-2025-36852 (CREEP):** פגיעות cache poisoning בכל bucket-based caches. **פתרון:** ב-MR pipelines הגדר `localMode: "read-only"`, רק ה-pipeline של `main` כותב לcache.

```json
{
  "s3Cache": {
    "endpoint": "https://minio.internal.company.com",
    "bucket": "nx-cache",
    "forcePathStyle": true,
    "localMode": "${CI_MERGE_REQUEST_IID != null ? 'read-only' : 'read-write'}"
  }
}
```

#### `portable-nx-cache` — אלטרנטיבה ללא תשתית

כלי Go קטן (MIT) שמיישם את OpenAPI spec של NX cache ושומר ב-GitLab CI cache. **אפס תשתית נוספת!**

```yaml
# .gitlab-ci.yml
variables:
  NX_CACHE_DIRECTORY: ".nx/cache"

cache:
  key: "nx-cache-$CI_COMMIT_REF_SLUG"
  paths:
    - .nx/cache/
```

### 3.9 Tags וגבולות מודולים

Tags ב-NX מאפשרים לאכוף **ארכיטקטורה** — מי רשאי לתלות במי.

#### מוסכמות מומלצות לtagging

```json
{
  "tags": ["python", "scope:shared", "type:lib"]
}
```

| מימד | ערכים | משמעות |
|------|-------|---------|
| `scope:` | `shared`, `core`, `feature`, `app` | שכבת שייכות |
| `type:` | `lib`, `app`, `tool`, `e2e` | סוג פרויקט |
| `tech:` | `python`, `vue`, `node` | טכנולוגיה |

#### אכיפת גבולות (עם ESLint)

```bash
npm install -D @nx/eslint-plugin
```

`.eslintrc.json`:
```json
{
  "rules": {
    "@nx/enforce-module-boundaries": [
      "error",
      {
        "depConstraints": [
          {
            "sourceTag": "scope:app",
            "onlyDependOnLibsWithTags": ["scope:feature", "scope:shared", "scope:core"]
          },
          {
            "sourceTag": "scope:feature",
            "onlyDependOnLibsWithTags": ["scope:shared", "scope:core"]
          },
          {
            "sourceTag": "scope:shared",
            "onlyDependOnLibsWithTags": ["scope:core"]
          },
          {
            "sourceTag": "type:app",
            "onlyDependOnLibsWithTags": ["type:lib"]
          }
        ]
      }
    ]
  }
}
```

### 3.10 Plugin @nxlv/python

`@nxlv/python` הוא ה-plugin המרכזי לאינטגרציה בין NX לPython/UV. מגרסה 21.3.0 הוא תומך ב-**automatic dependency inference**:

```bash
npm install -D @nxlv/python
```

הplugin:
1. **סורק `pyproject.toml`** של כל member בworkspace
2. **מזהה תלויות** בין workspace members
3. **בונה את ה-Project Graph** של NX אוטומטית
4. **מספק executors** מוכנים לbuild/test/lint/publish

```json
{
  "plugins": [
    {
      "plugin": "@nxlv/python",
      "options": {
        "inferDependencies": true,
        "pyprojectTomlConfigFilePath": "{projectRoot}/pyproject.toml"
      }
    }
  ]
}
```

### 3.11 Implicit Dependencies

כאשר שינוי בקובץ אחד משפיע על פרויקטים רבים (למשל `nx.json`, `ruff.toml`):

```json
{
  "namedInputs": {
    "sharedGlobals": [
      "{workspaceRoot}/nx.json",
      "{workspaceRoot}/ruff.toml",
      "{workspaceRoot}/pyproject.toml",
      "{workspaceRoot}/uv.lock"
    ]
  }
}
```

לתלויות implicit ישירות בין פרויקטים:
```json
{
  "name": "api-service",
  "implicitDependencies": ["shared-utils", "data-models"]
}
```

---

## 4. שלב 2 — UV: מיגרציה מ-Poetry

### 4.1 מהו UV ולמה לעבור?

UV הוא מנהל חבילות Python של חברת Astral (יוצרי RUFF), כתוב ב-Rust.

| מאפיין | Poetry | UV |
|--------|--------|-----|
| מהירות install (cold) | ~48 שניות | ~7 שניות (6-7x) |
| מהירות install (warm) | ~4 שניות | <1 שנייה |
| lockfile | תלוי פלטפורמה | Universal (Windows/Linux/Mac) |
| Workspace support | אין (אחיד) | כן (Cargo-style) |
| Python management | צריך pyenv | מובנה |
| תאימות PEP | חלקית (`[tool.poetry]`) | מלאה (PEP 621) |
| Build backend | Poetry-core | בחירה חופשית |

### 4.2 UV Workspaces — הלב של האינטגרציה

UV Workspace מאפשר ניהול מונורפו Python עם **lockfile אחד** לכל הפרויקטים.

**`pyproject.toml` בשורש:**
```toml
[project]
name = "monorepo"
version = "0.1.0"
requires-python = ">=3.11"
dependencies = []

[tool.uv.workspace]
members = ["packages/*", "apps/*"]
exclude = ["packages/legacy-*", "packages/deprecated-*"]

[tool.uv.sources]
# כל workspace member זמין לאחרים
shared-utils = { workspace = true }
data-models = { workspace = true }
ml-pipeline = { workspace = true }

[[tool.uv.index]]
name = "company-pypi"
url = "https://pypi.internal.company.com/simple/"
default = true

[tool.uv]
index-strategy = "first-index"
```

**`pyproject.toml` של member:**
```toml
[project]
name = "api-service"
version = "1.0.0"
requires-python = ">=3.11"
dependencies = [
    "fastapi>=0.100.0",
    "shared-utils",           # ← workspace member
    "data-models",            # ← workspace member
]

[tool.uv.sources]
shared-utils = { workspace = true }
data-models = { workspace = true }

[dependency-groups]
dev = [
    "pytest>=7.0",
    "pytest-cov>=4.0",
    "httpx>=0.24",
]
lint = [
    "ruff>=0.9",
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/api_service"]
```

### 4.3 מיגרציה מ-Poetry ל-UV — שלב אחר שלב

#### שלב א: ביצוע inventory

```bash
# מצא כל קבצי pyproject.toml
find . -name "pyproject.toml" -not -path "*/node_modules/*" -not -path "*/.venv/*"

# בדוק אילו package groups קיימים
grep -r "\[tool.poetry.group" .

# בדוק private sources
grep -r "\[\[tool.poetry.source\]\]" .
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
numpy = {version = ">=1.24", optional = true}

[tool.poetry.extras]
data = ["numpy"]

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

[project.optional-dependencies]
data = ["numpy>=1.24"]

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

#### שלב ג: מחיקת lockfile ישן ויצירה חדשה

```bash
# מחק lockfiles ישנים
find . -name "poetry.lock" -delete

# צור lockfile חדש
uv lock

# sync הסביבה
uv sync --all-groups
```

#### שלב ד: בדיקת תאימות

```bash
# וודא שכל ה-imports עובדים
uv run python -c "import my_package"

# הרץ tests
uv run pytest

# בדוק גרסאות
uv tree
```

### 4.4 טבלת המרת פקודות

| Poetry | UV |
|--------|-----|
| `poetry install` | `uv sync` |
| `poetry install --no-dev` | `uv sync --no-dev` |
| `poetry add requests` | `uv add requests` |
| `poetry add --dev pytest` | `uv add --dev pytest` |
| `poetry add --group lint ruff` | `uv add --group lint ruff` |
| `poetry remove requests` | `uv remove requests` |
| `poetry update` | `uv lock --upgrade` |
| `poetry update requests` | `uv lock --upgrade-package requests` |
| `poetry run python script.py` | `uv run python script.py` |
| `poetry build` | `uv build` |
| `poetry publish` | `uv publish` |
| `poetry lock` | `uv lock` |
| `poetry show --tree` | `uv tree` |
| `poetry env use 3.11` | `uv python pin 3.11` |
| `poetry export -f requirements.txt` | `uv export --format requirements.txt` |
| `poetry shell` | `source .venv/bin/activate` |

### 4.5 Private Registry לרשת סגורה

```toml
# pyproject.toml בשורש workspace
[[tool.uv.index]]
name = "company-artifactory"
url = "https://artifactory.internal.company.com/artifactory/api/pypi/python-local/simple"
publish-url = "https://artifactory.internal.company.com/artifactory/api/pypi/python-local"
default = true

[tool.uv]
no-index = false           # false = מאפשר גם PyPI (אם יש גישה)
index-strategy = "first-index"  # מניעת dependency confusion attacks
```

**Authentication (לעולם אל תשים credentials בקוד!):**
```bash
# CI/CD environment variables
export UV_INDEX_COMPANY_ARTIFACTORY_USERNAME="ci-user"
export UV_INDEX_COMPANY_ARTIFACTORY_PASSWORD="${ARTIFACTORY_TOKEN}"
```

**לסביבה ללא גישה לPyPI בכלל:**
```toml
[tool.uv]
no-index = true   # בלוק כל PyPI

[[tool.uv.index]]
name = "internal"
url = "https://pypi.internal.company.com/simple/"
default = true
```

### 4.6 ניהול גרסאות Python

```bash
# התקנת Python (ברשת פתוחה)
uv python install 3.12

# הגדרת גרסה לפרויקט
uv python pin 3.12

# ברשת סגורה — שימוש ב-Python המותקן במערכת
export UV_NO_MANAGED_PYTHON=1
# או בפרויקט:
```

```toml
[tool.uv]
python-preference = "only-system"   # לרשת סגורה
```

### 4.7 uv.lock — ה-Universal Lockfile

`uv.lock` הוא אחד היתרונות הגדולים של UV:

- **Universal** — עובד על כל OS (Linux, macOS, Windows) בקובץ אחד
- **Cross-Python** — עובד עם Python 3.10, 3.11, 3.12 מאותו lockfile
- **Workspace-wide** — lockfile אחד לכל ה-workspace members
- **Deterministic** — כולל hash של כל wheel/sdist

```bash
# יצירה/עדכון
uv lock

# בדיקה שהlockfile עדכני (לCI)
uv lock --check

# install מהlockfile בלבד (לCI — אין שינוי versions)
uv sync --frozen

# upgrade ספציפי
uv lock --upgrade-package requests
```

---

## 5. שלב 3 — RUFF: לינטינג ופורמטינג

### 5.1 מהו RUFF?

RUFF הוא לינטר Python כתוב ב-Rust, פי 10-100 מהיר מהכלים הקיימים. הוא מחליף:

| כלי ישן | מחליף עם RUFF |
|---------|--------------|
| flake8 | `ruff check` |
| pylint | `ruff check` (חלקי) |
| isort | `ruff check --select I` |
| black | `ruff format` |
| pyupgrade | `ruff check --select UP` |
| bandit | `ruff check --select S` |
| pydocstyle | `ruff check --select D` |

### 5.2 הגדרת RUFF גלובלי

**`ruff.toml` בשורש המונורפו:**
```toml
# ruff.toml — גלובלי לכל המונורפו
target-version = "py311"
line-length = 100

[lint]
select = [
    "E",   # pycodestyle errors
    "W",   # pycodestyle warnings
    "F",   # pyflakes
    "I",   # isort
    "B",   # flake8-bugbear
    "C4",  # flake8-comprehensions
    "UP",  # pyupgrade
    "N",   # pep8-naming
    "S",   # flake8-bandit (security)
    "ANN", # flake8-annotations
    "PTH", # flake8-use-pathlib
    "SIM", # flake8-simplify
    "RUF", # ruff-specific
]
ignore = [
    "E501",   # line too long (מטופל על ידי formatter)
    "ANN101", # missing type annotation for self
    "ANN102", # missing type annotation for cls
    "S101",   # use of assert (בסדר בtests)
]

[lint.per-file-ignores]
"tests/**/*.py" = ["S101", "ANN", "D"]
"**/migrations/**/*.py" = ["E", "W", "F"]

[lint.isort]
known-first-party = ["shared_utils", "data_models", "ml_pipeline"]
force-sort-within-sections = true

[lint.pydocstyle]
convention = "google"

[format]
quote-style = "double"
indent-style = "space"
skip-magic-trailing-comma = false
line-ending = "lf"           # חשוב! תמיד LF גם ב-Windows
```

### 5.3 RUFF per-project override

כל פרויקט יכול לרשת ולדרוס:

**`packages/my-lib/ruff.toml`:**
```toml
extend = "../../ruff.toml"   # ירושה מהשורש

[lint]
extend-select = ["D"]        # הוסף docstring checks
extend-ignore = ["UP007"]    # ignore שלא רלוונטי כאן

[lint.per-file-ignores]
"src/my_lib/internal/**" = ["D", "ANN"]
```

### 5.4 מיגרציה מ-flake8/black/isort

#### שלב א: הסרת כלים ישנים

```toml
# הסר מ-pyproject.toml:
# [tool.black]
# [tool.isort]
# [tool.flake8]
```

```bash
uv remove black isort flake8 flake8-bugbear
```

#### שלב ב: המרת קונפיגורציה

**`.flake8` הישן:**
```ini
[flake8]
max-line-length = 100
extend-ignore = E203, W503
exclude = .git, __pycache__, dist, build
per-file-ignores =
    tests/*: S101
```

**RUFF שקול:**
```toml
line-length = 100
[lint]
ignore = ["E203", "W503"]
extend-exclude = [".git", "__pycache__", "dist", "build"]
[lint.per-file-ignores]
"tests/**" = ["S101"]
```

**`setup.cfg`/`pyproject.toml` עם isort:**
```toml
# ישן
[tool.isort]
profile = "black"
known_first_party = ["mypackage"]

# RUFF שקול
[tool.ruff.lint.isort]
known-first-party = ["mypackage"]
force-sort-within-sections = true
```

#### שלב ג: הרצת auto-fix ראשונית

```bash
# תיקון אוטומטי של כל מה שאפשר
uv run ruff check . --fix --unsafe-fixes

# פורמוט קוד
uv run ruff format .

# commit הכל בcommit נפרד: "chore: apply RUFF formatting"
```

### 5.5 RUFF ב-pre-commit

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.9.10
    hooks:
      - id: ruff
        args: [--fix, --exit-non-zero-on-fix]
      - id: ruff-format
```

> ⚠️ **ברשת סגורה:** הגדר `repo: local` עם script מקומי במקום GitHub URL.

**pre-commit local לרשת סגורה:**
```yaml
repos:
  - repo: local
    hooks:
      - id: ruff-check
        name: RUFF lint
        language: system
        entry: uv run ruff check --fix
        types: [python]

      - id: ruff-format
        name: RUFF format
        language: system
        entry: uv run ruff format
        types: [python]
```

### 5.6 false positives נפוצים ופתרונות

| שגיאה | סיבה | פתרון |
|-------|-------|--------|
| `B008` ב-FastAPI | `Depends()` כ-default arg | `# noqa: B008` או ignore בper-file |
| `D401` | "First line should be in imperative mood" | `ignore = ["D401"]` |
| `ANN` ב-tests | annotations בtests | `per-file-ignores: "tests/**" = ["ANN"]` |
| `S101` | assert | ignore בtest files |
| `UP007` | Optional[X] vs X | ב-Python < 3.10 |
| `ISC001` | implicit string concat | conflict עם formatter |

### 5.7 RUFF כ-formatter בלבד

```bash
# פורמוט בלבד (כמו black)
uv run ruff format .

# בדיקה בלבד (ל-CI)
uv run ruff format --check .

# הצגת diff
uv run ruff format --diff .
```

---

## 6. שלב 4 — פורמטר מותאם

### 6.1 הסקריפט

**`tools/format.py`:**
```python
#!/usr/bin/env python3
"""Custom formatter script.

Usage:
    python tools/format.py <directory>
    python tools/format.py packages/my-lib
    python tools/format.py .  # כל המונורפו
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
    parser.add_argument(
        "directory",
        type=Path,
        help="Directory to format",
    )
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
    parser.add_argument(
        "--verbose",
        "-v",
        action="store_true",
        help="Verbose output",
    )
    return parser.parse_args()


def validate_directory(directory: Path) -> None:
    if not directory.exists():
        print(f"Error: Directory '{directory}' does not exist.", file=sys.stderr)
        sys.exit(1)
    if not directory.is_dir():
        print(f"Error: '{directory}' is not a directory.", file=sys.stderr)
        sys.exit(1)


def run_ruff_format(directory: Path, check: bool, diff: bool, verbose: bool) -> int:
    cmd = ["ruff", "format", str(directory)]

    if check:
        cmd.append("--check")
    if diff:
        cmd.append("--diff")
    if verbose:
        cmd.append("--verbose")

    if verbose:
        print(f"Running: {' '.join(cmd)}")

    result = subprocess.run(cmd, capture_output=False)
    return result.returncode


def run_ruff_lint_fix(directory: Path, verbose: bool) -> int:
    """Run RUFF lint with auto-fix for import sorting and simple fixes."""
    cmd = [
        "ruff", "check", str(directory),
        "--select", "I,F401,UP",  # isort + unused imports + pyupgrade
        "--fix",
    ]

    if verbose:
        print(f"Running: {' '.join(cmd)}")

    result = subprocess.run(cmd, capture_output=False)
    return result.returncode


def main() -> None:
    args = parse_args()

    validate_directory(args.directory)

    print(f"Formatting '{args.directory}'...")

    # שלב 1: תיקון imports (isort-style)
    if not args.check and not args.diff:
        rc = run_ruff_lint_fix(args.directory, args.verbose)
        if rc != 0:
            print("Warning: Some lint fixes could not be applied automatically.")

    # שלב 2: פורמוט (black-style)
    rc = run_ruff_format(args.directory, args.check, args.diff, args.verbose)

    if rc == 0:
        if args.check:
            print(f"✓ '{args.directory}' is properly formatted.")
        else:
            print(f"✓ '{args.directory}' formatted successfully.")
    else:
        if args.check:
            print(f"✗ '{args.directory}' needs formatting. Run without --check to fix.")
        else:
            print(f"✗ Formatting failed for '{args.directory}'.")

    sys.exit(rc)


if __name__ == "__main__":
    main()
```

### 6.2 שימוש

```bash
# פורמוט ספרייה
uv run python tools/format.py packages/shared-utils

# בדיקה בלבד (לCI)
uv run python tools/format.py packages/shared-utils --check

# הצגת diff
uv run python tools/format.py packages/shared-utils --diff

# פורמוט כל המונורפו
uv run python tools/format.py .

# verbose
uv run python tools/format.py packages/api-service --verbose
```

### 6.3 אינטגרציה ב-NX

ב-`project.json` של כל project:
```json
{
  "targets": {
    "format": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run python tools/format.py {projectRoot}",
        "cwd": "{workspaceRoot}"
      }
    },
    "format-check": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run python tools/format.py {projectRoot} --check",
        "cwd": "{workspaceRoot}"
      },
      "cache": true,
      "inputs": ["python"]
    }
  }
}
```

```bash
# פורמוט כל הprojcts המושפעים
nx affected -t format

# בדיקת פורמוט בCI
nx affected -t format-check
```

---

## 7. אינטגרציה עם GitLab CI/CD

### 7.1 משתני סביבה קריטיים

```yaml
variables:
  # UV
  UV_LINK_MODE: "copy"           # חובה! GitLab CI אינו תומך ב-hardlinks
  UV_CACHE_DIR: ".uv-cache"
  UV_FROZEN: "true"              # לCI: אל תשנה lockfile
  UV_NO_MANAGED_PYTHON: "1"      # לרשת סגורה: השתמש ב-Python המותקן
  UV_PYTHON_DOWNLOADS: "never"   # לרשת סגורה: אל תנסה להוריד Python

  # NX
  NX_BASE: "${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}"
  NX_HEAD: "$CI_COMMIT_SHA"
  NX_BRANCH: "$CI_COMMIT_REF_NAME"
  NX_NO_CLOUD: "true"            # לרשת סגורה: ללא NX Cloud

  # Credentials
  UV_INDEX_COMPANY_PYPI_USERNAME: "${COMPANY_PYPI_USER}"
  UV_INDEX_COMPANY_PYPI_PASSWORD: "${COMPANY_PYPI_TOKEN}"
```

> ⚠️ **`UV_LINK_MODE: "copy"` הוא קריטי!** GitLab CI יוצר mountpoint נפרד לתיקיית הbuild. hard links אינם עובדים בין mountpoints שונים. ללא הגדרה זו, UV ייכשל.

### 7.2 `.gitlab-ci.yml` מלא

```yaml
# .gitlab-ci.yml

include:
  - local: '.gitlab/ci/templates.yml'

variables:
  PYTHON_VERSION: "3.12"
  NODE_VERSION: "20"
  UV_VERSION: "0.11.0"
  UV_LINK_MODE: "copy"
  UV_CACHE_DIR: ".uv-cache"
  UV_FROZEN: "true"
  UV_NO_MANAGED_PYTHON: "1"
  UV_INDEX_COMPANY_PYPI_USERNAME: "${ARTIFACTORY_USER}"
  UV_INDEX_COMPANY_PYPI_PASSWORD: "${ARTIFACTORY_TOKEN}"
  NX_BASE: "${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}"
  NX_HEAD: "$CI_COMMIT_SHA"

default:
  image: registry.internal.company.com/ci/python-nx:latest
  tags:
    - docker
    - internal
  cache:
    - key:
        files:
          - uv.lock
        prefix: "uv-cache-${CI_JOB_IMAGE}"
      paths:
        - ${UV_CACHE_DIR}
      policy: pull
    - key: "nx-cache-${CI_COMMIT_REF_SLUG}"
      paths:
        - .nx/cache/
      policy: pull

stages:
  - setup
  - validate
  - test
  - build
  - release

# ─────────────────────────────────────────
# SETUP
# ─────────────────────────────────────────
setup:deps:
  stage: setup
  cache:
    - key:
        files:
          - uv.lock
        prefix: "uv-cache-${CI_JOB_IMAGE}"
      paths:
        - ${UV_CACHE_DIR}
      policy: pull-push
  script:
    - uv sync --all-groups --frozen
  after_script:
    - uv cache prune --ci
  artifacts:
    paths:
      - .venv/
    expire_in: 1 hour

# ─────────────────────────────────────────
# VALIDATE
# ─────────────────────────────────────────
validate:lint:
  stage: validate
  needs: [setup:deps]
  script:
    - |
      AFFECTED=$(npx nx show projects --affected --base=$NX_BASE --head=$NX_HEAD | tr '\n' ',')
      if [ -z "$AFFECTED" ]; then
        echo "No affected Python projects. Skipping lint."
        exit 0
      fi
      npx nx affected -t lint --base=$NX_BASE --head=$NX_HEAD --parallel=4
  artifacts:
    reports:
      codequality: ruff-code-quality.json
    when: always

validate:format-check:
  stage: validate
  needs: [setup:deps]
  script:
    - npx nx affected -t format-check --base=$NX_BASE --head=$NX_HEAD --parallel=4

validate:lockfile:
  stage: validate
  script:
    - uv lock --check
  rules:
    - changes:
        - "**/pyproject.toml"

# ─────────────────────────────────────────
# TEST
# ─────────────────────────────────────────
test:unit:
  stage: test
  needs: [setup:deps]
  script:
    - npx nx affected -t test --base=$NX_BASE --head=$NX_HEAD --parallel=2
  artifacts:
    reports:
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
      junit: "**/test-results.xml"
    when: always

test:frontend:
  stage: test
  needs: []
  script:
    - npm ci
    - npx nx affected -t test --base=$NX_BASE --head=$NX_HEAD --parallel=4
  rules:
    - changes:
        - "frontend/**/*"

# ─────────────────────────────────────────
# BUILD
# ─────────────────────────────────────────
build:packages:
  stage: build
  needs: [validate:lint, validate:format-check, test:unit]
  script:
    - npx nx affected -t build --base=$NX_BASE --head=$NX_HEAD --parallel=4
  artifacts:
    paths:
      - "**/dist/"
    expire_in: 1 week
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG

build:frontend:
  stage: build
  needs: [test:frontend]
  script:
    - npm ci
    - npx nx affected -t build --base=$NX_BASE --head=$NX_HEAD
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
    - if: $CI_COMMIT_TAG

# ─────────────────────────────────────────
# RELEASE
# ─────────────────────────────────────────
release:version:
  stage: release
  needs: [build:packages]
  script:
    - git config user.email "ci@company.com"
    - git config user.name "GitLab CI"
    - npx nx affected -t version --base=$NX_BASE --head=$NX_HEAD
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual

release:publish:
  stage: release
  needs: [release:version]
  script:
    - npx nx affected -t publish --base=$NX_BASE --head=$NX_HEAD
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
  environment:
    name: production
```

### 7.3 RUFF ב-GitLab Code Quality

```bash
# יצירת Code Quality report לGitLab
uv run ruff check . --output-format=gitlab --output-file=ruff-code-quality.json
```

ב-`.gitlab-ci.yml`:
```yaml
validate:lint:
  script:
    - uv run ruff check . --output-format=gitlab --output-file=ruff-code-quality.json
  artifacts:
    reports:
      codequality: ruff-code-quality.json
    when: always
```

תוצאות יופיעו ישירות בדף ה-MR של GitLab.

### 7.4 Dynamic Child Pipelines עם NX Affected

לmono-repos גדולים, כדאי ליצור pipeline דינמי:

```yaml
# .gitlab-ci.yml
generate:pipeline:
  stage: .pre
  script:
    - |
      AFFECTED_PROJECTS=$(npx nx show projects --affected \
        --base=$NX_BASE --head=$NX_HEAD \
        --json | jq -r '.[]')

      python3 scripts/generate-pipeline.py \
        --projects "$AFFECTED_PROJECTS" \
        --output generated-pipeline.yml
  artifacts:
    paths:
      - generated-pipeline.yml

trigger:dynamic:
  stage: .pre
  trigger:
    include:
      - artifact: generated-pipeline.yml
        job: generate:pipeline
    strategy: depend
  needs: [generate:pipeline]
```

### 7.5 Dockerfile ל-CI Runner פנימי

```dockerfile
FROM python:3.12-slim-bookworm

# UV
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /usr/local/bin/

# Node.js (לNX)
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs \
    && npm install -g pnpm

# NX globally
RUN npm install -g nx

# פרטי חברה
RUN pip config set global.index-url https://pypi.internal.company.com/simple/

# ENV
ENV UV_LINK_MODE=copy \
    UV_NO_MANAGED_PYTHON=1 \
    UV_PYTHON_DOWNLOADS=never \
    PATH="/root/.cargo/bin:$PATH"

WORKDIR /workspace
```

> **טיפ:** בנה image זה פעם אחת ושמור ב-GitLab Container Registry הפנימי. כך CI לא צריך להוריד כלים בכל פעם.

---

## 8. עבודה ברשת סגורה

### 8.1 אתגרי רשת סגורה

| רכיב | בעיה | פתרון |
|------|-------|--------|
| UV binary | לא ניתן להוריד מהאינטרנט | העברה ידנית / Docker |
| NX/npm packages | npm registry לא נגיש | Verdaccio/Nexus mirror |
| Python packages | PyPI לא נגיש | Artifactory/Nexus/devpi |
| Docker images | Docker Hub לא נגיש | GitLab Container Registry |
| Python downloads | UV מנסה להוריד Python | `UV_PYTHON_DOWNLOADS=never` |

### 8.2 הגדרת Artifactory כ-PyPI mirror

```toml
# pyproject.toml בשורש
[[tool.uv.index]]
name = "artifactory"
url = "https://artifactory.internal.company.com/artifactory/api/pypi/pypi-virtual/simple"
default = true

[tool.uv]
no-index = false                    # אפשר להשתמש בindexes אחרים אם צריך
index-strategy = "first-index"      # מניעת dependency confusion
```

**הגדרת Artifactory (Admin):**
1. צור remote repository מ-PyPI: `pypi-remote`
2. צור local repository: `pypi-local`
3. צור virtual repository: `pypi-virtual` שמשלב שניהם
4. **כבה** "Forward PyPI package requests to PyPI.org" לביטחון מלא

### 8.3 הגדרת Verdaccio כ-npm mirror

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

middlewares:
  audit:
    enabled: true
```

**הגדרת npm לרשת סגורה:**
```bash
npm config set registry https://verdaccio.internal.company.com/
```

### 8.4 UV ב-air-gapped (ללא גישה לאינטרנט)

```bash
# שלב 1: הכן UV binary על מכונה עם אינטרנט
# הורד מ: https://github.com/astral-sh/uv/releases
# דגם: uv-x86_64-unknown-linux-gnu.tar.gz

# שלב 2: העבר לשרת הפנימי
scp uv-x86_64-unknown-linux-gnu.tar.gz internal-server:/opt/tools/

# שלב 3: חלץ והוסף ל-PATH
tar -xzf uv-x86_64-unknown-linux-gnu.tar.gz
cp uv /usr/local/bin/

# שלב 4: הגדר environment variables
export UV_NO_MANAGED_PYTHON=1
export UV_PYTHON_DOWNLOADS=never
```

**`uv.toml` לסביבת airgap:**
```toml
[tool.uv]
no-index = false
python-preference = "only-system"
python-downloads = "never"

[[tool.uv.index]]
name = "internal"
url = "https://pypi.internal.company.com/simple/"
default = true
```

### 8.5 pre-populate UV cache

```bash
# על מכונה עם אינטרנט:
uv sync --all-groups
uv cache dir  # מציג מיקום cache

# העבר cache לשרת הפנימי:
tar -czf uv-cache.tar.gz ~/.cache/uv/

# ב-GitLab CI artifact או shared storage
```

### 8.6 Docker images לרשת סגורה

**Script להעברת images:**
```bash
#!/bin/bash
# mirror-images.sh

INTERNAL_REGISTRY="registry.internal.company.com"
IMAGES=(
    "ghcr.io/astral-sh/uv:latest"
    "ghcr.io/astral-sh/ruff:latest"
    "python:3.12-slim"
    "node:20-alpine"
)

for image in "${IMAGES[@]}"; do
    # Pull
    docker pull "$image"

    # Re-tag
    internal_name="${INTERNAL_REGISTRY}/mirrors/$(echo $image | tr '/:' '-')"
    docker tag "$image" "$internal_name"

    # Push
    docker push "$internal_name"

    echo "Mirrored: $image -> $internal_name"
done
```

### 8.7 GitLab Runner לרשת סגורה

```toml
# /etc/gitlab-runner/config.toml
[[runners]]
  name = "internal-runner"
  url = "https://gitlab.internal.company.com/"
  token = "${RUNNER_TOKEN}"
  executor = "docker"

  [runners.docker]
    image = "registry.internal.company.com/ci/python-nx:latest"
    pull_policy = ["if-not-present", "never"]  # אל תנסה Docker Hub
    volumes = ["/cache"]

  [runners.cache]
    Type = "s3"
    Shared = true
    [runners.cache.s3]
      ServerAddress = "minio.internal.company.com"
      AccessKey = "${MINIO_ACCESS_KEY}"
      SecretKey = "${MINIO_SECRET_KEY}"
      BucketName = "gitlab-runner-cache"
      Insecure = false
```

---

## 9. Release חכם עם NX

### 9.1 אסטרטגיית Release

בmonorepo גדול, יש שתי אסטרטגיות עיקריות:

| אסטרטגיה | מתאים כאשר | חיסרון |
|-----------|------------|---------|
| **Fixed versioning** | כל הספריות תמיד בגרסה זהה | release "מיותר" לספריות שלא השתנו |
| **Independent versioning** | כל ספרייה גרסה משלה | מורכבות גבוהה יותר |

**המלצה לכם:** Independent versioning — כי יש לכם "המון ספריות" עם תלויות שונות.

### 9.2 הגדרת NX Release

**`nx.json`:**
```json
{
  "release": {
    "projects": ["packages/*"],
    "projectsRelationship": "independent",
    "releaseTagPattern": "{projectName}@{version}",

    "changelog": {
      "automaticFromRef": true,
      "projectChangelogs": {
        "createRelease": "gitlab",
        "file": "{projectRoot}/CHANGELOG.md"
      }
    },

    "version": {
      "conventionalCommits": true,
      "generatorOptions": {
        "fallbackCurrentVersionResolver": "registry"
      }
    },

    "git": {
      "commit": true,
      "commitMessage": "chore(release): {projectName} v{version}",
      "tag": true,
      "tagMessage": "{projectName} v{version}",
      "push": true
    }
  }
}
```

### 9.3 Conventional Commits — המנגנון לautomation

NX Release קורא את commit messages ומחשב את bump הגרסה:

| Commit prefix | גרסה |
|---------------|------|
| `feat:` | minor (0.1.0 → 0.2.0) |
| `fix:` | patch (0.1.0 → 0.1.1) |
| `feat!:` / `BREAKING CHANGE:` | major (0.1.0 → 1.0.0) |
| `chore:`, `docs:`, `style:` | ללא bump |

**דוגמאות:**
```bash
git commit -m "feat(api-service): add pagination endpoint"
# → api-service minor bump

git commit -m "fix(shared-utils): handle None values in parser"
# → shared-utils patch bump

git commit -m "feat!: remove deprecated authentication method"
# → major bump לכל affected projects
```

### 9.4 Release של ספריות ספציפיות

```bash
# Release ספרייה ספציפית
npx nx release --projects=shared-utils

# Release קבוצת ספריות
npx nx release --projects=shared-utils,data-models

# Release לפי tag
npx nx release --projects=tag:scope:shared

# Preview בלבד (dry run)
npx nx release --projects=api-service --dry-run

# Release עם version ידני
npx nx release version --projects=shared-utils --specifier=2.0.0
```

### 9.5 @jscutlery/semver לGitLab Integration

`@jscutlery/semver` הוא plugin שמוסיף שחרור ל-GitLab releases:

```bash
npm install -D @jscutlery/semver
```

**`project.json`:**
```json
{
  "targets": {
    "version": {
      "executor": "@jscutlery/semver:version",
      "options": {
        "preset": "conventional-commits",
        "trackDeps": true,
        "commitMessageFormat": "chore(release): ${projectName} v${version}"
      }
    },
    "github": {
      "executor": "@jscutlery/semver:gitlab",
      "options": {
        "tag": "${version}",
        "notes": "${notes}"
      }
    }
  }
}
```

**`trackDeps: true`** — כאשר `shared-utils` מקבלת גרסה חדשה, `api-service` שתלויה בה תקבל אוטומטית patch bump.

### 9.6 Release Pipeline ב-GitLab

```yaml
# .gitlab-ci.yml
release:
  stage: release
  script:
    - git config user.email "ci-bot@company.com"
    - git config user.name "GitLab CI Bot"

    # מצא affected projects שהשתנו
    - |
      AFFECTED=$(npx nx show projects --affected \
        --base=last-release-tag \
        --head=$CI_COMMIT_SHA \
        --select=tag:type:lib)

    # Release רק אותם
    - |
      if [ -n "$AFFECTED" ]; then
        npx nx release \
          --projects=$AFFECTED \
          --base-branch=main \
          --yes
      fi
  rules:
    - if: $CI_COMMIT_BRANCH == $CI_DEFAULT_BRANCH
      when: manual
  environment:
    name: production

  variables:
    GIT_USER_EMAIL: "ci-bot@company.com"
    GL_TOKEN: "${GITLAB_RELEASE_TOKEN}"
```

### 9.7 GitLab Package Registry כ-PyPI

GitLab מציע Package Registry מובנה שיכול לשמש כ-PyPI פרטי:

```bash
# publish לGitLab Package Registry
uv publish \
  --publish-url https://gitlab.internal.company.com/api/v4/projects/${CI_PROJECT_ID}/packages/pypi \
  --username gitlab-ci-token \
  --password ${CI_JOB_TOKEN}
```

**index URL לgroup:**
```
https://gitlab.internal.company.com/api/v4/groups/{group_id}/-/packages/pypi/simple
```

> ⚠️ **קריטי:** כבה "Forward PyPI package requests to PyPI.org" בהגדרות הGroup ב-GitLab. ללא זה, packages שאינם ב-GitLab Registry "נופלים" ל-PyPI.org בשקט!

---

## 10. מפת דרכים ותכנון מיגרציה

### 10.1 ציר זמן מוצע (8 שבועות)

```
שבוע 1-2: הכנה ותשתית
    ├── Setup NX בסביבת dev
    ├── בניית Docker image ל-CI
    ├── הגדרת MinIO לNX cache
    └── הכנת Verdaccio + Artifactory mirrors

שבוע 3-4: NX
    ├── הוספת nx.json לשורש
    ├── יצירת project.json לכל פרויקט
    ├── בדיקת affected graph
    ├── אינטגרציה ראשונית עם GitLab CI
    └── בדיקות cache

שבוע 5: UV
    ├── המרת pyproject.toml בפרויקט pilot
    ├── בדיקת workspace members
    ├── המרת שאר הפרויקטים
    └── עדכון CI pipelines לUV

שבוע 6: RUFF
    ├── הוספת ruff.toml גלובלי
    ├── הרצת auto-fix ראשוני
    ├── commit מסודר של formatttng
    ├── הוספת pre-commit hooks
    └── אינטגרציה ב-CI

שבוע 7: Custom Formatter
    ├── כתיבת tools/format.py
    ├── הוספה ל-NX targets
    └── בדיקות

שבוע 8: Release + Polish
    ├── הגדרת NX Release
    ├── בדיקת Conventional Commits
    ├── release pipeline ב-GitLab
    └── תיעוד + הדרכת צוותים
```

### 10.2 רשימת משימות מפורטת

#### NX — שבוע 3-4

- [ ] `npm install -D nx@latest @nxlv/python @nx/vue @nx/vite @jscutlery/semver`
- [ ] יצירת `nx.json` עם namedInputs ו-targetDefaults
- [ ] סריקת כל הפרויקטים וסיווגם (lib/app/tool)
- [ ] יצירת `project.json` לכל פרויקט Python
- [ ] יצירת `project.json` לכל פרויקט Vue
- [ ] הגדרת tags לכל פרויקט
- [ ] הרצת `nx graph` לאימות הגרף
- [ ] הרצת `nx affected --dry-run` לאימות affected
- [ ] הגדרת MinIO ו-`@nx/s3-cache`
- [ ] עדכון `.gitlab-ci.yml` לשימוש ב-NX affected
- [ ] הוספת `NX_BASE`/`NX_HEAD` variables
- [ ] בדיקת cache בCI

#### UV — שבוע 5

- [ ] התקנת UV בסביבת dev (ראה 8.4 לרשת סגורה)
- [ ] בחירת פרויקט pilot למיגרציה ראשונה
- [ ] המרת `pyproject.toml` של ה-pilot (Poetry → UV)
- [ ] יצירת `uv.lock` ראשון
- [ ] בדיקת tests על ה-pilot
- [ ] המרת שאר הפרויקטים (batch)
- [ ] הוספת workspace root `pyproject.toml`
- [ ] הגדרת `[[tool.uv.index]]` לArtifactory
- [ ] עדכון Dockerfile של CI runner
- [ ] עדכון `.gitlab-ci.yml` (poetry install → uv sync)
- [ ] הוספת `UV_LINK_MODE=copy` ל-CI
- [ ] בדיקות integration מלאות
- [ ] עדכון תיעוד פנימי (README, onboarding)

#### RUFF — שבוע 6

- [ ] `uv add --dev ruff` בשורש הworkspace
- [ ] יצירת `ruff.toml` גלובלי
- [ ] הרצת `ruff check . --select ALL --statistics` לראות מצב
- [ ] הגדרת rules מתאימות (ראה סעיף 5.2)
- [ ] הרצת `ruff check . --fix --unsafe-fixes` לתיקון ראשוני
- [ ] הרצת `ruff format .`
- [ ] `git commit -m "chore: apply RUFF formatting (initial)"` — commit גדול, עצמאי
- [ ] הוספת `ruff.toml` per-project override לפרויקטים שצריכים
- [ ] הגדרת pre-commit hooks
- [ ] הסרת כלים ישנים (flake8, black, isort)
- [ ] עדכון CI targets (lint, format-check)
- [ ] הגדרת GitLab Code Quality reporting

#### Custom Formatter — שבוע 7

- [ ] כתיבת `tools/format.py`
- [ ] בדיקות ל-`tools/format.py`
- [ ] הוספת target `format` לכל `project.json`
- [ ] הוספת target `format-check` לכל `project.json`
- [ ] אינטגרציה ב-CI pipeline

#### Release — שבוע 8

- [ ] הגדרת `release` ב-`nx.json`
- [ ] בדיקת Conventional Commits history
- [ ] הגדרת `version` target בכל `project.json`
- [ ] בדיקת `npx nx release --dry-run`
- [ ] הגדרת GitLab integration (tokens, permissions)
- [ ] release pipeline ב-CI
- [ ] הדרכת כל הצוותים על Conventional Commits
- [ ] תיעוד תהליך release

---

## 11. בעיות ידועות וfootguns

### 11.1 NX — בעיות נפוצות

#### בעיה: שינוי ב-uv.lock מאפס cache לכולם
**תופעה:** כל שינוי בתלויות כלשהן גורם לbuild מחדש של **כל** הפרויקטים.
**סיבה:** `uv.lock` נמצא ב-`sharedGlobals` namedInput.
**פתרון:** זה כוונתי. אם תלויות השתנו, כדאי לבדוק. אפשר לדקדק:

```json
{
  "namedInputs": {
    "sharedGlobals": [
      "{workspaceRoot}/nx.json",
      "{workspaceRoot}/ruff.toml"
      // הסר uv.lock מכאן אם אתה בטוח
    ]
  }
}
```

#### בעיה: circular dependencies בProject Graph
**תופעה:** `nx graph` מציג לולאה / NX מסרב לרוץ.
**סיבה:** Project A תלוי ב-Project B וProject B תלוי ב-Project A.
**פתרון:** שבור את המעגל על ידי יצירת `Project C` שמחזיק את הקוד המשותף.

#### בעיה: affected מחשב יותר מדי / פחות מדי
**תופעה:** NX מריץ tests לפרויקטים שלא השתנו, או לא מריץ לכאלה שכן.
**סיבה:** `namedInputs` לא מוגדרים נכון, או `NX_BASE` שגוי.
**פתרון:**
```bash
# בדוק מה NX חושב שהשתנה
git diff $NX_BASE $NX_HEAD --name-only

# בדוק project graph
npx nx graph --affected --base=$NX_BASE --head=$NX_HEAD
```

#### בעיה: cache hit אבל tests נכשלים
**תופעה:** CI מציג "cache hit" אבל הקוד בפועל שבור.
**סיבה:** cache poisoning (CVE-2025-36852) — מישהו כתב תוצאה שגויה לcache.
**פתרון:** הגבל כתיבה לcache ל-main branch בלבד.

#### בעיה: project.json לא מוכר ב-NX
**תופעה:** `npx nx show projects` לא מציג project.
**סיבה:** `project.json` לא ב-path שNX סורק, או שגיאת syntax.
**פתרון:**
```bash
# בדוק שNX מוצא את הprojext
npx nx show projects --verbose

# בדוק syntax של project.json
cat packages/my-lib/project.json | python -m json.tool
```

### 11.2 UV — בעיות נפוצות

#### בעיה: conflict בגרסאות בין workspace members
**תופעה:** `uv lock` נכשל עם "conflicting versions".
**סיבה:** Package A צריך `numpy==1.24` ו-Package B צריך `numpy==2.0`.
**פתרון:**
```toml
# אל תשים packages סותרים באותו workspace
# השתמש ב-path dependencies במקום:
[tool.uv.sources]
package-a = { path = "../package-a", editable = true }
```

#### בעיה: `uv lock` איטי בפעם הראשונה
**תופעה:** resolution לוקח הרבה זמן.
**סיבה:** UV צריך להוריד metadata של כל הpackages לראשונה.
**פתרון:** pre-populate cache (ראה 8.5), ולאחר מכן warm cache מהיר מאוד.

#### בעיה: `UV_LINK_MODE` שגוי ב-CI
**תופעה:** שגיאת "cross-device link" או "Invalid cross-device link".
**פתרון:** `export UV_LINK_MODE=copy` — **חובה** ב-GitLab CI!

#### בעיה: Python version mismatch
**תופעה:** `uv sync` מציין Python 3.11 נדרש אבל 3.10 זמין.
**סיבה:** `requires-python` לא עקבי בין members.
**פתרון:**
```bash
# בדוק intersection
uv python list --only-installed
# הגדר .python-version בשורש
echo "3.12" > .python-version
```

#### בעיה: Poetry scripts שנשכחו
**תופעה:** `[tool.poetry.scripts]` לא הועבר ל-`[project.scripts]`.
**תסמין:** CLI commands נעלמו לאחר מיגרציה.
**פתרון:** חפש `grep -r "tool.poetry.scripts" .` ועדכן.

### 11.3 RUFF — בעיות נפוצות

#### בעיה: אלפי שגיאות בהרצה ראשונה
**תופעה:** `ruff check .` מחזיר אלפי violations.
**פתרון:** זה נורמלי! הרץ `ruff check . --fix --unsafe-fixes` ו-`ruff format .` בcommit נפרד. אל תנסה לתקן הכל ידנית.

#### בעיה: conflict בין rules שונים
**תופעה:** rule A מבקש לשנות X, rule B מבקש את ההיפך.
**דוגמה:** `ISC001` (implicit string concatenation) עם formatter.
**פתרון:** הוסף ל-ignore:
```toml
[lint]
ignore = ["ISC001"]  # conflict עם formatter
```

#### בעיה: FastAPI `Depends()` מעורר B008
**תופעה:** `B008: Do not perform function call 'Depends' in default argument`.
**פתרון:**
```toml
[lint.per-file-ignores]
"**/routers/**/*.py" = ["B008"]
"**/dependencies.py" = ["B008"]
```

#### בעיה: pre-commit hook רץ על כל הrepo בכל פעם
**תסמין:** pre-commit איטי מאוד.
**פתרון:** RUFF תומך ב-`pass_filenames: true` וירוץ רק על קבצים שהשתנו:
```yaml
- id: ruff
  pass_filenames: true
  types: [python]
```

### 11.4 GitLab CI — בעיות נפוצות

#### בעיה: `NX_BASE` שגוי בCI
**תופעה:** `nx affected` מריץ את הכל תמיד, או לא מריץ כלום.
**סיבה:** `CI_MERGE_REQUEST_DIFF_BASE_SHA` ריק מחוץ ל-MR, `CI_COMMIT_BEFORE_SHA` ריק ב-push הראשון.
**פתרון:**
```bash
# במקרה שNX_BASE ריק, השתמש ב-HEAD~1
NX_BASE="${CI_MERGE_REQUEST_DIFF_BASE_SHA:-${CI_COMMIT_BEFORE_SHA:-$(git rev-parse HEAD~1)}}"
```

#### בעיה: cache לא עובד בין branches
**תופעה:** cache חם בbranch main, קר בfeature branches.
**פתרון:** הגדר cache בsharing:
```yaml
cache:
  key: "nx-cache"  # מפתח גלובלי, לא per-branch
  paths:
    - .nx/cache/
  policy: pull  # feature branches רק קוראים
```

#### בעיה: uv.lock מסומן כ"dirty" ב-CI
**תופעה:** CI נכשל כי `uv.lock` השתנה.
**סיבה:** CI מריץ `uv sync` (לא `--frozen`) ומעדכן lockfile.
**פתרון:** השתמש תמיד ב-`uv sync --frozen` ב-CI.

---

## 12. פתרון בעיות נפוצות

### 12.1 Debug NX

```bash
# הצגת כל הprojects
npx nx show projects

# הצגת project ספציפי
npx nx show project my-lib

# הצגת project graph
npx nx graph

# הצגת affected
npx nx show projects --affected --base=main --head=HEAD

# debug cache
npx nx reset  # נקה cache מקומי

# verbose run
npx nx run my-lib:build --verbose

# skip cache (לdebugging)
npx nx run my-lib:build --skip-nx-cache
```

### 12.2 Debug UV

```bash
# עץ תלויות
uv tree

# עץ עבור package ספציפי
uv tree --package api-service

# בדיקת lockfile
uv lock --check

# verbose sync
uv sync --verbose

# info על Python
uv python list

# cache location
uv cache dir

# ניקוי cache
uv cache clean

# export לrequirements.txt (לdiagnostics)
uv export --format requirements.txt > /tmp/deps.txt
```

### 12.3 Debug RUFF

```bash
# הצגת כל הrules הפעילים
uv run ruff check --show-settings

# בדיקת rule ספציפי
uv run ruff check --select E501 .

# explain rule
uv run ruff rule E501

# statistics על violations
uv run ruff check . --statistics

# אילו קבצים נבדקים
uv run ruff check . --show-files
```

### 12.4 Debug GitLab CI

```bash
# בדוק variables
echo "NX_BASE=${NX_BASE}"
echo "NX_HEAD=${NX_HEAD}"

# בדוק UV config
uv pip list 2>/dev/null || echo "Not in project context"
uv --version

# בדוק NX
npx nx --version
npx nx show projects
```

---

## סיכום ועצות מפתח

### עצות הזהב

1. **NX affected הוא המטרה העיקרית** — שאר הכלים חשובים, אבל החיסכון הגדול ביותר בCI הוא NX affected. השקיעו בnamedInputs נכונים.

2. **commit גדול לformatting** — כאשר מטמיעים RUFF, עשו commit אחד גדול עם `"chore: apply RUFF formatting"`. זה ישמור על git history נקי. הסבירו לצוות מראש שזה יגרום ל-merge conflicts בPRs פתוחים.

3. **UV_LINK_MODE=copy — אל תשכחו** — ללא זה, UV יכשל בGitLab CI. הגדירו בdefault variables ב-`gitlab-ci.yml`.

4. **private registry ראשון** — לפני שאתם מתקינים כלום, ודאו שכל הpackages שאתם צריכים נמצאים ב-Artifactory. בדקו שהForward לPyPI כבוי.

5. **uv lock --check בCI** — הוסיפו job ש-validates שהlockfile עדכני. זה ימנע drift בין lockfile ל-pyproject.toml.

6. **conventional commits הדרכה** — הצלחת NX Release תלויה ב-commit messages נכונים. הדריכו את כל הצוות ואכפו עם pre-commit hook.

7. **pilot project קודם** — אל תמגרו את כל המונורפו בבת אחת. בחרו ספרייה אחת, עשו בה כל המיגרציה, בדקו שהכל עובד, ואז הרחיבו.

8. **cache read-only ב-MRs** — הגנה מפני CVE-2025-36852. רק main branch יכתוב לcache.

### מדדי הצלחה

| מדד | לפני | יעד |
|-----|------|-----|
| זמן CI pipeline (full) | X דקות | <30% מX (NX affected) |
| זמן install תלויות | X שניות | <10% מX (UV warm cache) |
| זמן lint | X שניות | <5% מX (RUFF) |
| Release ידני/שבועי | Manual | אוטומטי + סלקטיבי |
| false positives בCI | ? | 0 (RUFF מדויק) |

---

*מסמך זה עודכן לאחרונה: מרץ 2026. הכלים מתפתחים במהירות — בדקו תמיד את הdocs הרשמיים לגרסאות עדכניות.*

**מקורות:**
- [NX Documentation](https://nx.dev)
- [UV Documentation](https://docs.astral.sh/uv)
- [RUFF Documentation](https://docs.astral.sh/ruff)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [@nxlv/python Plugin](https://www.npmjs.com/package/@nxlv/python)
- [@jscutlery/semver](https://github.com/jscutlery/semver)
