# מדריך מקיף: מעבר מ-Poetry ל-UV, הטמעת NX, RUFF ופורמטר מותאם אישית במונורפו גדול

> **קהל יעד:** מפתחים ומנהלי DevOps בארגון גדול הפועל ברשת סגורה (air-gapped / private network), עם מונורפו Python מבוסס GitLab CI/CD.
>
> **סדר עדיפויות:** NX → UV → RUFF → Custom Formatter

---

## תוכן עניינים

1. [רקע ומוטיבציה](#1-רקע-ומוטיבציה)
2. [מפת דרכים - סקירה כללית](#2-מפת-דרכים---סקירה-כללית)
3. [שלב א׳: NX — מנהל המשימות החכם של המונורפו](#3-שלב-א׳-nx--מנהל-המשימות-החכם-של-המונורפו)
4. [שלב ב׳: UV — מנהל החבילות החדש](#4-שלב-ב׳-uv--מנהל-החבילות-החדש)
5. [שלב ג׳: RUFF — לינטר ופורמטר מהיר](#5-שלב-ג׳-ruff--לינטר-ופורמטר-מהיר)
6. [שלב ד׳: פורמטר Python מותאם אישית](#6-שלב-ד׳-פורמטר-python-מותאם-אישית)
7. [אינטגרציה מלאה: NX + UV + RUFF ב-GitLab CI/CD](#7-אינטגרציה-מלאה-nx--uv--ruff-ב-gitlab-cicd)
8. [קונפיגורציה של מונורפו: הירארכיה ושיתוף הגדרות](#8-קונפיגורציה-של-מונורפו-הירארכיה-ושיתוף-הגדרות)
9. [ניהול Release חכם עם NX](#9-ניהול-release-חכם-עם-nx)
10. [עבודה ברשת סגורה (Air-Gapped)](#10-עבודה-ברשת-סגורה-air-gapped)
11. [בעיות ידועות ומלכודות נפוצות](#11-בעיות-ידועות-ומלכודות-נפוצות)
12. [checklist מלא לכל שלב](#12-checklist-מלא-לכל-שלב)

---

## 1. רקע ומוטיבציה

### מדוע אנחנו עושים את זה?

המונורפו שלנו גדל לגודל שבו הכלים הישנים כבר לא מספיקים:

| בעיה | מצב נוכחי (Poetry) | מצב רצוי |
|------|---------------------|-----------|
| מהירות התקנת dependencies | איטית (pip-based) | 10-100x מהיר יותר עם UV |
| ניהול builds ותלויות בין packages | ידני / scripts | אוטומטי עם NX affected |
| לינטינג ופורמטינג | flake8 + black + isort (איטי) | RUFF: הכל כלי אחד, מהיר פי 100 |
| Release חכם | ידני / custom scripts | `nx release` עם changelog אוטומטי |
| שימוש חוזר ב-CI cache | מוגבל | NX computation cache + GitLab cache |

### מה כל כלי עושה?

```
┌─────────────────────────────────────────────────────────────┐
│                        MONOREPO                             │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  NX - אורקסטרציה של משימות, graph, affected, release │  │
│  └──────────────────────────────────────────────────────┘  │
│                                                             │
│  ┌────────────────────┐    ┌──────────────────────────┐    │
│  │  UV - ניהול חבילות │    │  RUFF - לינטר + פורמטר  │    │
│  │  workspaces        │    │  (מחליף flake8/black)    │    │
│  └────────────────────┘    └──────────────────────────┘    │
│                                                             │
│  ┌──────────────────────────────────────────────────────┐  │
│  │  Custom Formatter Script - סקריפט Python מותאם      │  │
│  └──────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. מפת דרכים - סקירה כללית

### לוח זמנים מומלץ

```
שבוע 1-2:  הטמעת NX על המונורפו הקיים
שבוע 3-4:  מעבר מ-Poetry ל-UV (migration מלא)
שבוע 5:    הטמעת RUFF (החלפת flake8/black/isort)
שבוע 6:    פורמטר מותאם אישית + אינטגרציה מלאה
```

### מבנה מונורפו מומלץ לאחר המעבר

```
monorepo/
├── nx.json                          # קונפיגורציית NX ראשית
├── pyproject.toml                   # UV workspace root + shared RUFF config
├── uv.lock                          # lock file משותף לכל ה-workspace
├── .pre-commit-config.yaml          # hooks גלובליים
├── .gitlab-ci.yml                   # CI/CD pipeline
├── tools/
│   └── format.py                    # Custom formatter script
├── packages/
│   ├── lib-a/
│   │   ├── pyproject.toml           # UV package config
│   │   ├── project.json             # NX project config
│   │   └── src/lib_a/
│   ├── lib-b/
│   │   ├── pyproject.toml
│   │   ├── project.json
│   │   └── src/lib_b/
│   └── service-x/
│       ├── pyproject.toml
│       ├── project.json
│       └── src/service_x/
└── apps/
    ├── api-server/
    │   ├── pyproject.toml
    │   ├── project.json
    │   └── src/
    └── worker/
        ├── pyproject.toml
        ├── project.json
        └── src/
```

---

## 3. שלב א׳: NX — מנהל המשימות החכם של המונורפו

### 3.1 מה זה NX ומדוע אנחנו צריכים אותו?

NX הוא **build system חכם למונורפו**. הוא פותר שלוש בעיות קריטיות:

1. **Project Graph** — NX מנתח אוטומטית את התלויות בין הפרויקטים במונורפו ובונה גרף.
2. **Affected Commands** — במקום להריץ tests/builds על כל הקוד, NX מזהה רק מה השתנה ומה תלוי בו.
3. **Computation Cache** — NX לא מריץ משימות שכבר הורצו עם אותם inputs — הוא מחזיר את התוצאה מהcache.
4. **`nx release`** — ניהול versioning, changelog, ו-publishing חכם לפי שינויי git.

### 3.2 התקנת NX

**דרישות מוקדמות:** Node.js 18+ (NX עצמו כתוב ב-TypeScript אבל תומך בכל שפה כולל Python).

```bash
# התקנה גלובלית
npm install -g nx

# או עבור הרצה חד-פעמית
npx nx@latest --version
```

**הוספת NX למונורפו קיים:**

```bash
# בתיקיית root של המונורפו
npx nx@latest init
```

פקודה זו יוצרת:
- `nx.json` — קונפיגורציית workspace
- `package.json` (אם לא קיים) עם `nx` כ-dev dependency

### 3.3 קונפיגורציית `nx.json` מלאה

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "defaultBase": "main",
  "namedInputs": {
    "default": [
      "{projectRoot}/**/*",
      "sharedGlobals"
    ],
    "production": [
      "default",
      "!{projectRoot}/**/*.spec.py",
      "!{projectRoot}/**/*.test.py",
      "!{projectRoot}/tests/**/*",
      "!{projectRoot}/**/*.md"
    ],
    "sharedGlobals": [
      "{workspaceRoot}/pyproject.toml",
      "{workspaceRoot}/uv.lock",
      "{workspaceRoot}/ruff.toml"
    ]
  },
  "targetDefaults": {
    "lint": {
      "cache": true,
      "inputs": ["default", "^production"],
      "executor": "nx:run-commands"
    },
    "format": {
      "cache": true,
      "inputs": ["default"],
      "executor": "nx:run-commands"
    },
    "format-check": {
      "cache": true,
      "inputs": ["default"],
      "executor": "nx:run-commands"
    },
    "test": {
      "cache": true,
      "inputs": ["default", "^production"],
      "outputs": ["{projectRoot}/coverage"],
      "executor": "nx:run-commands",
      "dependsOn": ["^build"]
    },
    "build": {
      "cache": true,
      "inputs": ["production", "^production"],
      "outputs": ["{projectRoot}/dist"],
      "executor": "nx:run-commands",
      "dependsOn": ["^build"]
    },
    "release": {
      "executor": "nx:run-commands"
    }
  },
  "release": {
    "projects": ["packages/*", "apps/*"],
    "projectsRelationship": "independent",
    "releaseTagPattern": "{projectName}@{version}",
    "changelog": {
      "automaticFromRef": true,
      "projectChangelogs": true,
      "workspaceChangelog": {
        "createRelease": false
      }
    },
    "version": {
      "conventionalCommits": true,
      "generatorOptions": {
        "packageRoot": "{projectRoot}",
        "currentVersionResolver": "git-tag"
      }
    }
  },
  "plugins": [],
  "parallel": 4
}
```

### 3.4 קונפיגורציית `project.json` לכל Package

כל package במונורפו צריך `project.json` המגדיר את ה-targets שלו:

```json
{
  "$schema": "../../node_modules/nx/schemas/project-schema.json",
  "name": "lib-a",
  "projectType": "library",
  "root": "packages/lib-a",
  "sourceRoot": "packages/lib-a/src",
  "tags": ["scope:shared", "type:library"],
  "targets": {
    "lint": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run ruff check packages/lib-a",
        "cwd": "{workspaceRoot}"
      },
      "inputs": [
        "{projectRoot}/**/*.py",
        "{workspaceRoot}/ruff.toml",
        "{workspaceRoot}/pyproject.toml"
      ]
    },
    "lint-fix": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run ruff check --fix packages/lib-a",
        "cwd": "{workspaceRoot}"
      }
    },
    "format": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run ruff format packages/lib-a",
        "cwd": "{workspaceRoot}"
      },
      "inputs": [
        "{projectRoot}/**/*.py",
        "{workspaceRoot}/ruff.toml"
      ]
    },
    "format-check": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run ruff format --check packages/lib-a",
        "cwd": "{workspaceRoot}"
      },
      "inputs": [
        "{projectRoot}/**/*.py",
        "{workspaceRoot}/ruff.toml"
      ]
    },
    "test": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run pytest packages/lib-a/tests --cov=packages/lib-a/src --cov-report=xml:packages/lib-a/coverage/coverage.xml",
        "cwd": "{workspaceRoot}"
      },
      "outputs": ["{projectRoot}/coverage"],
      "inputs": [
        "{projectRoot}/**/*.py",
        "{workspaceRoot}/uv.lock"
      ]
    },
    "build": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv build --package lib-a",
        "cwd": "{workspaceRoot}"
      },
      "outputs": ["{projectRoot}/dist"],
      "dependsOn": ["^build"]
    },
    "type-check": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run mypy packages/lib-a/src",
        "cwd": "{workspaceRoot}"
      }
    }
  }
}
```

### 3.5 פקודות NX בשימוש יומיומי

```bash
# הרצת target על פרויקט ספציפי
nx run lib-a:lint
nx run lib-a:test
nx run lib-a:build

# הרצה על כל הפרויקטים
nx run-many -t lint
nx run-many -t lint test build

# הרצה רק על מה שהשתנה (ה-feature הכי חשוב!)
nx affected -t lint
nx affected -t lint test build

# הצגת project graph
nx graph

# הצגת affected projects
nx graph --affected

# הצגת project graph לפרויקט ספציפי
nx show project lib-a --web

# dry-run לראות מה ירוץ
nx affected -t build --dry-run
```

### 3.6 כיצד עובד ה-Affected Algorithm?

```
git diff main..HEAD → רשימת קבצים שהשתנו
         ↓
NX project graph → איזה packages מכילים את הקבצים האלה?
         ↓
transitive dependencies → אילו packages תלויים בpackages שהשתנו?
         ↓
affected set → הפרויקטים שצריך לבדוק/לבנות
```

**דוגמה:**

```
lib-shared ← lib-a ← service-x
                   ← api-server
```

אם שינינו קוד ב-`lib-shared`:
- `nx affected -t test` ירוץ על: `lib-shared`, `lib-a`, `service-x`, `api-server`
- אם שינינו רק ב-`lib-a`: ירוץ על `lib-a`, `service-x`, `api-server`

### 3.7 module boundary enforcement

NX מאפשר אכיפת גבולות ארכיטקטוניים דרך tags:

```json
// packages/shared-utils/project.json
{
  "tags": ["scope:shared", "type:library"]
}

// apps/api-server/project.json
{
  "tags": ["scope:backend", "type:app"]
}
```

ניתן להגדיר בהמשך כללי ESLint (עבור JS) או כלל conformance (עבור Python) שיאכפו ש-`scope:backend` לא יוכל לייבא מ-`scope:frontend`.

---

## 4. שלב ב׳: UV — מנהל החבילות החדש

### 4.1 מה זה UV ומדוע לעבור?

UV הוא כלי לניהול Python packages ו-projects שנבנה ב-Rust על ידי Astral (אותה חברה שעומדת מאחורי RUFF). הוא מחליף:

| כלי ישן | UV equivalent |
|---------|---------------|
| `poetry install` | `uv sync` |
| `poetry add package` | `uv add package` |
| `poetry shell` | `uv run` (ללא צורך ב-activate) |
| `pyenv install 3.11` | `uv python install 3.11` |
| `pip install` | `uv pip install` |
| `virtualenv` | מובנה ב-UV |
| `pip-tools` | `uv lock` + `uv export` |

**יתרונות מרכזיים:**
- **10-100x מהיר** יותר מ-pip ו-Poetry בהתקנת dependencies
- **Universal lockfile** — קובץ `uv.lock` אחד לכל ה-workspace
- **Workspace support** (כמו Cargo ב-Rust) — מושלם למונורפו
- **dependency groups** — הפרדה בין dev, test, lint dependencies
- **Python version management** — ניהול גרסאות Python ללא pyenv

### 4.2 התקנת UV

**ברשת פתוחה:**
```bash
# Linux/macOS
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows PowerShell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"

# דרך pip (אם pip זמין)
pip install uv
```

**ברשת סגורה (air-gapped):**

UV מפרסם binary releases ב-GitHub. הורד מראש:
```
https://github.com/astral-sh/uv/releases/latest
```

העתק את ה-binary לשרת הפנימי והוסף ל-PATH:
```bash
# Linux
cp uv /usr/local/bin/uv
chmod +x /usr/local/bin/uv

# וידוא
uv --version
```

### 4.3 הגדרת UV Workspace (מונורפו)

**`pyproject.toml` ברמת ה-root:**

```toml
# pyproject.toml (workspace root)
[project]
name = "monorepo-root"
version = "0.0.1"
description = "Monorepo workspace root"
requires-python = ">=3.11"

# ==============================
# UV Workspace Definition
# ==============================
[tool.uv.workspace]
members = [
    "packages/*",
    "apps/*"
]
exclude = [
    "packages/legacy-*",     # ספריות legacy שעוד לא מיגרנו
    "packages/experiments/*"  # תיקיות ניסיוניות
]

# ==============================
# Private Package Registry
# ==============================
[[tool.uv.index]]
name = "internal-registry"
url = "https://pypi.internal.company.com/simple/"
default = true    # זה ה-index הראשי שלנו, לא PyPI

[[tool.uv.index]]
name = "pypi"
url = "https://pypi.org/simple/"
# explicit = true  # uncomment אם רוצים שחבילות ציבוריות יהיו opt-in בלבד

# ==============================
# Dev Tools (global workspace)
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

**`pyproject.toml` לכל package בתוך ה-workspace:**

```toml
# packages/lib-a/pyproject.toml
[project]
name = "lib-a"
version = "1.2.0"
description = "Shared utilities library"
requires-python = ">=3.11"
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.7",
    "lib-b",   # תלות בpackage אחר ב-workspace
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "pytest-cov>=5.0",
]
lint = [
    "ruff>=0.9.0",
]

# Internal workspace dependency
[tool.uv.sources]
lib-b = { workspace = true }

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/lib_a"]
```

### 4.4 פקודות UV בשימוש יומיומי

```bash
# ========================
# ניהול dependencies
# ========================

# הוספת dependency לpackage ספציפי
uv add httpx --package lib-a

# הוספת dev dependency
uv add --dev pytest --package lib-a

# הסרת dependency
uv remove httpx --package lib-a

# עדכון dependency לגרסה חדשה
uv lock --upgrade-package httpx

# ========================
# התקנה וסנכרון
# ========================

# התקנת כל ה-workspace
uv sync

# התקנת package ספציפי
uv sync --package lib-a

# התקנה ללא dev dependencies (production)
uv sync --no-dev

# ========================
# הרצת פקודות
# ========================

# הרצה ב-context של ה-workspace
uv run python -m pytest packages/lib-a

# הרצה ב-context של package ספציפי
uv run --package lib-a python -m pytest

# הרצת script
uv run tools/format.py packages/lib-a

# ========================
# ניהול Python versions
# ========================

# התקנת Python 3.12
uv python install 3.12

# רשימת גרסאות זמינות
uv python list

# הגדרת Python לפרויקט
uv python pin 3.12

# ========================
# Build ו-Publish
# ========================

# בניית package
uv build --package lib-a

# פרסום ל-private registry
uv publish --index internal-registry --package lib-a
```

### 4.5 Migration מ-Poetry ל-UV

#### שלב 1: הכנה

```bash
# גיבוי קיים
cp pyproject.toml pyproject.toml.poetry.bak
cp poetry.lock poetry.lock.bak

# ייצוא requirements מ-Poetry (לצורך השוואה)
poetry export -f requirements.txt --output requirements-poetry.txt
poetry export -f requirements.txt --dev --output requirements-dev-poetry.txt
```

#### שלב 2: שינוי `pyproject.toml`

**לפני (Poetry):**
```toml
[tool.poetry]
name = "lib-a"
version = "1.0.0"
description = "My library"

[tool.poetry.dependencies]
python = "^3.11"
httpx = "^0.27"
pydantic = "^2.7"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0"
black = "^24.0"
flake8 = "^7.0"
isort = "^5.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

**אחרי (UV / PEP 621):**
```toml
[project]
name = "lib-a"
version = "1.0.0"
description = "My library"
requires-python = ">=3.11"
dependencies = [
    "httpx>=0.27",
    "pydantic>=2.7",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "ruff>=0.9.0",  # מחליף את black + flake8 + isort
]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/lib_a"]
```

**שינויים עיקריים:**
- `[tool.poetry]` → `[project]` (PEP 621 standard)
- `python = "^3.11"` → `requires-python = ">=3.11"`
- `[tool.poetry.group.dev.dependencies]` → `[dependency-groups]` dev
- build backend: `poetry-core` → `hatchling` (או `flit-core`, `setuptools`)
- גרסאות: `^0.27` (Poetry syntax) → `>=0.27` (PEP 440)

#### שלב 3: יצירת UV lockfile

```bash
# בתיקיית ה-root של המונורפו
uv lock

# בדיקה שהתקנה עובדת
uv sync

# השוואה עם requirements ישן
uv export > requirements-uv.txt
diff requirements-poetry.txt requirements-uv.txt
```

#### שלב 4: עדכון scripts ב-CI/CD

```yaml
# לפני (Poetry)
before_script:
  - pip install poetry
  - poetry install

# אחרי (UV)
before_script:
  - pip install uv   # או: copy binary
  - uv sync --frozen
```

#### שלב 5: עדכון ה-Makefile / scripts המקומיים

```bash
# לפני
poetry run pytest
poetry run black .
poetry run flake8 .

# אחרי
uv run pytest
uv run ruff format .
uv run ruff check .
```

### 4.6 הגדרת private registry ברשת סגורה

```toml
# pyproject.toml (root workspace)

# אם כל החבילות הן פנימיות - רק registry פנימי
[[tool.uv.index]]
name = "internal"
url = "https://pypi.internal.company.com/simple/"
default = true
authenticate = "always"

# אם צריך גם חבילות ציבוריות
[[tool.uv.index]]
name = "pypi"
url = "https://pypi.org/simple/"
```

**authentication ב-CI:**
```bash
# environment variables
export UV_INDEX_INTERNAL_USERNAME="ci-user"
export UV_INDEX_INTERNAL_PASSWORD="$CI_REGISTRY_TOKEN"

# או ב-.netrc
echo "machine pypi.internal.company.com login ci-user password $TOKEN" >> ~/.netrc
```

**הגדרה ב-`uv.toml` (קובץ קונפיגורציה נוסף, לא ב-repo):**
```toml
# /etc/uv/uv.toml (שרת CI)
[index]
url = "https://pypi.internal.company.com/simple/"
authenticate = "always"

[network]
offline = false
retries = 3
```

### 4.7 אזהרות ומלכודות במעבר מ-Poetry

#### מלכודת 1: גרסאות Python

Poetry תומך ב-`python = "^3.11"` שפירושו `>=3.11, <4.0`.
UV/PEP 440 משתמש ב-`requires-python = ">=3.11"`.

**חשוב:** UV מאכף שכל ה-workspace members ישתמשו ב-intersection של `requires-python`. אם ל-`lib-a` יש `>=3.11` ול-`lib-b` יש `>=3.10`, UV ידרוש `>=3.11`.

#### מלכודת 2: build backend

Poetry השתמש ב-`poetry-core` כ-build backend הייחודי שלו. לאחר המעבר:
- עבור ספריות פשוטות: `hatchling` (מומלץ)
- עבור ספריות עם C extensions: `setuptools`
- עבור ספריות minimalistic: `flit-core`

```toml
# hatchling - הפשוט ביותר
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/my_package"]
```

#### מלכודת 3: optional dependencies

```toml
# Poetry
[tool.poetry.extras]
redis = ["redis"]

# UV / PEP 621
[project.optional-dependencies]
redis = ["redis>=5.0"]

# התקנה עם extras:
# uv add "lib-a[redis]"
```

#### מלכודת 4: poetry scripts

```toml
# Poetry
[tool.poetry.scripts]
my-cli = "my_package.cli:main"

# UV / PEP 621
[project.scripts]
my-cli = "my_package.cli:main"
```

#### מלכודת 5: workspace members עם גרסאות שונות של אותה dependency

```
lib-a מצריך: httpx>=0.27
service-x מצריך: httpx>=0.25,<0.27
```

UV ינסה למצוא גרסה שמתאימה לשניהם, ואם לא ייתכן — יכשל עם שגיאת resolution.
**פתרון:** היו explict בגבולות הגרסאות, ועדכנו את ה-constraints שיהיו תואמים.

---

## 5. שלב ג׳: RUFF — לינטר ופורמטר מהיר

### 5.1 מה זה RUFF ומדוע לעבור?

RUFF הוא לינטר ופורמטר Python שנבנה ב-Rust, מהיר פי 10-100 מכל הכלים הקיימים:

| כלי | RUFF equivalent | מהירות יחסית |
|-----|----------------|--------------|
| `flake8` | `ruff check` | ~100x מהיר |
| `black` | `ruff format` | ~50x מהיר |
| `isort` | `ruff check --select I` | ~100x מהיר |
| `pylint` | `ruff check --select PL` | ~1000x מהיר |
| `pydocstyle` | `ruff check --select D` | ~50x מהיר |
| `pyupgrade` | `ruff check --select UP` | ~50x מהיר |
| `autoflake` | `ruff check --select F` | ~50x מהיר |
| `bandit` | `ruff check --select S` | ~50x מהיר |

**ציטוטים אמיתיים:**
> "On our 250k LOC codebase, pylint takes 2.5 minutes while Ruff takes 0.4 seconds" — Lee Byron, GraphQL co-creator

> "Ruff is so fast that sometimes I add an intentional bug just to confirm it's actually running" — Sebastián Ramírez, creator of FastAPI

### 5.2 הבנת מערכת ה-Rules

RUFF מארגן חוקים לפי prefix:

```
F    - Pyflakes (שגיאות Python בסיסיות)
E    - pycodestyle errors
W    - pycodestyle warnings
I    - isort (מיון imports)
B    - flake8-bugbear (bugs נפוצים)
UP   - pyupgrade (Python syntax מודרני)
C4   - flake8-comprehensions
N    - pep8-naming
D    - pydocstyle (docstrings)
S    - flake8-bandit (security)
PL   - Pylint rules
RUF  - Ruff-specific rules
ANN  - flake8-annotations (type hints)
SIM  - flake8-simplify
TCH  - flake8-type-checking
PT   - flake8-pytest-style
PERF - Perflint (performance)
```

### 5.3 קונפיגורציית RUFF מלאה

**קובץ `ruff.toml` ב-root של המונורפו:**

```toml
# ruff.toml (workspace root)
# ==============================
# הגדרות כלליות
# ==============================
line-length = 88       # תואם ל-Black
indent-width = 4
target-version = "py311"
respect-gitignore = true

# ==============================
# קבצים לדלג עליהם
# ==============================
exclude = [
    ".git",
    ".venv",
    "__pycache__",
    "*.egg-info",
    "dist",
    "build",
    ".tox",
    ".eggs",
    "node_modules",
    "migrations",          # Django/Alembic migrations
    "**/generated/**",     # קוד שנוצר אוטומטית
    "**/_vendor/**",       # vendored code
]

# ==============================
# הגדרות Linter
# ==============================
[lint]
# חוקים מופעלים
select = [
    "E",      # pycodestyle errors
    "W",      # pycodestyle warnings
    "F",      # Pyflakes
    "I",      # isort
    "B",      # flake8-bugbear
    "C4",     # flake8-comprehensions
    "UP",     # pyupgrade
    "RUF",    # Ruff-specific
    "N",      # pep8-naming
    "SIM",    # flake8-simplify
    "TID",    # flake8-tidy-imports
    "PERF",   # Perflint
    "PL",     # Pylint
]

# חוקים מושתקים (false positives נפוצים)
ignore = [
    "E501",   # line too long - ה-formatter מטפל בזה
    "E731",   # do not assign a lambda expression - לפעמים מוצדק
    "B008",   # do not perform function call in default arguments - pydantic נפגע
    "PLR0913", # too many arguments - לפעמים נחוץ
    "PLR2004", # magic value used in comparison - לפעמים מוצדק
    "PLC0415", # import-outside-toplevel - לפעמים נדרש
    "SIM108",  # ternary operator - לפעמים פחות קריא
]

# auto-fix מופעל לכל החוקים (שבטוחים לתיקון)
fixable = ["ALL"]
unfixable = [
    "F841",  # unused local variable - לפעמים שם משמש כdocumentation
]

# ==============================
# הגדרות per-file
# ==============================
[lint.per-file-ignores]
"**/__init__.py" = [
    "F401",   # imported but unused - __init__.py מייצא לרוב
    "E402",   # module level import not at top
]
"**/tests/**" = [
    "S101",   # assert - בtest זה בסדר
    "ANN",    # annotations - בtest לא תמיד נדרש
    "D",      # docstrings - בtest לא נדרש
    "PLR2004", # magic values - בtest בסדר
    "SLF001",  # private member access - בtest מקובל
]
"**/migrations/**" = ["ALL"]  # migrations - דלג על הכל
"**/conftest.py" = [
    "S101",
    "F401",
]
"tools/**" = [
    "T20",    # print statements - בtools מותר
    "INP001", # implicit namespace package
]

# ==============================
# isort configuration
# ==============================
[lint.isort]
known-first-party = [
    "lib_a",
    "lib_b",
    "service_x",
]
force-sort-within-sections = true
split-on-trailing-comma = true
combine-as-imports = true

# ==============================
# pydocstyle
# ==============================
[lint.pydocstyle]
convention = "google"   # google | numpy | pep257

# ==============================
# flake8-bugbear
# ==============================
[lint.flake8-bugbear]
extend-immutable-calls = [
    "fastapi.Depends",
    "fastapi.Query",
    "fastapi.Header",
    "fastapi.Cookie",
    "fastapi.Body",
    "fastapi.Form",
    "fastapi.File",
]

# ==============================
# McCabe complexity
# ==============================
[lint.mccabe]
max-complexity = 10

# ==============================
# Pylint
# ==============================
[lint.pylint]
max-args = 8
max-branches = 12
max-returns = 6

# ==============================
# הגדרות Formatter
# ==============================
[format]
quote-style = "double"         # כמו Black
indent-style = "space"         # spaces, לא tabs
line-ending = "lf"             # Unix line endings
docstring-code-format = true   # פורמוט code ב-docstrings
skip-magic-trailing-comma = false
```

### 5.4 הירארכיית קונפיגורציה במונורפו

RUFF תומך ב-nested configuration — כל package יכול לדרוס הגדרות מה-root:

```toml
# packages/legacy-lib/ruff.toml
# ירש מה-root אבל עם כמה דרסות ספציפיות
extend = "../../ruff.toml"    # ירושה מה-root

[lint]
# החוקים מה-root + הוספות
extend-select = ["ANN"]  # הוספת type annotations enforcement

# דרוס: בpackage זה יש legacy code
ignore = [
    "ANN001",  # missing type annotation for function argument
    "N803",    # argument name should be lowercase (legacy names)
]

[lint.per-file-ignores]
"src/legacy_lib/old_module.py" = [
    "E",   # ignore all pycodestyle in this file
    "W",
    "C4",
]
```

### 5.5 Migration מ-flake8/black/isort ל-RUFF

#### שלב 1: Inventory

```bash
# בדוק מה יש לך כרגע
grep -r "flake8\|black\|isort\|pylint\|autopep8" requirements*.txt pyproject.toml

# הסתכל על ה-.flake8 config שיש לך
cat .flake8
cat setup.cfg   # אם יש
```

#### שלב 2: המרת קונפיגורציית flake8

```ini
# .flake8 (ישן)
[flake8]
max-line-length = 88
extend-ignore = E203, E266, W503
exclude = .git, __pycache__, migrations
per-file-ignores =
    __init__.py: F401
    tests/*: S101
```

```toml
# ruff.toml (חדש - equivalent)
line-length = 88

[lint]
select = ["E", "W", "F"]
ignore = ["E203", "E266", "W503"]
exclude = [".git", "__pycache__", "migrations"]

[lint.per-file-ignores]
"**/__init__.py" = ["F401"]
"**/tests/**" = ["S101"]
```

#### שלב 3: הוספת noqa comments לקוד קיים

```bash
# הוסף noqa לכל הקבצים הקיימים שנכשלים
# (כדי לאפשר adoption הדרגתי)
uv run ruff check --add-noqa .

# אחר כך ניתן להסיר noqa comments אחד אחד
uv run ruff check --select UP035 --add-noqa .  # רק rule ספציפי
```

#### שלב 4: הסרת הכלים הישנים

```bash
# הסרה מ-UV workspace
uv remove flake8 black isort autopep8 pylint

# או אם הם ב-dependency-groups
# ערוך pyproject.toml ידנית והסר אותם
```

#### שלב 5: עדכון `pyproject.toml`

הסר את ה-sections הישנים:
```toml
# הסר אלה:
[tool.black]
[tool.isort]
[tool.flake8]   # (לרוב לא ב-pyproject.toml, ב-.flake8 נפרד)
```

### 5.6 פקודות RUFF בשימוש יומיומי

```bash
# ========================
# Linting
# ========================

# בדיקה בסיסית
uv run ruff check .
uv run ruff check packages/lib-a

# תיקון אוטומטי (safe fixes)
uv run ruff check --fix .

# תיקון כולל unsafe fixes
uv run ruff check --fix --unsafe-fixes .

# הצגת כל ה-violations כולל unfixable
uv run ruff check --show-fixes .

# בדיקה על חוק ספציפי
uv run ruff check --select F401 .

# debugging: מה ה-settings הפעילים?
uv run ruff check packages/lib-a/src/main.py --show-settings

# ========================
# Formatting
# ========================

# פורמוט
uv run ruff format .

# בדיקה בלבד (no write)
uv run ruff format --check .

# הצגת diff
uv run ruff format --diff .

# ========================
# CI mode output
# ========================

# פלט בפורמט GitLab
uv run ruff check --output-format=gitlab .

# פלט JSON
uv run ruff check --output-format=json .

# פלט GitHub
uv run ruff check --output-format=github .
```

### 5.7 RUFF בפורמט GitLab CI

```yaml
# .gitlab-ci.yml snippet
ruff-lint:
  stage: lint
  script:
    - uv run ruff check --output-format=gitlab .
  artifacts:
    reports:
      codequality: ruff-report.json
    when: always
  allow_failure: false
```

```bash
# יצירת code quality report לGitLab
uv run ruff check --output-format=json . > ruff-report.json
```

### 5.8 pre-commit hooks

```yaml
# .pre-commit-config.yaml
repos:
  # ==============================
  # RUFF
  # ==============================
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.9.0     # עדכן לגרסה האחרונה
    hooks:
      - id: ruff-check
        args: [--fix, --exit-non-zero-on-fix]
        types_or: [python, pyi, jupyter]
      - id: ruff-format
        types_or: [python, pyi, jupyter]

  # ==============================
  # UV lockfile sync
  # ==============================
  - repo: https://github.com/astral-sh/uv-pre-commit
    rev: 0.11.0
    hooks:
      - id: uv-lock
        # מוודא שה-lock file מעודכן

  # ==============================
  # General
  # ==============================
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-merge-conflict
      - id: check-added-large-files
        args: ['--maxkb=1000']
```

**ברשת סגורה — offline mode:**

```yaml
# pre-commit לא יכול להוריד repos ברשת סגורה
# פתרון: mirror local של ה-repos

repos:
  - repo: /opt/pre-commit-mirrors/ruff-pre-commit
    rev: v0.9.0
    hooks:
      - id: ruff-check
      - id: ruff-format
```

### 5.9 ביצועים — RUFF vs הכלים הישנים

```
250,000 שורות קוד Python:
┌─────────────────┬──────────────────┬─────────────────────┐
│ כלי             │ זמן ריצה         │ הערות               │
├─────────────────┼──────────────────┼─────────────────────┤
│ pylint          │ ~150 שניות       │                     │
│ flake8          │ ~45 שניות        │                     │
│ black           │ ~30 שניות        │ formatting בלבד     │
│ isort           │ ~15 שניות        │ imports בלבד        │
│ RUFF (lint)     │ ~0.4 שניות       │ 375x מהיר מpylint  │
│ RUFF (format)   │ ~0.5 שניות       │ 60x מהיר מblack    │
└─────────────────┴──────────────────┴─────────────────────┘
```

### 5.10 מלכודות נפוצות ו-false positives ב-RUFF

#### מלכודת 1: B008 — FastAPI/Pydantic default values

```python
# FastAPI pattern שגורם לB008 false positive
from fastapi import Depends, Query

def endpoint(
    q: str = Query(default=None),      # B008!
    db = Depends(get_db),              # B008!
):
    pass
```

**פתרון:**
```toml
[lint.flake8-bugbear]
extend-immutable-calls = [
    "fastapi.Depends",
    "fastapi.Query",
    "fastapi.Body",
    "fastapi.Header",
    "fastapi.Cookie",
]
```

#### מלכודת 2: D401 — First line should be in imperative mood

```python
def get_user():
    """Returns the current user."""  # D401: "Returns" אמור להיות "Return"
```

**פתרון:**
```toml
[lint]
ignore = ["D401"]  # אם הסגנון הזה מקובל אצלכם
```

#### מלכודת 3: E501 + Black/Ruff format conflict

RUFF formatter יטפל ב-line length אוטומטית, לכן אין צורך ב-E501:

```toml
[lint]
ignore = [
    "E501",  # handled by formatter
]
```

#### מלכודת 4: ANN101/ANN102 — missing type annotation for self/cls

```python
class MyClass:
    def method(self) -> None:  # ANN101: Missing type annotation for self
        pass
```

```toml
[lint]
ignore = [
    "ANN101",  # self annotation not needed
    "ANN102",  # cls annotation not needed
]
```

#### מלכודת 5: ISC001 — conflict עם formatter

```toml
[lint]
ignore = [
    "ISC001",   # conflicts with ruff format
    "COM812",   # conflicts with ruff format (trailing comma)
]
```

#### מלכודת 6: UP007 — Use X | Y for union type annotations

```python
# Python 3.9+ בלבד
from typing import Optional

def func(x: Optional[str]) -> None:  # UP007: use str | None
    pass
```

**פתרון:** וודא שה-`target-version` מוגדר נכון:
```toml
target-version = "py311"  # תאפשר UP007
```

---

## 6. שלב ד׳: פורמטר Python מותאם אישית

### 6.1 מה זה הפורמטר המותאם אישית?

סקריפט Python שמקבל כארגומנט שורת פקודה **תיקייה** ומחיל עליה את כל תהליך ה-formatting. הסקריפט הוא wrapper חכם סביב `ruff format` ו-`ruff check --fix`.

### 6.2 הסקריפט המלא: `tools/format.py`

```python
#!/usr/bin/env python3
"""
Custom Python Formatter Script
================================
Applies RUFF format and lint fixes to a given directory.

Usage:
    python tools/format.py <directory> [options]
    python tools/format.py packages/lib-a
    python tools/format.py packages/lib-a --check
    python tools/format.py packages/lib-a --unsafe-fixes
    python tools/format.py . --all

Exit codes:
    0 - Success, all formatting applied (or no changes needed in --check mode)
    1 - Formatting needed (in --check mode) or lint errors found
    2 - Configuration or runtime error
"""

from __future__ import annotations

import argparse
import subprocess
import sys
from pathlib import Path


def parse_args() -> argparse.Namespace:
    """Parse CLI arguments."""
    parser = argparse.ArgumentParser(
        description="Apply RUFF formatting and linting to a directory",
        formatter_class=argparse.RawDescriptionHelpFormatter,
        epilog=__doc__,
    )
    parser.add_argument(
        "directory",
        type=Path,
        help="Directory to format (relative to workspace root)",
    )
    parser.add_argument(
        "--check",
        action="store_true",
        default=False,
        help="Check only, do not modify files (exit 1 if changes needed)",
    )
    parser.add_argument(
        "--fix",
        action="store_true",
        default=True,
        help="Apply automatic fixes (default: True)",
    )
    parser.add_argument(
        "--unsafe-fixes",
        action="store_true",
        default=False,
        help="Apply potentially unsafe fixes",
    )
    parser.add_argument(
        "--no-lint",
        action="store_true",
        default=False,
        help="Skip linting, only format",
    )
    parser.add_argument(
        "--no-format",
        action="store_true",
        default=False,
        help="Skip formatting, only lint",
    )
    parser.add_argument(
        "--output-format",
        choices=["text", "json", "gitlab", "github", "junit"],
        default="text",
        help="Output format for lint results",
    )
    parser.add_argument(
        "--config",
        type=Path,
        default=None,
        help="Path to ruff.toml config file (default: auto-detect)",
    )
    parser.add_argument(
        "--verbose",
        "-v",
        action="store_true",
        default=False,
        help="Verbose output",
    )
    return parser.parse_args()


def find_workspace_root() -> Path:
    """Find the workspace root by looking for uv.lock or nx.json."""
    current = Path.cwd()
    for parent in [current, *current.parents]:
        if (parent / "uv.lock").exists() or (parent / "nx.json").exists():
            return parent
    return current


def run_command(
    cmd: list[str],
    cwd: Path,
    verbose: bool = False,
) -> tuple[int, str, str]:
    """Run a subprocess command and return (returncode, stdout, stderr)."""
    if verbose:
        print(f"Running: {' '.join(str(c) for c in cmd)}", file=sys.stderr)

    result = subprocess.run(
        cmd,
        cwd=cwd,
        capture_output=True,
        text=True,
        check=False,
    )
    return result.returncode, result.stdout, result.stderr


def validate_directory(directory: Path, workspace_root: Path) -> Path:
    """Validate that the directory exists and is within the workspace."""
    # Handle absolute and relative paths
    if directory.is_absolute():
        resolved = directory.resolve()
    else:
        resolved = (workspace_root / directory).resolve()

    if not resolved.exists():
        print(f"Error: Directory '{resolved}' does not exist", file=sys.stderr)
        sys.exit(2)

    if not resolved.is_dir():
        print(f"Error: '{resolved}' is not a directory", file=sys.stderr)
        sys.exit(2)

    # Security: ensure directory is within workspace
    try:
        resolved.relative_to(workspace_root.resolve())
    except ValueError:
        print(
            f"Error: Directory '{resolved}' is outside workspace root '{workspace_root}'",
            file=sys.stderr,
        )
        sys.exit(2)

    return resolved


def run_ruff_format(
    directory: Path,
    workspace_root: Path,
    check_only: bool = False,
    config: Path | None = None,
    verbose: bool = False,
) -> int:
    """Run ruff format on the directory."""
    cmd = ["uv", "run", "ruff", "format"]

    if check_only:
        cmd.append("--check")
        cmd.append("--diff")

    if config:
        cmd.extend(["--config", str(config)])

    # Pass relative path for cleaner output
    rel_dir = directory.relative_to(workspace_root)
    cmd.append(str(rel_dir))

    returncode, stdout, stderr = run_command(cmd, workspace_root, verbose)

    if stdout:
        print(stdout, end="")
    if stderr:
        print(stderr, end="", file=sys.stderr)

    if verbose:
        if returncode == 0:
            print(f"✓ Format: No changes needed in {rel_dir}", file=sys.stderr)
        else:
            if check_only:
                print(f"✗ Format: Changes needed in {rel_dir}", file=sys.stderr)
            else:
                print(f"✓ Format: Applied changes in {rel_dir}", file=sys.stderr)

    return returncode


def run_ruff_lint(
    directory: Path,
    workspace_root: Path,
    fix: bool = True,
    unsafe_fixes: bool = False,
    check_only: bool = False,
    output_format: str = "text",
    config: Path | None = None,
    verbose: bool = False,
) -> int:
    """Run ruff check (lint) on the directory."""
    cmd = ["uv", "run", "ruff", "check"]

    if not check_only and fix:
        cmd.append("--fix")

    if unsafe_fixes and not check_only:
        cmd.append("--unsafe-fixes")

    if output_format != "text":
        cmd.extend(["--output-format", output_format])

    if config:
        cmd.extend(["--config", str(config)])

    rel_dir = directory.relative_to(workspace_root)
    cmd.append(str(rel_dir))

    returncode, stdout, stderr = run_command(cmd, workspace_root, verbose)

    if stdout:
        print(stdout, end="")
    if stderr:
        print(stderr, end="", file=sys.stderr)

    return returncode


def main() -> int:
    """Main entrypoint."""
    args = parse_args()
    workspace_root = find_workspace_root()

    if args.verbose:
        print(f"Workspace root: {workspace_root}", file=sys.stderr)

    # Validate directory
    target_dir = validate_directory(args.directory, workspace_root)

    if args.verbose:
        print(f"Target directory: {target_dir}", file=sys.stderr)
        print(f"Mode: {'check' if args.check else 'fix'}", file=sys.stderr)

    exit_codes = []

    # Step 1: Run formatter
    if not args.no_format:
        if args.verbose:
            print("\n=== Running ruff format ===", file=sys.stderr)

        fmt_code = run_ruff_format(
            directory=target_dir,
            workspace_root=workspace_root,
            check_only=args.check,
            config=args.config,
            verbose=args.verbose,
        )
        exit_codes.append(fmt_code)

    # Step 2: Run linter
    if not args.no_lint:
        if args.verbose:
            print("\n=== Running ruff check ===", file=sys.stderr)

        lint_code = run_ruff_lint(
            directory=target_dir,
            workspace_root=workspace_root,
            fix=args.fix and not args.check,
            unsafe_fixes=args.unsafe_fixes,
            check_only=args.check,
            output_format=args.output_format,
            config=args.config,
            verbose=args.verbose,
        )
        exit_codes.append(lint_code)

    # Return 0 if all succeeded, 1 if any changes/errors
    final_code = max(exit_codes) if exit_codes else 0

    if args.verbose:
        status = "SUCCESS" if final_code == 0 else "ISSUES FOUND"
        print(f"\n=== {status} (exit code: {final_code}) ===", file=sys.stderr)

    return final_code


if __name__ == "__main__":
    sys.exit(main())
```

### 6.3 שימוש בסקריפט

```bash
# פורמוט directory ספציפי
python tools/format.py packages/lib-a

# check mode בלבד (לCI)
python tools/format.py packages/lib-a --check

# עם verbose output
python tools/format.py packages/lib-a --verbose

# פורמוט כל ה-workspace
python tools/format.py .

# עם unsafe fixes
python tools/format.py packages/lib-a --unsafe-fixes

# פלט GitLab
python tools/format.py packages/lib-a --check --output-format gitlab

# רק format, ללא lint
python tools/format.py packages/lib-a --no-lint

# רק lint, ללא format
python tools/format.py packages/lib-a --no-format

# דרך UV
uv run python tools/format.py packages/lib-a
```

### 6.4 אינטגרציה עם NX targets

ב-`project.json` של כל package:

```json
{
  "targets": {
    "format": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run python tools/format.py packages/lib-a",
        "cwd": "{workspaceRoot}"
      },
      "inputs": [
        "{projectRoot}/**/*.py",
        "{workspaceRoot}/ruff.toml",
        "{workspaceRoot}/tools/format.py"
      ],
      "cache": true
    },
    "format-check": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run python tools/format.py packages/lib-a --check",
        "cwd": "{workspaceRoot}"
      },
      "inputs": [
        "{projectRoot}/**/*.py",
        "{workspaceRoot}/ruff.toml",
        "{workspaceRoot}/tools/format.py"
      ],
      "cache": true
    }
  }
}
```

**הרצה דרך NX:**
```bash
# פורמוט package ספציפי
nx run lib-a:format

# בדיקת format ל-affected packages
nx affected -t format-check

# פורמוט כל המונורפו
nx run-many -t format
```

### 6.5 הסקריפט בתור NX target גלובלי

ב-`nx.json`, הוסף format target גלובלי:

```json
{
  "targetDefaults": {
    "format": {
      "executor": "nx:run-commands",
      "cache": true,
      "inputs": [
        "{projectRoot}/**/*.py",
        "{workspaceRoot}/ruff.toml",
        "{workspaceRoot}/tools/format.py"
      ]
    }
  }
}
```

ניתן גם להגדיר target ב-root `package.json`:

```json
{
  "scripts": {
    "format": "uv run python tools/format.py .",
    "format:check": "uv run python tools/format.py . --check",
    "format:lib-a": "uv run python tools/format.py packages/lib-a"
  },
  "nx": {
    "includedScripts": ["format", "format:check"]
  }
}
```

---

## 7. אינטגרציה מלאה: NX + UV + RUFF ב-GitLab CI/CD

### 7.1 `.gitlab-ci.yml` מלא

```yaml
# .gitlab-ci.yml
# ==============================
# Global Configuration
# ==============================
default:
  image: python:3.11-slim

variables:
  # UV Cache
  UV_CACHE_DIR: ".uv-cache"
  UV_LINK_MODE: "copy"          # נדרש ב-GitLab CI
  UV_FROZEN: "true"             # אל תשנה את uv.lock ב-CI
  UV_NO_PROGRESS: "true"        # נקי יותר ב-CI logs

  # Python
  PYTHONDONTWRITEBYTECODE: "1"
  PYTHONUNBUFFERED: "1"

  # NX
  NX_NON_NATIVE_HASHER: "true"
  CI: "true"

  # GitLab artifacts
  GIT_DEPTH: "0"               # Full clone for nx affected

stages:
  - setup
  - validate
  - test
  - build
  - release

# ==============================
# Cache templates
# ==============================
.uv-cache: &uv-cache
  cache:
    key:
      files:
        - uv.lock
    paths:
      - .uv-cache
    policy: pull-push

.nx-cache: &nx-cache
  cache:
    key:
      files:
        - nx.json
        - "**/*.py"
        - uv.lock
    paths:
      - .nx
    policy: pull-push

# ==============================
# Templates
# ==============================
.base-python:
  <<: *uv-cache
  before_script:
    # התקנת Node.js עבור NX
    - apt-get update -qq && apt-get install -y -qq nodejs npm curl
    # התקנת UV
    - pip install uv --quiet
    # התקנת NX
    - npm install -g nx --silent
    # sync dependencies
    - uv sync --frozen --no-dev
    # חישוב NX base/head
    - export NX_HEAD=$CI_COMMIT_SHA
    - export NX_BASE=${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}
  after_script:
    # ניקוי UV cache כדי לא לצבור גדלים
    - uv cache prune --ci

# ==============================
# Setup Stage
# ==============================
setup-dependencies:
  stage: setup
  extends: .base-python
  script:
    - uv sync --frozen
    - uv run python -c "import sys; print(f'Python {sys.version}')"
    - uv run ruff --version
    - nx --version
  artifacts:
    paths:
      - .venv
    expire_in: 1 hour

# ==============================
# Validate Stage
# ==============================
lint:
  stage: validate
  extends: .base-python
  needs: ["setup-dependencies"]
  script:
    # הרצת ruff lint על affected packages
    - >
      nx affected -t lint
      --base=$NX_BASE
      --head=$NX_HEAD
      --output-style=stream
      --parallel=4
  rules:
    - if: $CI_MERGE_REQUEST_ID
    - if: $CI_COMMIT_BRANCH == "main"

format-check:
  stage: validate
  extends: .base-python
  needs: ["setup-dependencies"]
  script:
    # בדיקת format - לא מבצע שינויים
    - nx affected -t format-check --base=$NX_BASE --head=$NX_HEAD
  rules:
    - if: $CI_MERGE_REQUEST_ID
    - if: $CI_COMMIT_BRANCH == "main"

ruff-report:
  stage: validate
  extends: .base-python
  needs: ["setup-dependencies"]
  script:
    # יצירת GitLab Code Quality report
    - uv run ruff check --output-format=json . > ruff-quality-report.json || true
  artifacts:
    reports:
      codequality: ruff-quality-report.json
    when: always
    expire_in: 1 week
  rules:
    - if: $CI_MERGE_REQUEST_ID

# ==============================
# Test Stage
# ==============================
test:
  stage: test
  extends: .base-python
  needs: ["setup-dependencies", "lint"]
  script:
    - uv sync --frozen  # כולל dev dependencies
    - nx affected -t test --base=$NX_BASE --head=$NX_HEAD --parallel=4
  coverage: '/TOTAL.*\s+(\d+%)$/'
  artifacts:
    reports:
      junit: "**/junit.xml"
      coverage_report:
        coverage_format: cobertura
        path: "**/coverage.xml"
    paths:
      - "**/coverage/"
    when: always
    expire_in: 1 week

# ==============================
# Build Stage
# ==============================
build:
  stage: build
  extends: .base-python
  needs: ["test"]
  script:
    - nx affected -t build --base=$NX_BASE --head=$NX_HEAD
  artifacts:
    paths:
      - "**/dist/"
    expire_in: 1 week
  rules:
    - if: $CI_COMMIT_BRANCH == "main"

# ==============================
# Full pipeline (main branch push)
# ==============================
full-validation:
  stage: validate
  extends: .base-python
  needs: ["setup-dependencies"]
  script:
    - nx run-many -t lint format-check --parallel=4
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: never  # לא מריצים פעמיים על main
    - if: $CI_PIPELINE_SOURCE == "schedule"

# ==============================
# Release Stage
# ==============================
release:
  stage: release
  extends: .base-python
  needs: ["build"]
  script:
    # חישוב version בלבד תחילה (dry-run)
    - nx release version --dry-run
    # לאחר אישור:
    - nx release --skip-publish  # version + changelog
    # publish ל-internal registry
    - uv publish --index internal-registry
  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual  # Manual trigger בלבד
  environment:
    name: production
```

### 7.2 `.gitlab-ci.yml` לרשת סגורה (air-gapped)

```yaml
# .gitlab-ci.yml - Air-Gapped Version
variables:
  # Internal Docker registry
  DOCKER_REGISTRY: "registry.internal.company.com"
  PYTHON_IMAGE: "${DOCKER_REGISTRY}/python:3.11-slim"

  # Internal PyPI
  UV_DEFAULT_INDEX: "https://pypi.internal.company.com/simple/"
  UV_INDEX_INTERNAL_USERNAME: "${INTERNAL_PYPI_USER}"
  UV_INDEX_INTERNAL_PASSWORD: "${INTERNAL_PYPI_TOKEN}"

  # Disable external downloads
  UV_PYTHON_DOWNLOADS: "never"
  UV_NO_GITHUB_FAST_PATH: "true"

default:
  image: ${PYTHON_IMAGE}

before_script:
  # UV כבר מותקן ב-Docker image הפנימי
  - uv --version
  # NX כבר מותקן
  - nx --version
  # sync מ-internal registry בלבד
  - uv sync --frozen --no-dev
```

### 7.3 Dockerfile לselfhosted CI runner

```dockerfile
# CI Runner Docker Image
FROM python:3.11-slim

# Install Node.js
RUN apt-get update && apt-get install -y \
    nodejs npm curl git \
    && rm -rf /var/lib/apt/lists/*

# Install UV
RUN pip install uv==0.5.0  # גרסה ספציפית לstability

# Install NX globally
RUN npm install -g nx@21.0.0

# Configure UV
ENV UV_CACHE_DIR=/tmp/uv-cache
ENV UV_PYTHON_DOWNLOADS=never
ENV UV_LINK_MODE=copy

# Workspace setup
WORKDIR /workspace
```

---

## 8. קונפיגורציה של מונורפו: הירארכיה ושיתוף הגדרות

### 8.1 ארכיטקטורת הקונפיגורציה

```
monorepo/
├── ruff.toml              ← הגדרות RUFF גלובליות
├── pyproject.toml         ← UV workspace + שיתוף RUFF config
├── nx.json                ← NX globals
│
├── packages/
│   ├── lib-a/
│   │   ├── ruff.toml      ← extend + overrides ספציפיים (אופציונלי)
│   │   └── pyproject.toml ← UV package config
│   │
│   └── legacy-lib/
│       ├── ruff.toml      ← ירושה עם relaxed rules לlegacy code
│       └── pyproject.toml
```

### 8.2 RUFF inheritance במונורפו

```toml
# packages/legacy-lib/ruff.toml
# ירושה מה-root + overrides
extend = "../../ruff.toml"

[lint]
# הוסף חוקים ספציפיים
extend-select = [
    "ANN",   # annotation enforcement רק לlib חדש
]

# הסר חוקים בעייתיים לlegacy
extend-ignore = [
    "N803",   # argument name should be lowercase (legacy names)
    "N806",   # variable in function should be lowercase
    "UP",     # pyupgrade - legacy code doesn't need to be modern
]

[lint.per-file-ignores]
"src/legacy_lib/old_api.py" = ["ALL"]  # קובץ ישן מאוד - דלג על הכל
"src/legacy_lib/compat/*.py" = [
    "E",
    "W",
    "F401",
]
```

### 8.3 UV configuration inheritance

```toml
# packages/service-x/pyproject.toml
[project]
name = "service-x"
version = "2.1.0"
requires-python = ">=3.11"
dependencies = [
    "lib-a",        # internal dependency
    "lib-b",        # internal dependency
    "fastapi>=0.110",
    "uvicorn>=0.29",
]

[dependency-groups]
dev = [
    "pytest>=8.0",
    "httpx>=0.27",   # for test client
]
test = [
    { include-group = "dev" },
    "pytest-asyncio>=0.23",
    "factory-boy>=3.3",
]
lint = [
    "ruff>=0.9.0",
    "mypy>=1.10",
]

# Internal workspace dependencies
[tool.uv.sources]
lib-a = { workspace = true }
lib-b = { workspace = true }

# Per-package index override (אם ה-package צריך index שונה)
# [[tool.uv.index]]
# name = "special-index"
# url = "https://special.internal.company.com/simple/"
# explicit = true
```

### 8.4 NX tags לarchitecture enforcement

```json
// packages/shared-models/project.json
{
  "tags": ["scope:shared", "type:library", "layer:data"],
  "targets": { ... }
}

// packages/business-logic/project.json
{
  "tags": ["scope:backend", "type:library", "layer:service"],
  "targets": { ... }
}

// apps/api-server/project.json
{
  "tags": ["scope:backend", "type:app", "layer:api"],
  "implicitDependencies": ["shared-models", "business-logic"],
  "targets": { ... }
}
```

---

## 9. ניהול Release חכם עם NX

### 9.1 אסטרטגיית Versioning

NX תומך בשני מצבי release:

**1. Independent versioning** (מומלץ למונורפו גדול):
- כל package מקבל גרסה עצמאית
- `lib-a@1.5.0`, `lib-b@3.2.1`, `service-x@0.8.0`
- שינוי ב-`lib-a` לא מעלה גרסה ל-`lib-b` אוטומטית

**2. Fixed/Locked versioning:**
- כל הpackages מקבלים גרסה זהה בכל פעם
- קל יותר לנהל אבל פחות גמיש

### 9.2 קונפיגורציית `nx.json` לrelease

```json
{
  "release": {
    "projects": ["packages/*", "apps/*"],
    "projectsRelationship": "independent",
    "releaseTagPattern": "{projectName}@{version}",
    "changelog": {
      "automaticFromRef": true,
      "projectChangelogs": {
        "createRelease": false,
        "file": "{projectRoot}/CHANGELOG.md",
        "renderOptions": {
          "authors": true,
          "commitReferences": true,
          "versionTitleDate": true
        }
      },
      "workspaceChangelog": {
        "createRelease": false,
        "file": "CHANGELOG.md"
      }
    },
    "version": {
      "conventionalCommits": true,
      "generatorOptions": {
        "currentVersionResolver": "git-tag",
        "specifierSource": "conventional-commits"
      }
    }
  }
}
```

### 9.3 Conventional Commits

NX משתמש ב-Conventional Commits לחישוב אוטומטי של גרסאות:

```
feat: add new authentication method       → minor bump (1.0.0 → 1.1.0)
fix: resolve null pointer in parser       → patch bump (1.0.0 → 1.0.1)
feat!: breaking change in API             → major bump (1.0.0 → 2.0.0)
chore: update dependencies                → no bump
docs: update README                       → no bump
```

**הגדרת commit hooks:**
```yaml
# .pre-commit-config.yaml
- repo: https://github.com/compilerla/conventional-pre-commit
  rev: v3.3.0
  hooks:
    - id: conventional-pre-commit
      stages: [commit-msg]
      args: [feat, fix, chore, docs, refactor, test, ci, perf]
```

### 9.4 תהליך Release

```bash
# ==============================
# Dry-run - לראות מה יקרה
# ==============================
nx release --dry-run

# ==============================
# Release מלא
# ==============================

# שלב 1: חשב versions
nx release version

# שלב 2: צור changelog
nx release changelog

# שלב 3: commit ו-tag
nx release --git-commit --git-push --git-tag

# שלב 4: build
nx run-many -t build --projects $(nx release --print-affected-projects)

# שלב 5: publish
uv publish --index internal-registry

# ==============================
# Release לpackage ספציפי בלבד
# ==============================
nx release --projects lib-a --dry-run
nx release --projects lib-a --git-commit --git-push --git-tag

# ==============================
# First release
# ==============================
nx release --first-release --dry-run
nx release --first-release
```

### 9.5 Release ב-GitLab CI

```yaml
# .gitlab-ci.yml - release job
release:
  stage: release
  extends: .base-python
  script:
    # וודא שאנחנו על main branch
    - |
      if [ "$CI_COMMIT_BRANCH" != "main" ]; then
        echo "Release only allowed from main branch"
        exit 1
      fi

    # חישוב versions
    - nx release version --dry-run
    - nx release version

    # יצירת changelog
    - nx release changelog --git-commit --git-push --git-tag

    # build packages
    - nx run-many -t build

    # publish ל-internal registry
    - uv publish --index internal-registry --token $INTERNAL_PYPI_TOKEN

    # צור GitLab release note
    - |
      TAG=$(git describe --tags --abbrev=0)
      curl --request POST \
        --header "PRIVATE-TOKEN: $CI_API_TOKEN" \
        --data-urlencode "name=Release $TAG" \
        --data-urlencode "tag_name=$TAG" \
        --data-urlencode "description=$(cat CHANGELOG.md | head -50)" \
        "$CI_API_V4_URL/projects/$CI_PROJECT_ID/releases"

  rules:
    - if: $CI_COMMIT_BRANCH == "main"
      when: manual
  environment:
    name: production
```

---

## 10. עבודה ברשת סגורה (Air-Gapped)

### 10.1 האתגרים

ברשת סגורה:
1. אין גישה ל-PyPI
2. אין גישה ל-GitHub (pre-commit repos, UV binaries)
3. NX Cloud לא זמין
4. Docker images צריכים להיות mirrored

### 10.2 הגדרת Internal PyPI Mirror

**אפשרות 1: Artifactory / Nexus**
```toml
# pyproject.toml
[[tool.uv.index]]
name = "artifactory"
url = "https://artifactory.internal.company.com/artifactory/api/pypi/pypi-virtual/simple/"
default = true
authenticate = "always"
```

**אפשרות 2: Devpi (open source)**
```toml
[[tool.uv.index]]
name = "devpi"
url = "https://devpi.internal.company.com/root/pypi/+simple/"
default = true
```

**אפשרות 3: bandersnatch (PyPI mirror)**
```bash
# הרצת mirror מקומי
pip install bandersnatch
bandersnatch mirror --config /etc/bandersnatch.conf
```

### 10.3 UV ב-Air-Gapped Environment

```bash
# Environment variables לרשת סגורה
export UV_DEFAULT_INDEX="https://pypi.internal.company.com/simple/"
export UV_INDEX_INTERNAL_USERNAME="user"
export UV_INDEX_INTERNAL_PASSWORD="token"
export UV_PYTHON_DOWNLOADS="never"   # אל תנסה להוריד Python
export UV_NO_GITHUB_FAST_PATH="true"
export UV_HTTP_TIMEOUT="60"          # timeout ארוך יותר לרשת פנימית

# sync ללא גישה לאינטרנט
uv sync --frozen --no-python-downloads
```

### 10.4 NX ברשת סגורה

```bash
# התקנת NX מ-internal npm registry
npm install -g nx --registry https://npm.internal.company.com/

# או
npm config set registry https://npm.internal.company.com/
npm install -g nx
```

**נסיון caching ללא NX Cloud:**

```json
// nx.json - local cache בלבד
{
  "tasksRunnerOptions": {
    "default": {
      "runner": "nx/tasks-runners/default",
      "options": {
        "cacheableOperations": ["build", "test", "lint", "format-check"],
        "cacheDirectory": "/shared/nx-cache"  // shared filesystem cache
      }
    }
  }
}
```

**שיתוף cache דרך artifact storage:**

```yaml
# .gitlab-ci.yml
cache:
  key: nx-cache-${CI_COMMIT_REF_NAME}
  paths:
    - .nx/cache
  policy: pull-push
```

### 10.5 pre-commit ברשת סגורה

```yaml
# .pre-commit-config.yaml - using local mirrors
repos:
  # Mirror את ה-repos לGitLab פנימי
  - repo: https://gitlab.internal.company.com/mirrors/ruff-pre-commit
    rev: v0.9.0
    hooks:
      - id: ruff-check
        additional_dependencies: []  # אל תתקין כלום נוסף
      - id: ruff-format
```

**חלופה — local hooks:**
```yaml
repos:
  - repo: local
    hooks:
      - id: ruff-check
        name: Ruff Linter
        entry: uv run ruff check --fix
        language: system
        types_or: [python, pyi]
        require_serial: false

      - id: ruff-format
        name: Ruff Formatter
        entry: uv run ruff format
        language: system
        types_or: [python, pyi]
        require_serial: false
```

---

## 11. בעיות ידועות ומלכודות נפוצות

### 11.1 בעיות NX

#### בעיה 1: NX לא מזהה Python projects

**תסמין:** `nx graph` לא מציג את הPython packages

**פתרון:** ודא שלכל package יש `project.json` תקין:
```bash
# בדיקה
ls packages/*/project.json

# יצירת project.json בסיסי
cat > packages/lib-a/project.json << 'EOF'
{
  "name": "lib-a",
  "projectType": "library",
  "root": "packages/lib-a",
  "targets": {}
}
EOF
```

#### בעיה 2: NX affected לא עובד כראוי ב-CI

**תסמין:** `nx affected` מריץ על כל הפרויקטים

**סיבה:** `GIT_DEPTH` קצר מדי ב-GitLab

**פתרון:**
```yaml
# .gitlab-ci.yml
variables:
  GIT_DEPTH: "0"  # Full clone
  # או
  GIT_DEPTH: "50"  # לפחות 50 commits
```

#### בעיה 3: NX cache miss תמידי

**תסמין:** NX תמיד מריץ מחדש, אף פעם לא מגיע מcache

**סיבה:** ה-inputs מוגדרים רחב מדי

**פתרון:** הגדר inputs ספציפיים:
```json
{
  "targets": {
    "lint": {
      "inputs": [
        "{projectRoot}/src/**/*.py",
        "{projectRoot}/tests/**/*.py",
        "{workspaceRoot}/ruff.toml"
      ]
    }
  }
}
```

#### בעיה 4: Circular dependencies בין packages

**תסמין:** `nx graph` מציג cycles

**פתרון:**
```bash
# בדוק cycles
nx graph --focus lib-a

# NX ידגים את ה-cycle
# תקן ידנית על ידי refactoring
```

### 11.2 בעיות UV

#### בעיה 1: Resolution conflicts בין workspace members

**תסמין:**
```
error: Because lib-a requires httpx>=0.27 and service-x requires httpx<0.27,
we can conclude that lib-a and service-x cannot be used together.
```

**פתרון:**
```bash
# בדוק מי דורש מה
uv tree --package lib-a
uv tree --package service-x

# עדכן constraints
# בlib-a:
# httpx>=0.25  (רחב יותר)
# בservice-x:
# httpx>=0.25  (אל תגביל מלמעלה אלא אם יש סיבה)
```

#### בעיה 2: uv.lock conflicts ב-git

**תסמין:** conflicts ב-`uv.lock` אחרי merge

**פתרון:**
```bash
# אחרי resolving merge conflicts ב-pyproject.toml
git checkout uv.lock  # החזר ל-HEAD
uv lock               # צור lockfile חדש
git add uv.lock
git commit -m "chore: regenerate uv.lock after merge"
```

#### בעיה 3: UV לא מוצא Python

```bash
# בדוק גרסאות זמינות
uv python list

# התקן גרסה
uv python install 3.11

# pin לpyproject.toml
uv python pin 3.11
```

#### בעיה 4: editable installs לא עובד בCI

**תסמין:** workspace members לא נמצאים ב-CI

**פתרון:**
```bash
# sync עם editable mode מפורש
uv sync --frozen

# ודא שהpackages מוגדרים נכון כworkspace members
# בroot pyproject.toml:
# [tool.uv.workspace]
# members = ["packages/*"]
```

### 11.3 בעיות RUFF

#### בעיה 1: RUFF מפרמט שונה מBlack

**תסמין:** אחרי migration, PR מציג שינויים שהם רק styling

**סיבה:** RUFF formatter שונה קצת מBlack בf-strings ובsome edge cases

**פתרון:**
```bash
# הרץ ruff format על כל הcodebase כ-one-time cleanup
uv run ruff format .
git add -A
git commit -m "chore: apply ruff format (replaces black)"
```

#### בעיה 2: isort order שונה

```toml
# ruff.toml - קונפיגורציה לתאימות מקסימלית עם isort black profile
[lint.isort]
force-sort-within-sections = false
lines-after-imports = 2
force-single-line = false
```

#### בעיה 3: noqa comments לא עובדים

```python
# זה לא עובד:
import os  # noqa
# כי noqa בלי code מושתק הכל, אבל RUFF עשוי להתריע

# זה עובד:
import os  # noqa: F401
```

#### בעיה 4: RUFF בוצע על notebooks בCI

```toml
# ruff.toml - דלג על notebooks
[lint]
exclude = ["**/*.ipynb"]

[format]
exclude = ["**/*.ipynb"]
```

#### בעיה 5: D rules (pydocstyle) קפדניות מדי

```toml
[lint]
# התחל עם D בלי D213/D203 (conflicting rules)
select = ["D"]

[lint.pydocstyle]
convention = "google"  # זה אוטומטית מבטל חוקים מתנגשים
```

### 11.4 בעיות GitLab CI

#### בעיה 1: UV link mode error

```
error: Failed to hardlink files
```

**פתרון:**
```yaml
variables:
  UV_LINK_MODE: "copy"  # GitLab CI לא תומך בhardlinks
```

#### בעיה 2: NX affected מריץ על כל הפרויקטים ב-push ל-main

**פתרון:**
```yaml
script:
  - export NX_HEAD=$CI_COMMIT_SHA
  - export NX_BASE=$CI_COMMIT_BEFORE_SHA  # ה-commit הקודם ב-main
  - |
    # אם זה ה-commit הראשון ב-branch
    if [ "$NX_BASE" = "0000000000000000000000000000000000000000" ]; then
      export NX_BASE=$(git rev-list --max-parents=0 HEAD)
    fi
  - nx affected -t lint test --base=$NX_BASE --head=$NX_HEAD
```

---

## 12. checklist מלא לכל שלב

### 12.1 NX Setup Checklist

```
□ Node.js 18+ מותקן על כל מכונות ה-development וCI
□ nx מותקן גלובלית (npm install -g nx)
□ nx.json נוצר ב-root המונורפו
□ project.json נוצר לכל package
□ targets מוגדרים: lint, format, test, build
□ namedInputs מוגדרים ב-nx.json
□ targetDefaults מוגדרים ב-nx.json
□ tags מוגדרים לכל project
□ nx graph עובד ומציג את הgraph הנכון
□ nx affected -t test עובד בlocal
□ CI/CD מוגדר עם NX_BASE ו-NX_HEAD נכון
□ GIT_DEPTH=0 מוגדר ב-GitLab CI
□ cache מוגדר ב-CI (.nx/cache)
□ nx release מוגדר ב-nx.json
□ conventional commits מיושמים בצוות
```

### 12.2 UV Migration Checklist

```
□ UV מותקן על כל מכונות ה-development
□ UV מותקן ב-CI (docker image או pip install)
□ root pyproject.toml מוגדר כworkspace root
□ כל package מוגדר כ-workspace member
□ pyproject.toml של כל package ב-PEP 621 format
□ [tool.poetry] section הוסר מכל pyproject.toml
□ build-system מוגדר (hatchling/setuptools/flit)
□ uv.lock נוצר (uv lock)
□ uv sync עובד בlocal
□ uv run pytest עובד לכל package
□ internal registry מוגדר ב-pyproject.toml
□ CI מגדיר UV_ environment variables
□ CI משתמש ב-UV_FROZEN=true
□ uv-lock pre-commit hook מוגדר
□ poetry.lock נמחק מה-repo (לאחר מיגרציה מוצלחת)
□ .python-version קבצים קיימים לכל package
```

### 12.3 RUFF Setup Checklist

```
□ ruff.toml נוצר ב-root המונורפו
□ target-version מוגדר נכון
□ rule sets מוגדרים (select, ignore)
□ per-file-ignores מוגדרים
□ isort configuration מוגדרת (known-first-party)
□ format configuration מוגדרת
□ flake8/black/isort/pylint הוסרו מdependencies
□ [tool.black] ו-[tool.isort] הוסרו מpyproject.toml
□ .flake8 / setup.cfg sections הוסרו
□ noqa comments עודכנו לformat החדש
□ pre-commit hooks מוגדרים (ruff-check, ruff-format)
□ NX targets מוגדרים: lint, lint-fix, format, format-check
□ CI job מוגדר עם GitLab output format
□ Code Quality report מוגדר ב-CI artifacts
□ uv run ruff check . עובד clean
□ uv run ruff format --check . עובד clean
```

### 12.4 Custom Formatter Checklist

```
□ tools/format.py קיים ב-root
□ סקריפט עם +x permissions (על Linux)
□ NX targets מוגדרים: format, format-check
□ pre-commit hook local מוגדר (אופציונלי)
□ שימוש בסקריפט מתועד ב-README
□ CI job מריץ format-check
□ סקריפט תומך ב-exit codes נכונים (0/1/2)
□ סקריפט תומך ב--output-format gitlab לCI
```

---

## נספח: מבנה קבצים מלא לדוגמה

### `nx.json` מלא

```json
{
  "$schema": "./node_modules/nx/schemas/nx-schema.json",
  "defaultBase": "main",
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/**/*.spec.py",
      "!{projectRoot}/tests/**/*",
      "!{projectRoot}/**/*.md",
      "!{projectRoot}/CHANGELOG.md"
    ],
    "sharedGlobals": [
      "{workspaceRoot}/pyproject.toml",
      "{workspaceRoot}/uv.lock",
      "{workspaceRoot}/ruff.toml"
    ]
  },
  "targetDefaults": {
    "lint": {
      "cache": true,
      "inputs": ["default", "^production", "sharedGlobals"]
    },
    "format-check": {
      "cache": true,
      "inputs": ["default", "sharedGlobals"]
    },
    "test": {
      "cache": true,
      "inputs": ["default", "^production"],
      "outputs": ["{projectRoot}/coverage"],
      "dependsOn": ["^build"]
    },
    "build": {
      "cache": true,
      "inputs": ["production", "^production"],
      "outputs": ["{projectRoot}/dist"],
      "dependsOn": ["^build"]
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
      "conventionalCommits": true
    }
  }
}
```

### `ruff.toml` מלא

```toml
line-length = 88
indent-width = 4
target-version = "py311"
respect-gitignore = true

exclude = [
    ".git", ".venv", "__pycache__",
    "*.egg-info", "dist", "build",
    "migrations", "**/generated/**",
]

[lint]
select = [
    "E", "W", "F", "I",
    "B", "C4", "UP", "RUF",
    "N", "SIM", "PERF", "PL",
]
ignore = [
    "E501", "E731",
    "B008",
    "PLR0913", "PLR2004",
    "ISC001", "COM812",
]
fixable = ["ALL"]

[lint.per-file-ignores]
"**/__init__.py" = ["F401", "E402"]
"**/tests/**" = ["S101", "ANN", "D", "PLR2004", "SLF001"]
"**/migrations/**" = ["ALL"]
"**/conftest.py" = ["S101", "F401"]
"tools/**" = ["T20", "INP001"]

[lint.isort]
known-first-party = ["lib_a", "lib_b", "service_x"]
force-sort-within-sections = true
split-on-trailing-comma = true

[lint.pydocstyle]
convention = "google"

[lint.flake8-bugbear]
extend-immutable-calls = [
    "fastapi.Depends", "fastapi.Query",
    "fastapi.Header", "fastapi.Cookie",
    "fastapi.Body", "fastapi.Form",
]

[lint.mccabe]
max-complexity = 10

[lint.pylint]
max-args = 8
max-branches = 12

[format]
quote-style = "double"
indent-style = "space"
line-ending = "lf"
docstring-code-format = true
```

### `.pre-commit-config.yaml` מלא

```yaml
default_language_version:
  python: python3.11

repos:
  # RUFF
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.9.0
    hooks:
      - id: ruff-check
        args: [--fix, --exit-non-zero-on-fix]
        types_or: [python, pyi]
      - id: ruff-format
        types_or: [python, pyi]

  # UV lockfile
  - repo: https://github.com/astral-sh/uv-pre-commit
    rev: 0.11.0
    hooks:
      - id: uv-lock

  # General
  - repo: https://github.com/pre-commit/pre-commit-hooks
    rev: v4.6.0
    hooks:
      - id: trailing-whitespace
      - id: end-of-file-fixer
      - id: check-yaml
      - id: check-toml
      - id: check-merge-conflict
      - id: check-added-large-files
        args: ['--maxkb=1000']
      - id: debug-statements

  # Conventional commits
  - repo: https://github.com/compilerla/conventional-pre-commit
    rev: v3.3.0
    hooks:
      - id: conventional-pre-commit
        stages: [commit-msg]
```

---

## סיכום והמלצות

### סדר עדיפויות מוצע

**שבוע 1:** הטמעת NX ביסודיות — הגדר את כל ה-project.json, ודא ש-nx affected עובד ב-CI, הגדר caching.

**שבוע 2:** migration של poetry → UV — התחל מספריה אחת, ודא שהכל עובד, הרחב לשאר.

**שבוע 3:** הטמעת RUFF — החלפת flake8/black/isort, הגדרת rules, הרצת `--add-noqa` על קוד קיים.

**שבוע 4:** Custom formatter + polish — הסקריפט, ה-pre-commit hooks, ה-CI jobs המלאים.

### הצלחות מדידות

| מדד | לפני | אחרי |
|-----|------|------|
| זמן CI לפר PR | ~15 דקות | ~3 דקות (affected בלבד) |
| זמן lint | ~45 שניות | ~1 שנייה |
| זמן `pip install` | ~120 שניות | ~8 שניות (UV) |
| PR על ספריה ספציפית | בונה הכל | בונה רק affected |
| release process | ידני / שעות | אוטומטי / דקות |

### טיפ אחרון

**אל תנסו לעשות הכל בבת אחת.** המעבר ל-UV, NX, RUFF בו זמנית הוא רסק. כל כלי הוא migration process נפרד שדורש testing, דיבוג, ולמידה. הגישה השלבית המוצגת כאן — שבועיים לכל כלי — היא לא רק המלצה, היא הכרח.

הצלחה!

---

*מסמך זה נוצר ב-2026-03-24. הגרסאות המוזכרות עשויות להתיישן — בדקו תמיד את התיעוד הרשמי של [UV](https://docs.astral.sh/uv), [RUFF](https://docs.astral.sh/ruff), ו-[NX](https://nx.dev) לגרסאות העדכניות ביותר.*
