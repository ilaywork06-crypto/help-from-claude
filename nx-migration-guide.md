# מדריך מקצועי: מעבר ל-NX, UV, Ruff ופורמטר מותאם בסביבת GitLab

> **קהל יעד:** צוות DevOps, מפתחים בכירים ומנהלי תשתית
> **סביבה:** מונורפו קיים, GitLab CI/CD, Python + Vue.js, Poetry כמנהל חבילות נוכחי
> **גרסה:** מדריך זה מבוסס על NX 22+, UV 0.6+, Ruff 0.9+

---

## תוכן עניינים

1. [רקע ומוטיבציה](#רקע-ומוטיבציה)
2. [NX - מה זה ואיך זה עובד](#nx---מה-זה-ואיך-זה-עובד)
3. [מושגי ליבה ב-NX](#מושגי-ליבה-ב-nx)
4. [הוספת NX למונורפו קיים](#הוספת-nx-למונורפו-קיים)
5. [NX לפרויקטי Python](#nx-לפרויקטי-python)
6. [NX Affected - הכלי החכם לזיהוי שינויים](#nx-affected---הכלי-החכם-לזיהוי-שינויים)
7. [NX Release - ניהול גרסאות ופרסום סלקטיבי](#nx-release---ניהול-גרסאות-ופרסום-סלקטיבי)
8. [NX Cache - מטמון מקומי ומרוחק](#nx-cache---מטמון-מקומי-ומרוחק)
9. [NX עם GitLab CI/CD](#nx-עם-gitlab-cicd)
10. [NX עם Vue.js](#nx-עם-vuejs)
11. [מבנה project.json](#מבנה-projectjson)
12. [מבנה nx.json](#מבנה-nxjson)
13. [Tags ו-Implicit Dependencies](#tags-ו-implicit-dependencies)
14. [גרף תלויות - Dependency Graph](#גרף-תלויות---dependency-graph)
15. [מלכודות נפוצות ואיך להימנע מהן](#מלכודות-נפוצות-ואיך-להימנע-מהן)
16. [UV - מעבר מ-Poetry](#uv---מעבר-מ-poetry)
17. [Ruff - לינטר מהיר לפייתון](#ruff---לינטר-מהיר-לפייתון)
18. [פורמטר מותאם אישית](#פורמטר-מותאם-אישית)
19. [תוכנית מעבר שלב-אחר-שלב](#תוכנית-מעבר-שלב-אחר-שלב)
20. [בעיות צפויות ופתרונות](#בעיות-צפויות-ופתרונות)

---

## רקע ומוטיבציה

כארגון גדול עם מונורפו המשלב ספריות Python רבות, חבילות משותפות, פרויקטי Vue.js, ותלויות מורכבות - הגעתם לנקודה שבה הכלים הקיימים כבר לא מספיקים. המצב הנוכחי כנראה נראה כך:

- **כל שינוי קטן גורם ל-CI לרוץ שעות** - כי אין אינטליגנציה שמבינה מה *באמת* השתנה
- **פרסום חבילה ספציפית מצריך עבודה ידנית** - סקריפטים מיוחדים, תיאום בין צוותים
- **אין ניראות לתלויות בין הפרויקטים** - לא ברור מה ישבור אם תשנו ספרייה בסיסית
- **Poetry מאט את תהליכי ה-CI** - `poetry install` לוקח זמן רב, אין cache חכם
- **אין אכיפה של גבולות ארכיטקטוניים** - כל פרויקט יכול לייבא מכל מקום

**הפתרון המוצע:** שילוב של NX כמנהל workspace חכם, UV כמנהל חבילות מהיר, Ruff ליניט וסדר בקוד, ופורמטר מותאם לצרכי הארגון.

---

## NX - מה זה ואיך זה עובד

### הגדרה

NX הוא **מערכת Build חכמה למונורפו** שמוסיפה שכבת אינטליגנציה מעל הכלים הקיימים שלכם. הוא **לא מחליף** את Poetry/UV, pytest, או כל כלי אחר - הוא **עוטף אותם** ומוסיף:

- **Cache חכם**: אם הקוד לא השתנה - תוצאת הריצה הקודמת מוחזרת מיד
- **Affected Detection**: מבין בדיוק אילו פרויקטים הושפעו משינוי
- **Task Orchestration**: מריץ משימות במקביל בסדר הנכון
- **Dependency Graph**: מפה ויזואלית של כל התלויות במונורפו
- **Release Management**: פרסום סלקטיבי של ספריות ספציפיות

### המודל המנטלי

```
Workspace (המונורפו כולו)
├── Projects (פרויקט בודד - ספרייה/אפליקציה)
│   ├── Targets (משימה - build/test/lint/publish)
│   │   ├── Executor (מי מריץ את המשימה)
│   │   ├── Options (הגדרות המשימה)
│   │   ├── Inputs (מה משפיע על ה-cache)
│   │   └── Outputs (מה נשמר ב-cache)
│   └── Tags (תגיות ארגוניות)
└── nx.json (הגדרות ברמת ה-workspace)
```

NX בונה **Project Graph** - גרף ישיר שמשקף את התלויות בין הפרויקטים. מ-Project Graph הוא גוזר **Task Graph** - את הסדר שבו צריך להריץ משימות.

---

## מושגי ליבה ב-NX

### Workspace

ה-Workspace הוא ה-Git repository כולו. NX מזהה אותו לפי קובץ `nx.json` בשורש. כל פרויקט ב-workspace מזוהה על ידי נוכחות של `project.json` או `package.json` בתיקייתו.

### Projects

פרויקט הוא יחידה לוגית בתוך ה-workspace - ספרייה Python, אפליקציית Vue, שירות backend. לכל פרויקט יש:
- **שם ייחודי** (name)
- **תיקייה** (root)
- **סוג** (application/library)
- **Tags** לארגון
- **Targets** להרצה

### Targets

Target הוא פעולה שניתן להריץ על פרויקט. דוגמאות:
- `build` - בנייה
- `test` - הרצת בדיקות
- `lint` - בדיקת קוד
- `publish` - פרסום לרגיסטרי

```bash
# הרצת target ספציפי על פרויקט ספציפי
nx test my-python-lib

# הרצת target על כל הפרויקטים
nx run-many -t test

# הרצה רק על מה שהשתנה
nx affected -t test
```

### Executors

Executor הוא ה"מנוע" שמריץ target. זה יכול להיות:
1. **Executor מובנה** - `@nx/vite:build`, `@nx/jest:jest`
2. **`nx:run-commands`** - הרצת פקודת shell כלשהי (הכי שימושי ל-Python)
3. **`command`** shorthand - קיצור לפקודת shell פשוטה

```json
// project.json לפרויקט Python
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

Generator הוא סקריפט TypeScript שיוצר או משנה קוד. הוא עוזר ל:
- יצירת פרויקט חדש עם כל הקבצים הדרושים בצורה סטנדרטית
- הוספת פיצ'ר לפרויקט קיים (למשל הוספת Storybook)
- אכיפת סטנדרטים ארגוניים

```bash
# יצירת אפליקציית Vue חדשה
nx g @nx/vue:application apps/my-app --bundler=vite --style=scss

# יצירת ספריית Vue משותפת
nx g @nx/vue:library libs/ui-components --publishable --importPath=@myorg/ui

# רישום generator מותאם
nx g @myorg/custom-plugin:python-lib libs/my-new-lib
```

### Plugins

Plugin הוא חבילה שמוסיפה ל-NX:
- Executors חדשים
- Generators חדשים
- **Inference** - זיהוי אוטומטי של tasks מתוך קבצי הגדרות

**Plugins רלוונטיים לפרויקט שלכם:**

| Plugin | מטרה |
|--------|------|
| `@nx/vue` | תמיכה ב-Vue.js - generators ו-executors |
| `@nx/vite` | Vite build system |
| `@nx/eslint` | ESLint integration |
| `@nx/playwright` | E2E testing |

> **הערה חשובה:** אין plugin רשמי של NX ל-Python. נשתמש ב-`nx:run-commands` להגדרת targets ידנית, שזו הגישה הנכונה למונורפו Python מקצועי.

---

## הוספת NX למונורפו קיים

### שלב 1: התקנת NX

```bash
# בשורש ה-monorepo
npm init -y  # רק אם אין package.json בשורש
npm install --save-dev nx
```

> **למה npm ולא poetry?** NX הוא כלי JavaScript/TypeScript. הוא ינהל את ה-Node.js tooling (NX עצמו, Vue.js) ובנוסף יתזמר את כלי ה-Python שלכם. זהו ה-"שכבת תיאום" בין שני העולמות.

### שלב 2: יצירת nx.json בסיסי

```json
// nx.json
{
  "defaultBase": "main",
  "namedInputs": {
    "default": ["{projectRoot}/**/*", "sharedGlobals"],
    "production": [
      "default",
      "!{projectRoot}/**/*.spec.py",
      "!{projectRoot}/**/*.test.ts",
      "!{projectRoot}/**/test_*.py",
      "!{projectRoot}/**/__tests__/**"
    ],
    "sharedGlobals": [
      "{workspaceRoot}/uv.lock",
      "{workspaceRoot}/pyproject.toml"
    ]
  },
  "targetDefaults": {
    "build": {
      "cache": true,
      "dependsOn": ["^build"],
      "inputs": ["production", "^production"]
    },
    "test": {
      "cache": true,
      "inputs": ["default", "^production"]
    },
    "lint": {
      "cache": true,
      "inputs": ["default", "{workspaceRoot}/.ruff.toml"]
    }
  },
  "parallel": 4
}
```

### שלב 3: יצירת project.json לכל פרויקט Python

בכל תיקיית פרויקט Python קיים, יוצרים `project.json`:

```json
// packages/my-python-lib/project.json
{
  "name": "my-python-lib",
  "root": "packages/my-python-lib",
  "sourceRoot": "packages/my-python-lib/src",
  "projectType": "library",
  "tags": ["lang:python", "scope:shared"],
  "targets": {
    "build": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv build",
        "cwd": "{projectRoot}"
      },
      "outputs": ["{projectRoot}/dist"],
      "cache": true
    },
    "test": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run pytest tests/ -v --tb=short",
        "cwd": "{projectRoot}"
      },
      "cache": true,
      "inputs": [
        "{projectRoot}/**/*.py",
        "{projectRoot}/pyproject.toml"
      ]
    },
    "lint": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run ruff check .",
        "cwd": "{projectRoot}"
      },
      "cache": true
    },
    "format": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run ruff format .",
        "cwd": "{projectRoot}"
      }
    },
    "type-check": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run mypy src/",
        "cwd": "{projectRoot}"
      },
      "cache": true
    }
  }
}
```

### שלב 4: עדכון .gitignore

```
# NX cache
.nx/cache
.nx/workspace-data

# Python
__pycache__/
*.pyc
.venv/
dist/
*.egg-info/
```

### שלב 5: אימות ה-workspace

```bash
# בדקו שNX מזהה את כל הפרויקטים
npx nx show projects

# בדקו את הגרף
npx nx graph

# הריצו בדיקות ידנית
npx nx test my-python-lib
```

---

## NX לפרויקטי Python

### ארכיטקטורת Python ב-NX

ב-NX ל-Python **אין executor חכם** כמו `@nx/jest` שמבין pytest. במקום זאת, משתמשים ב-`nx:run-commands` שמריץ פקודות shell. זה בעצם יתרון - **גמישות מלאה** לכל פקודת Python.

### מבנה תיקיות מומלץ

```
monorepo/
├── nx.json
├── package.json          # רק עבור NX עצמו
├── pyproject.toml        # workspace UV root
├── uv.lock               # lockfile משותף
├── .python-version       # גרסת Python לכל ה-workspace
├── packages/             # ספריות Python
│   ├── core-lib/
│   │   ├── project.json
│   │   ├── pyproject.toml
│   │   └── src/
│   │       └── core_lib/
│   ├── data-utils/
│   │   ├── project.json
│   │   ├── pyproject.toml
│   │   └── src/
│   └── api-client/
│       ├── project.json
│       ├── pyproject.toml
│       └── src/
├── apps/                 # אפליקציות
│   ├── backend-service/
│   │   ├── project.json
│   │   ├── pyproject.toml
│   │   └── src/
│   └── frontend-dashboard/  # Vue.js
│       ├── project.json
│       ├── package.json
│       └── src/
└── libs/                 # ספריות Vue.js/TS משותפות
    └── ui-components/
        ├── project.json
        ├── package.json
        └── src/
```

### תלויות בין ספריות Python

NX מזהה תלויות Python על ידי ניתוח `pyproject.toml` של כל פרויקט. כאשר `data-utils` תלוי ב-`core-lib`, NX יבנה גרף תלויות נכון.

```toml
# packages/data-utils/pyproject.toml
[project]
name = "data-utils"
version = "1.2.0"
dependencies = [
    "core-lib>=1.0.0",  # תלות פנימית
]

[tool.uv.sources]
core-lib = { workspace = true }  # NX + UV יבינו שזו תלות פנימית
```

### project.json מתקדם לפרויקט Python

```json
// packages/data-utils/project.json
{
  "name": "data-utils",
  "root": "packages/data-utils",
  "sourceRoot": "packages/data-utils/src",
  "projectType": "library",
  "tags": ["lang:python", "scope:data", "type:lib"],
  "implicitDependencies": ["core-lib"],
  "targets": {
    "build": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv build --no-sources",
        "cwd": "{projectRoot}"
      },
      "dependsOn": ["^build"],
      "outputs": ["{projectRoot}/dist"],
      "inputs": [
        "{projectRoot}/src/**/*.py",
        "{projectRoot}/pyproject.toml",
        "^production"
      ],
      "cache": true
    },
    "test": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run pytest tests/ -v --cov=src --cov-report=xml:coverage.xml",
        "cwd": "{projectRoot}"
      },
      "outputs": ["{projectRoot}/coverage.xml"],
      "cache": true
    },
    "lint": {
      "executor": "nx:run-commands",
      "options": {
        "commands": [
          "uv run ruff check .",
          "uv run ruff format --check ."
        ],
        "parallel": false,
        "cwd": "{projectRoot}"
      },
      "cache": true
    },
    "publish": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv publish --index internal",
        "cwd": "{projectRoot}"
      },
      "dependsOn": ["build"],
      "cache": false
    },
    "sync-deps": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv sync",
        "cwd": "{workspaceRoot}"
      },
      "cache": false
    }
  }
}
```

### כיצד NX מזהה תלויות Python אוטומטית

NX מנתח את ה-`pyproject.toml` ואת `[tool.uv.sources]` כדי לבנות את גרף התלויות. חשוב מאוד:

1. **תמיד הגדירו `[tool.uv.sources]`** עם `workspace = true` לתלויות פנימיות
2. **השתמשו ב-`implicitDependencies`** ב-`project.json` כ-fallback אם NX לא מזהה אוטומטית
3. **הריצו `nx graph`** לאחר כל שינוי מבני לוודא שהגרף נכון

```json
// project.json - הצהרת תלות מפורשת
{
  "name": "data-utils",
  "implicitDependencies": ["core-lib"],
  ...
}
```

---

## NX Affected - הכלי החכם לזיהוי שינויים

### איך זה עובד

`nx affected` פועל בשלושה שלבים:

1. **Git Diff**: משתמש ב-Git כדי לזהות קבצים שהשתנו בין שני commits
2. **Project Mapping**: ממפה כל קובץ לפרויקט שהוא שייך אליו
3. **Dependency Propagation**: מסמן כ"מושפע" כל פרויקט שתלוי בפרויקט שהשתנה

**דוגמה:** שינוי ב-`core-lib` יסמן כמושפעים גם את `data-utils` ואת `api-client` שתלויים בו.

### פקודות עיקריות

```bash
# בדיקות רק על מה שהשתנה
nx affected -t test

# בנייה של כל מה שהושפע
nx affected -t build

# לינט רק על מה שהשתנה - עם 5 worker threads
nx affected -t lint --parallel=5

# הצגת אילו פרויקטים מושפעים (בלי להריץ שום דבר)
nx affected --print-affected

# השוואה מול branch ספציפי
nx affected -t test --base=main --head=HEAD

# כולל שינויים שעדיין לא עשיתם commit
nx affected -t test --uncommitted

# הפסקה בכשל ראשון
nx affected -t test --nxBail
```

### הגדרת Base ו-Head ב-GitLab

ב-GitLab CI/CD, יש להגדיר את משתני הסביבה הנכונים:

```yaml
# .gitlab-ci.yml
variables:
  NX_HEAD: "$CI_COMMIT_SHA"
  NX_BASE: "${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}"
```

- **NX_HEAD**: ה-commit הנוכחי
- **NX_BASE**: ה-commit האחרון שנבדק בהצלחה ב-main, או base ה-MR

### הגדרת base חכמה ל-CI

**הבעיה:** אם ה-CI נכשל ב-commit X, ה-affected עבור commit X+1 יצטרך לבדוק גם את שינויי X. ב-GitLab, `CI_COMMIT_BEFORE_SHA` מחזיר את ה-commit הקודם, אבל **לא בהכרח** את הצלחתי האחרון.

**הפתרון המומלץ:** שמור את ה-commit של הריצה המוצלחת האחרונה:

```bash
# בסוף pipeline מוצלח - שמור את ה-SHA
echo "$CI_COMMIT_SHA" > .last-successful-sha
git push origin main  # או שמור ב-GitLab CI variable
```

### הגדרת `projectsAffectedByDependencyUpdates`

כאשר `uv.lock` משתנה, ברירת המחדל היא לסמן **כל** הפרויקטים כמושפעים. ניתן להגדיר התנהגות חכמה יותר:

```json
// nx.json
{
  "affected": {
    "defaultBase": "main"
  },
  "targetDefaults": {
    "test": {
      "inputs": [
        "{projectRoot}/**/*.py",
        "{projectRoot}/pyproject.toml",
        "{workspaceRoot}/uv.lock"
      ]
    }
  }
}
```

### .nxignore - הוצאת קבצים מהחישוב

צרו קובץ `.nxignore` בשורש לקבצים שלא צריכים לגרום ל-affected:

```
# .nxignore
*.md
*.txt
docs/
*.drawio
**/*.png
**/*.jpg
scripts/maintenance/
```

---

## NX Release - ניהול גרסאות ופרסום סלקטיבי

### סקירה כללית

`nx release` הוא אחד הכלים החשובים ביותר לארגון שלכם. הוא מאפשר:

- **Versioning**: עדכון אוטומטי של מספרי גרסה
- **Changelog Generation**: יצירת changelog מתוך commit messages
- **Publishing**: פרסום לרגיסטרי (PyPI, GitLab Package Registry)
- **Selective Release**: פרסום ספריות ספציפיות בלבד

### שלושת שלבי ה-Release

```
1. nx release version   → עדכן גרסה ב-pyproject.toml / package.json
2. nx release changelog → צור/עדכן CHANGELOG.md
3. nx release publish   → פרסם לרגיסטרי
```

או הכל ביחד:
```bash
nx release --dry-run   # תמיד קודם dry-run!
nx release
```

### הגדרת Release ב-nx.json

#### מצב Independent (כל ספרייה בגרסה עצמאית - **מומלץ לכם**)

```json
// nx.json
{
  "release": {
    "projects": ["packages/*"],
    "projectsRelationship": "independent",
    "changelog": {
      "projectChangelogs": true,
      "workspaceChangelog": false
    },
    "version": {
      "conventionalCommits": true,
      "updateDependents": "always"
    },
    "releaseTag": {
      "pattern": "{projectName}@{version}"
    },
    "git": {
      "commit": true,
      "commitMessage": "chore(release): {version}",
      "tag": true
    }
  }
}
```

#### Release Groups - קבוצות עם תצורות שונות

```json
// nx.json - מספר קבוצות שחרור
{
  "release": {
    "groups": {
      "python-libs": {
        "projects": ["packages/core-lib", "packages/data-utils", "packages/api-client"],
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
      "vue-libs": {
        "projects": ["libs/*"],
        "projectsRelationship": "fixed",
        "releaseTag": {
          "pattern": "ui-v{version}"
        }
      }
    }
  }
}
```

### פרסום ספריות Python ספציפיות בלבד

```bash
# פרסום ספרייה אחת
nx release --projects=core-lib

# פרסום מספר ספריות
nx release --projects=core-lib,data-utils

# פרסום קבוצה שלמה
nx release --groups=python-libs

# פרסום לפי tag
nx release --projects="tag:scope:data"

# Dry run תחילה - תמיד!
nx release --projects=core-lib --dry-run
```

### Conventional Commits - גרסאות אוטומטיות מ-commits

הגדירו את צוות המפתחים לכתוב commits בפורמט:

```
feat(core-lib): add new data processing pipeline
fix(data-utils): resolve edge case in date parsing
feat!: breaking change in API signature
chore: update dependencies
```

מיפוי ל-bump גרסה:
- `fix:` → Patch (1.0.0 → 1.0.1)
- `feat:` → Minor (1.0.0 → 1.1.0)
- `feat!:` או `BREAKING CHANGE:` → Major (1.0.0 → 2.0.0)

### Release עבור Python - Custom Version Actions

ל-Python, NX Release עובד נהדר עם `pyproject.toml`. צרו version action מותאם:

```typescript
// tools/release/python-version-action.ts
import { readFileSync, writeFileSync } from 'fs';
import * as toml from '@iarna/toml';

export async function readVersion(options: { projectRoot: string }) {
  const pyprojectPath = `${options.projectRoot}/pyproject.toml`;
  const content = toml.parse(readFileSync(pyprojectPath, 'utf-8'));
  return (content as any).project.version;
}

export async function writeVersion(options: {
  projectRoot: string;
  newVersion: string;
}) {
  const pyprojectPath = `${options.projectRoot}/pyproject.toml`;
  let content = readFileSync(pyprojectPath, 'utf-8');
  content = content.replace(
    /^version = ".*"$/m,
    `version = "${options.newVersion}"`
  );
  writeFileSync(pyprojectPath, content);
}
```

```json
// nx.json - שימוש ב-version action מותאם
{
  "release": {
    "groups": {
      "python-libs": {
        "projects": ["packages/*"],
        "version": {
          "versionActions": "./tools/release/python-version-action.ts"
        }
      }
    }
  }
}
```

### Programmatic API - סקריפט Release מלא

עבור תהליך release מורכב, השתמשו ב-API תכנותי:

```typescript
// tools/scripts/release.ts
import { releaseChangelog, releasePublish, releaseVersion } from 'nx/release';
import * as yargs from 'yargs';

(async () => {
  const options = await yargs
    .option('version', { type: 'string', description: 'Version specifier' })
    .option('dryRun', { alias: 'd', type: 'boolean', default: true })
    .option('projects', { type: 'string', description: 'Comma-separated project names' })
    .option('groups', { type: 'string', description: 'Release group name' })
    .parseAsync();

  // שלב 1: Versioning
  const { workspaceVersion, projectsVersionData, releaseGraph } =
    await releaseVersion({
      specifier: options.version,
      dryRun: options.dryRun,
      projects: options.projects ? options.projects.split(',') : undefined,
      groups: options.groups ? [options.groups] : undefined,
    });

  // שלב 2: Changelog
  await releaseChangelog({
    releaseGraph,
    versionData: projectsVersionData,
    version: workspaceVersion,
    dryRun: options.dryRun,
  });

  if (!options.dryRun) {
    // שלב 3: Build Python packages
    const projectsList = Object.keys(projectsVersionData).join(',');
    console.log(`Building: ${projectsList}`);

    // שלב 4: Publish
    const publishResults = await releasePublish({
      releaseGraph,
      dryRun: options.dryRun,
    });

    const allSuccess = Object.values(publishResults).every((r: any) => r.code === 0);
    process.exit(allSuccess ? 0 : 1);
  }
})();
```

```bash
# הרצה:
npx tsx tools/scripts/release.ts --projects=core-lib --dryRun
npx tsx tools/scripts/release.ts --groups=python-libs --dryRun=false
```

### הגדרת Registry מותאם (GitLab Package Registry)

```toml
# pyproject.toml שורש workspace
[[tool.uv.index]]
name = "gitlab-internal"
url = "https://gitlab.your-company.com/api/v4/groups/YOUR_GROUP_ID/-/packages/pypi/simple"
publish-url = "https://gitlab.your-company.com/api/v4/projects/YOUR_PROJECT_ID/packages/pypi"
authenticate = "always"
default = true
```

```bash
# משתני סביבה לאימות
export UV_INDEX_GITLAB_INTERNAL_USERNAME=__token__
export UV_INDEX_GITLAB_INTERNAL_PASSWORD=$GITLAB_DEPLOY_TOKEN

# פרסום
uv publish --index gitlab-internal
```

---

## NX Cache - מטמון מקומי ומרוחק

### Cache מקומי

NX שומר cache ב-`.nx/cache` בשורש ה-workspace. כאשר אתם מריצים task:

1. NX מחשב hash מ: קבצי source + קבצי config + משתני סביבה + גרסאות תלויות
2. אם ה-hash קיים ב-cache → מחזיר תוצאות קיימות מיד
3. אם לא → מריץ ומשמור תוצאות

```bash
# מחיקת cache מקומי
nx reset

# הרצה תוך דילוג על cache
nx test my-lib --skipNxCache
```

### הגדרת Cache Inputs ו-Outputs

```json
// project.json
{
  "targets": {
    "test": {
      "cache": true,
      "inputs": [
        "{projectRoot}/**/*.py",
        "!{projectRoot}/**/__pycache__/**",
        "{projectRoot}/pyproject.toml",
        "{workspaceRoot}/uv.lock"
      ],
      "outputs": [
        "{projectRoot}/coverage.xml",
        "{projectRoot}/.coverage",
        "{projectRoot}/htmlcov"
      ]
    },
    "build": {
      "cache": true,
      "inputs": [
        "{projectRoot}/src/**/*.py",
        "{projectRoot}/pyproject.toml"
      ],
      "outputs": ["{projectRoot}/dist"]
    }
  }
}
```

**חשוב:** Cache עובד רק עבור tasks **deterministic** - אותם inputs → תמיד אותם outputs. אל תכניסו ל-cache tasks שמשפיעים על מצב חיצוני (פרסום, deployment).

### Named Inputs - שימוש חוזר בהגדרות Input

```json
// nx.json
{
  "namedInputs": {
    "default": [
      "{projectRoot}/**/*",
      "!{projectRoot}/**/__pycache__/**",
      "!{projectRoot}/**/*.pyc",
      "!{projectRoot}/.venv/**"
    ],
    "production": [
      "default",
      "!{projectRoot}/**/*.spec.py",
      "!{projectRoot}/**/test_*.py",
      "!{projectRoot}/**/__tests__/**",
      "!{projectRoot}/tests/**"
    ],
    "pythonConfig": [
      "{workspaceRoot}/pyproject.toml",
      "{workspaceRoot}/uv.lock",
      "{workspaceRoot}/.python-version"
    ],
    "lintConfig": [
      "{workspaceRoot}/.ruff.toml",
      "{workspaceRoot}/pyproject.toml"
    ]
  }
}
```

### Remote Cache - מטמון משותף בין מפתחים ו-CI

**האפשרות הטובה ביותר לרשת סגורה:** Self-hosted Nx Cache Server.

**Option 1: NX Cloud Self-Hosted** (מחייב רישיון Enterprise):
```bash
# התקנה
npm install --save-dev @nrwl/nx-cloud
```

**Option 2: Remote Cache בגיטלאב** עם artifact sharing:

```yaml
# .gitlab-ci.yml - שמירת cache ב-GitLab
cache:
  key: "nx-cache-$CI_COMMIT_REF_SLUG"
  paths:
    - .nx/cache
  policy: pull-push
```

**Option 3: S3-compatible storage** (MinIO לרשת סגורה):

```json
// nx.json
{
  "nxCloudId": "your-cloud-id",
  "cacheDirectory": ".nx/cache"
}
```

הגדרת `NX_CACHE_DIRECTORY` ל-mounted shared volume ב-GitLab runners.

### Cache גרסאות מקסימלית

```json
// nx.json
{
  "maxCacheSize": "5GB"
}
```

---

## NX עם GitLab CI/CD

### Pipeline בסיסי

```yaml
# .gitlab-ci.yml
image: node:20-slim

variables:
  CI: "true"
  NX_HEAD: "$CI_COMMIT_SHA"
  NX_BASE: "${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}"
  UV_CACHE_DIR: ".uv-cache"
  PIP_NO_INDEX: "true"  # סביבה סגורה - לא לגשת ל-PyPI ישירות
  UV_INDEX_INTERNAL_USERNAME: "$INTERNAL_PYPI_USER"
  UV_INDEX_INTERNAL_PASSWORD: "$INTERNAL_PYPI_TOKEN"

before_script:
  - apt-get update && apt-get install -y python3 curl git
  - curl -LsSf https://astral.sh/uv/install.sh | sh
  - export PATH="$HOME/.cargo/bin:$PATH"
  - npm ci
  - uv sync --all-packages

cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - .nx/cache/
    - .uv-cache/
    - node_modules/

stages:
  - validate
  - test
  - build
  - release

lint:
  stage: validate
  interruptible: true
  only:
    - main
    - merge_requests
  script:
    - npx nx affected -t lint --parallel=4

type-check:
  stage: validate
  interruptible: true
  only:
    - main
    - merge_requests
  script:
    - npx nx affected -t type-check --parallel=2

test:
  stage: test
  interruptible: true
  only:
    - main
    - merge_requests
  script:
    - npx nx affected -t test --parallel=4
  artifacts:
    reports:
      junit: "**/coverage.xml"
    expire_in: 1 week

build:
  stage: build
  only:
    - main
    - merge_requests
  script:
    - npx nx affected -t build --parallel=4
  artifacts:
    paths:
      - "**/dist/"
    expire_in: 1 day

release-libs:
  stage: release
  only:
    - main
  when: manual  # טריגר ידני לשחרור
  script:
    - npx tsx tools/scripts/release.ts --dryRun=false
  environment:
    name: production
```

### Pipeline מתקדם עם Matrix

```yaml
# בדיקה על מספר גרסאות Python
test-python:
  stage: test
  parallel:
    matrix:
      - PYTHON_VERSION: ["3.11", "3.12"]
  script:
    - uv python install $PYTHON_VERSION
    - uv python pin $PYTHON_VERSION
    - npx nx affected -t test
```

### Affected בין MR ל-main

```yaml
# עבור Merge Request
.mr-config: &mr-config
  only:
    - merge_requests
  variables:
    NX_BASE: "$CI_MERGE_REQUEST_DIFF_BASE_SHA"
    NX_HEAD: "$CI_COMMIT_SHA"

# עבור push ישיר ל-main
.main-config: &main-config
  only:
    - main
  variables:
    NX_BASE: "$CI_COMMIT_BEFORE_SHA"
    NX_HEAD: "$CI_COMMIT_SHA"

test-mr:
  <<: *mr-config
  stage: test
  script:
    - npx nx affected -t test

test-main:
  <<: *main-config
  stage: test
  script:
    - npx nx affected -t test
```

### NX Graph ב-Pipeline

```yaml
# יצירת graph artifact לדיבוג
dependency-graph:
  stage: validate
  script:
    - npx nx graph --file=nx-graph.json
  artifacts:
    paths:
      - nx-graph.json
    expire_in: 1 week
  only:
    - merge_requests
```

---

## NX עם Vue.js

### התקנת @nx/vue Plugin

```bash
nx add @nx/vue
```

### יצירת אפליקציית Vue חדשה

```bash
nx g @nx/vue:application apps/my-dashboard \
  --bundler=vite \
  --style=scss \
  --routing=true \
  --unitTestRunner=vitest \
  --e2eTestRunner=playwright \
  --tags="lang:ts,type:app,scope:dashboard"
```

### יצירת ספריית Vue משותפת

```bash
nx g @nx/vue:library libs/ui-components \
  --bundler=vite \
  --publishable \
  --importPath=@myorg/ui-components \
  --unitTestRunner=vitest \
  --tags="lang:ts,type:lib,scope:shared"
```

### project.json של אפליקציית Vue

```json
// apps/my-dashboard/project.json
{
  "name": "my-dashboard",
  "root": "apps/my-dashboard",
  "sourceRoot": "apps/my-dashboard/src",
  "projectType": "application",
  "tags": ["lang:ts", "type:app", "scope:dashboard"],
  "targets": {
    "build": {
      "executor": "@nx/vite:build",
      "options": {
        "outputPath": "dist/apps/my-dashboard",
        "buildLibsFromSource": true
      },
      "configurations": {
        "production": {
          "mode": "production"
        },
        "development": {
          "mode": "development",
          "sourcemap": true
        }
      },
      "cache": true,
      "inputs": ["production", "^production"],
      "outputs": ["{workspaceRoot}/dist/apps/my-dashboard"]
    },
    "serve": {
      "executor": "@nx/vite:dev-server",
      "defaultConfiguration": "development",
      "options": {
        "buildTarget": "my-dashboard:build"
      },
      "configurations": {
        "development": {
          "buildTarget": "my-dashboard:build:development",
          "hmr": true
        }
      }
    },
    "test": {
      "executor": "@nx/vite:test",
      "options": {
        "reportsDirectory": "../../coverage/apps/my-dashboard"
      },
      "cache": true
    },
    "lint": {
      "executor": "@nx/eslint:lint",
      "options": {
        "lintFilePatterns": ["apps/my-dashboard/**/*.{ts,vue}"]
      },
      "cache": true
    }
  }
}
```

### vite.config.ts לאפליקציית Vue ב-NX

```typescript
// apps/my-dashboard/vite.config.ts
import { defineConfig } from 'vite';
import vue from '@vitejs/plugin-vue';
import { nxViteTsPaths } from '@nx/vite/plugins/nx-tsconfig-paths.plugin';

export default defineConfig({
  root: __dirname,
  cacheDir: '../../node_modules/.vite/apps/my-dashboard',
  plugins: [
    vue(),
    nxViteTsPaths(),  // חשוב! מאפשר ל-Vue לייבא מ-@myorg/ui-components
  ],
  build: {
    outDir: '../../dist/apps/my-dashboard',
    emptyOutDir: true,
    reportCompressedSize: true,
    commonjsOptions: {
      transformMixedEsModules: true,
    },
  },
  test: {
    globals: true,
    environment: 'jsdom',
    include: ['src/**/*.{test,spec}.{js,mjs,cjs,ts,mts,cts,jsx,tsx}'],
    coverage: {
      reportsDirectory: '../../coverage/apps/my-dashboard',
      provider: 'v8',
    },
  },
});
```

### שילוב Vue עם Python Backend

```json
// apps/my-dashboard/project.json - target שמריץ frontend ו-backend ביחד
{
  "targets": {
    "serve-full": {
      "executor": "nx:run-commands",
      "options": {
        "commands": [
          "nx serve my-dashboard",
          "nx serve backend-service"
        ],
        "parallel": true
      }
    }
  }
}
```

---

## מבנה project.json

### מדריך שלם לכל השדות

```json
{
  // מזהה ייחודי של הפרויקט ב-workspace
  "name": "my-library",

  // תיקיית שורש הפרויקט (יחסית ל-workspace root)
  "root": "packages/my-library",

  // תיקיית קוד המקור
  "sourceRoot": "packages/my-library/src",

  // סוג הפרויקט
  "projectType": "library",  // או "application"

  // תגיות לארגון ואכיפת כללים
  "tags": ["lang:python", "scope:core", "type:lib"],

  // תלויות שNX לא יגלה אוטומטית
  "implicitDependencies": ["shared-config", "!excluded-project"],

  // Targets - המשימות האפשריות
  "targets": {

    "build": {
      // מי מריץ
      "executor": "nx:run-commands",
      // או:
      // "command": "uv build",  // קיצור לפקודה פשוטה

      "options": {
        "command": "uv build --no-sources",
        "cwd": "{projectRoot}",
        // עבור מספר פקודות:
        "commands": [
          "uv run python -m build",
          "uv run twine check dist/*"
        ],
        "parallel": false,
        // משתני סביבה
        "env": {
          "PYTHONPATH": "{workspaceRoot}/packages"
        }
      },

      // תצורות שונות
      "configurations": {
        "production": {
          "options": {
            "command": "uv build --no-sources --wheel"
          }
        },
        "development": {
          "options": {
            "command": "uv build --no-sources --sdist"
          }
        }
      },

      // תצורת ברירת מחדל
      "defaultConfiguration": "production",

      // tasks שצריכים לרוץ לפני
      "dependsOn": [
        "^build",     // build של כל התלויות קודם
        "lint",       // lint של הפרויקט הנוכחי קודם
        {
          "target": "build",
          "projects": "dependencies",
          "params": "forward"
        }
      ],

      // cache
      "cache": true,

      // מה משפיע על cache
      "inputs": [
        "{projectRoot}/src/**/*.py",
        "{projectRoot}/pyproject.toml",
        "^production",  // production inputs של תלויות
        {
          "env": "BUILD_VERSION"
        },
        {
          "externalDependencies": ["uv", "python"]
        }
      ],

      // מה נשמר ב-cache
      "outputs": [
        "{projectRoot}/dist/**",
        "{projectRoot}/*.egg-info"
      ]
    },

    "test": {
      "executor": "nx:run-commands",
      "options": {
        "command": "uv run pytest tests/ -v --tb=short --junitxml=test-results.xml",
        "cwd": "{projectRoot}"
      },
      "cache": true,
      "inputs": [
        "{projectRoot}/**/*.py",
        "!{projectRoot}/**/__pycache__/**",
        "{projectRoot}/pyproject.toml",
        "{workspaceRoot}/uv.lock"
      ],
      "outputs": [
        "{projectRoot}/test-results.xml",
        "{projectRoot}/.coverage"
      ]
    }
  }
}
```

---

## מבנה nx.json

### מדריך שלם

```json
{
  // base branch לחישוב affected
  "defaultBase": "main",

  // מספר tasks מקסימלי במקביל
  "parallel": 4,

  // תיקיית cache מקומי
  "cacheDirectory": ".nx/cache",

  // גודל cache מקסימלי
  "maxCacheSize": "10GB",

  // Input groups לשימוש חוזר
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
      "{workspaceRoot}/uv.lock",
      "{workspaceRoot}/pyproject.toml",
      "{workspaceRoot}/.python-version"
    ]
  },

  // ברירות מחדל לכל ה-targets
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"],
      "cache": true,
      "inputs": ["production", "^production"]
    },
    "test": {
      "cache": true,
      "inputs": ["default", "^production"]
    },
    "lint": {
      "cache": true,
      "inputs": [
        "default",
        "{workspaceRoot}/.ruff.toml",
        "{workspaceRoot}/pyproject.toml"
      ]
    },
    "e2e": {
      "cache": false  // E2E לא מועיל ל-cache
    }
  },

  // פלאגינים
  "plugins": [
    "@nx/vue/plugin",
    "@nx/vite/plugin",
    {
      "plugin": "@nx/eslint/plugin",
      "options": {
        "targetName": "lint"
      },
      "include": ["apps/**/*", "libs/**/*"],
      "exclude": ["**/*-e2e/**/*"]
    }
  ],

  // ברירות מחדל לגנרטורים
  "generators": {
    "@nx/vue:library": {
      "linter": "eslint",
      "unitTestRunner": "vitest"
    },
    "@nx/vue:application": {
      "bundler": "vite",
      "style": "scss"
    }
  },

  // הגדרות Release
  "release": {
    "projects": ["packages/*", "libs/*"],
    "projectsRelationship": "independent",
    "changelog": {
      "projectChangelogs": true,
      "workspaceChangelog": {
        "createRelease": false
      }
    },
    "version": {
      "conventionalCommits": true,
      "updateDependents": "always"
    },
    "releaseTag": {
      "pattern": "{projectName}@{version}"
    },
    "git": {
      "commit": true,
      "commitMessage": "chore(release): publish packages",
      "tag": true
    }
  }
}
```

---

## Tags ו-Implicit Dependencies

### מהם Tags?

Tags הם תוויות מטא-דטה שמוסיפים לפרויקטים ב-`project.json`. הם משמשים ל:
1. **ארגון** - מי אחראי על מה
2. **אכיפת כללים** - מי יכול לייבא ממי
3. **Filtering** - `nx affected` על תת-קבוצה

### מוסכמות נוסכמות לשמות Tags

```json
// convention: "category:value"
{
  "tags": [
    "lang:python",          // שפת תכנות
    "lang:typescript",
    "scope:core",           // אחריות/תחום
    "scope:data",
    "scope:frontend",
    "type:lib",             // סוג הפרויקט
    "type:app",
    "type:e2e",
    "team:backend",         // בעלות צוות
    "team:frontend",
    "visibility:public",    // נראות
    "visibility:internal"
  ]
}
```

### כללי תלויות (Module Boundaries)

```javascript
// .eslintrc.json - לפרויקטי TypeScript/Vue
{
  "rules": {
    "@nx/enforce-module-boundaries": [
      "error",
      {
        "depConstraints": [
          // App יכול להשתמש בכל lib, אבל לא ב-app אחר
          {
            "sourceTag": "type:app",
            "onlyDependOnLibsWithTags": ["type:lib", "type:util"]
          },
          // Core lib לא יכול לייבא מ-scope:data
          {
            "sourceTag": "scope:core",
            "notDependOnLibsWithTags": ["scope:data"]
          },
          // Frontend יכול רק מ-shared ומ-frontend
          {
            "sourceTag": "scope:frontend",
            "onlyDependOnLibsWithTags": ["scope:frontend", "scope:shared"]
          }
        ]
      }
    ]
  }
}
```

### Implicit Dependencies - תלויות ידניות

```json
// project.json
{
  "name": "backend-service",
  "implicitDependencies": [
    "shared-config",    // תמיד תלוי
    "core-lib",
    "!legacy-module"    // ה-! אומר: לא תלוי (override)
  ]
}
```

**מתי להשתמש ב-`implicitDependencies`:**
- כאשר NX לא מזהה אוטומטית את התלות (למשל תלות ב-shared config file)
- כאשר שינוי בפרויקט X צריך לגרום ל-rebuild של Y גם אם אין import ישיר

### Filtering לפי Tags ב-CI

```bash
# הרץ בדיקות רק על Python projects
nx run-many -t test --projects="tag:lang:python"

# הרץ build רק על team:backend
nx affected -t build --projects="tag:team:backend"

# הרץ לכולם חוץ מ-e2e
nx run-many -t lint --exclude="tag:type:e2e"
```

---

## גרף תלויות - Dependency Graph

### הצגת הגרף

```bash
# פתח בדפדפן
npx nx graph

# פוקוס על פרויקט ספציפי
npx nx graph --focus=core-lib

# שמור כ-JSON לניתוח
npx nx graph --file=dependency-graph.json

# הצגת גרף tasks
npx nx build my-app --graph
npx nx affected -t test --graph
```

### הבנת הגרף

הגרף מציג:
- **Projects**: עיגולים, כל פרויקט ב-workspace
- **Dependencies**: חצים, כיוון התלות
- **Affected**: הדגשה של פרויקטים מושפעים

הגרף עוזר לאבחן:
- **Circular dependencies**: תלויות מעגליות (בעיה!)
- **Heavy hitters**: פרויקטים שהרבה תלויים בהם
- **Isolation**: פרויקטים שאין להם תלויות (נכסים ניתנים לפרסום עצמאי)

### שימוש בגרף ל-Release החלטות

לפני שחרור ספרייה, בדקו מי תלוי בה:

```bash
# ראו את כל הפרויקטים שתלויים ב-core-lib
npx nx graph --focus=core-lib
# ואז הסתכלו על החצים הנכנסים
```

---

## מלכודות נפוצות ואיך להימנע מהן

### 1. Cache Miss מתמיד

**הבעיה:** Tasks רצים כל פעם למרות שלא השתנה כלום.

**הסיבות הנפוצות:**
- `inputs` לא מוגדרים נכון (כולל קבצים volatile כמו timestamps)
- קובץ שמשתנה תמיד (כמו `__pycache__`) כלול ב-inputs

**הפתרון:**
```json
// הוסיפו exclusions מפורשות
"inputs": [
  "{projectRoot}/**/*.py",
  "!{projectRoot}/**/__pycache__/**",
  "!{projectRoot}/**/*.pyc",
  "!{projectRoot}/.venv/**",
  "!{projectRoot}/**/.coverage"
]
```

```bash
# בדקו למה cache miss קרה
NX_VERBOSE_LOGGING=true nx test my-lib
```

### 2. Project לא מזוהה

**הבעיה:** NX לא רואה פרויקט מסוים.

**הסיבות:**
- אין `project.json` בתיקייה
- תיקייה בתוך `.nxignore` או `.gitignore`
- שם פרויקט כפול

**הפתרון:**
```bash
# בדקו אילו פרויקטים NX מזהה
npx nx show projects --verbose

# בדקו פרויקט ספציפי
npx nx show project my-lib --web
```

### 3. Circular Dependencies

**הבעיה:** A תלוי ב-B, B תלוי ב-A.

**הסיבות:** ארכיטקטורה שגויה.

**הפתרון:**
```bash
# NX יתריע על circular deps
npx nx graph  # יציין את המעגל ויסמן אדום
```
פתרון: חלצו את הקוד המשותף לספרייה שלישית C.

### 4. `uv.lock` גורם ל-Affected לכל הפרויקטים

**הבעיה:** כל שינוי ב-`uv.lock` גורם ל-affected לכל ה-workspace.

**הפתרון:** הגדירו `sharedGlobals` כ-named input והכניסו רק לפרויקטים שצריכים:

```json
// nx.json
{
  "namedInputs": {
    "sharedGlobals": ["{workspaceRoot}/uv.lock"],
    "default": ["{projectRoot}/**/*"]  // ללא sharedGlobals!
  },
  "targetDefaults": {
    "test": {
      "inputs": ["default", "sharedGlobals"]
    }
  }
}
```

### 5. Build Order שגוי

**הבעיה:** `data-utils` נבנה לפני `core-lib` שהוא תלוי בו.

**הפתרון:** ודאו שיש `dependsOn: ["^build"]` ב-build target:

```json
// nx.json targetDefaults
{
  "targetDefaults": {
    "build": {
      "dependsOn": ["^build"]
    }
  }
}
```

### 6. Task רץ גם ב-Cache Hit

**הבעיה:** ה-task מריץ משהו ב-side-effect (כמו עדכון בסיס נתונים).

**הפתרון:** task שעושה side-effect **לא צריך** להיות cacheable:
```json
{
  "targets": {
    "migrate-db": {
      "cache": false  // אל תכניסו ל-cache!
    }
  }
}
```

### 7. NX לא מזהה תלויות Python

**הבעיה:** שינוי ב-`core-lib` לא מסמן את `data-utils` כ-affected.

**הסיבה:** NX לא ניתח את `pyproject.toml` נכון.

**הפתרון:**
```json
// packages/data-utils/project.json
{
  "implicitDependencies": ["core-lib"]
}
```

ו/או וודאו ב-`pyproject.toml`:
```toml
[tool.uv.sources]
core-lib = { workspace = true }
```

### 8. Release מפרסם את כל הספריות

**הבעיה:** `nx release` פרסם הכל, לא רק מה שהשתנה.

**הפתרון:**
```bash
# תמיד ספציפיים
nx release --projects=core-lib --dry-run
nx release --groups=python-libs --dry-run

# ולעולם לא
nx release  # ללא --projects ב-release ידני
```

---

## UV - מעבר מ-Poetry

### מה זה UV?

UV הוא מנהל חבילות Python מודרני כתוב ב-Rust, שמחליף:
- `poetry` - ניהול פרויקטים
- `pip` + `pip-tools` - התקנת חבילות
- `pyenv` - ניהול גרסאות Python
- `virtualenv` - virtual environments
- `twine` - פרסום חבילות

**יתרונות על Poetry:**
- **מהיר פי 10-100** מ-pip (פי 3-5 מ-poetry)
- **Workspace support** מובנה (כמו Cargo ב-Rust)
- **Universal lockfile** שעובד cross-platform
- **פשטות** - פחות הגדרות, יותר intuitive

### טבלת השוואת פקודות

| Poetry | UV | תיאור |
|--------|----|-------|
| `poetry init` | `uv init` | אתחול פרויקט |
| `poetry add requests` | `uv add requests` | הוספת תלות |
| `poetry add --dev pytest` | `uv add --dev pytest` | תלות פיתוח |
| `poetry install` | `uv sync` | התקנת תלויות |
| `poetry run pytest` | `uv run pytest` | הרצה ב-venv |
| `poetry build` | `uv build` | בנייה |
| `poetry publish` | `uv publish` | פרסום |
| `poetry shell` | `uv run bash` / `source .venv/bin/activate` | כניסה ל-venv |
| `poetry lock` | `uv lock` | עדכון lockfile |
| `poetry update` | `uv lock --upgrade` | עדכון תלויות |
| `poetry env info` | `uv python list` | מידע על Python |

### pyproject.toml - השוואה

```toml
# Poetry - לפני
[tool.poetry]
name = "my-lib"
version = "1.0.0"
description = "My library"
authors = ["Team <team@company.com>"]

[tool.poetry.dependencies]
python = "^3.11"
requests = "^2.28.0"
pandas = ">=2.0.0"

[tool.poetry.group.dev.dependencies]
pytest = "^7.0.0"
ruff = "^0.1.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

```toml
# UV - אחרי
[project]
name = "my-lib"
version = "1.0.0"
description = "My library"
authors = [{ name = "Team", email = "team@company.com" }]
requires-python = ">=3.11"
dependencies = [
    "requests>=2.28.0",
    "pandas>=2.0.0",
]

[dependency-groups]
dev = [
    "pytest>=7.0.0",
    "ruff>=0.1.0",
]

[build-system]
requires = ["hatchling"]  # או setuptools
build-backend = "hatchling.build"

[tool.uv]
dev-dependencies = []
```

### הגדרת UV Workspace (מחליף Poetry workspaces)

```toml
# pyproject.toml ב-שורש ה-workspace
[project]
name = "monorepo-root"
version = "0.0.0"
requires-python = ">=3.11"

[tool.uv.workspace]
members = [
    "packages/*",
    "apps/backend-*"
]
exclude = [
    "packages/legacy-*"
]
```

כל חבר workspace מוסיף:
```toml
# packages/data-utils/pyproject.toml
[project]
name = "data-utils"
version = "1.2.0"
dependencies = [
    "core-lib",
    "pandas>=2.0.0"
]

[tool.uv.sources]
# הפנייה לחבר workspace אחר
core-lib = { workspace = true }
```

### תהליך המעבר מ-Poetry ל-UV

#### שלב 1: התקנת UV

```bash
# Linux/macOS
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -c "irm https://astral.sh/uv/install.ps1 | iex"

# לאחר התקנה
uv --version
```

#### שלב 2: המרת pyproject.toml

```bash
# ב-UV יש כלי migration (experimental):
uv init --from-poetry

# או ידנית:
# 1. שנה את [tool.poetry] ל-[project]
# 2. שנה [tool.poetry.dependencies] ל-[project.dependencies]
# 3. שנה [tool.poetry.group.dev.dependencies] ל-[dependency-groups.dev]
# 4. שנה build-system
```

#### שלב 3: יצירת lockfile

```bash
# ב-שורש ה-workspace
uv lock

# זה יצור uv.lock שמחליף את poetry.lock
git add uv.lock
git rm poetry.lock
```

#### שלב 4: עדכון CI/CD

```yaml
# .gitlab-ci.yml - לפני (Poetry)
before_script:
  - pip install poetry
  - poetry install

# אחרי (UV)
before_script:
  - curl -LsSf https://astral.sh/uv/install.sh | sh
  - export PATH="$HOME/.cargo/bin:$PATH"
  - uv sync --all-packages
```

#### שלב 5: עדכון scripts

```bash
# לפני
poetry run pytest
poetry run python script.py

# אחרי
uv run pytest
uv run python script.py
```

### הגדרת Registry פרטי עם UV

```toml
# pyproject.toml שורש workspace - סביבה סגורה
[[tool.uv.index]]
name = "internal"
url = "https://pypi.internal.company.com/simple"
publish-url = "https://pypi.internal.company.com/upload"
authenticate = "always"
default = true  # שימוש ב-registry זה בלבד

# ביטול PyPI הציבורי בסביבה סגורה
[tool.uv]
index-url = "https://pypi.internal.company.com/simple"
no-index = false
```

```bash
# משתני סביבה לאימות
export UV_INDEX_INTERNAL_USERNAME="ci-user"
export UV_INDEX_INTERNAL_PASSWORD="$CI_TOKEN"

# הגדרה ב-.env מקומי (מחוץ ל-git)
echo "UV_INDEX_INTERNAL_PASSWORD=mytoken" >> .env
```

### UV ב-Docker

```dockerfile
# Dockerfile
FROM python:3.12-slim

# התקנת UV
COPY --from=ghcr.io/astral-sh/uv:latest /uv /uvx /bin/

# הגדרת workdir
WORKDIR /app

# העתקת lockfile וsync
COPY uv.lock pyproject.toml ./
COPY packages/my-service/pyproject.toml packages/my-service/
RUN uv sync --package my-service --no-dev

# העתקת קוד
COPY packages/my-service/src ./packages/my-service/src

CMD ["uv", "run", "--package", "my-service", "python", "-m", "my_service.main"]
```

---

## Ruff - לינטר מהיר לפייתון

### מה זה Ruff?

Ruff הוא linter ו-formatter לפייתון כתוב ב-Rust. הוא מחליף:
- **Flake8** + עשרות plugins
- **isort** (מיון imports)
- **pyupgrade** (עדכון syntax)
- **Black** (formatting)
- **pydocstyle** (בדיקת docstrings)

**מהיר פי 10-100 מ-Flake8**, עם תמיכה ב-800+ כללים.

### התקנה

```bash
# הוספה ל-dev dependencies של הפרויקט
uv add --dev ruff

# או ב-workspace root
uv add --dev ruff  # בשורש
```

### קובץ הגדרות מרכזי

```toml
# pyproject.toml שורש workspace (חל על כל הפרויקטים)
[tool.ruff]
line-length = 100
target-version = "py311"
exclude = [
    ".git",
    "__pycache__",
    ".venv",
    "dist",
    "build",
    "*.egg-info",
    "migrations",   # אם יש Django migrations
]

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "W",    # pycodestyle warnings
    "F",    # pyflakes
    "I",    # isort
    "B",    # flake8-bugbear
    "C4",   # flake8-comprehensions
    "UP",   # pyupgrade
    "N",    # pep8-naming
    "ANN",  # flake8-annotations (type hints)
    "S",    # flake8-bandit (security)
    "T20",  # flake8-print (no print statements)
]

ignore = [
    "E501",   # line too long (handled by formatter)
    "ANN101", # missing type annotation for self
    "S101",   # use of assert (ok in tests)
]

[tool.ruff.lint.per-file-ignores]
"tests/**/*.py" = ["ANN", "S"]     # בדיקות - רגישות יותר
"**/migrations/*.py" = ["ALL"]     # migrations - לא נוגעים
"scripts/*.py" = ["T20"]           # scripts - מותר print

[tool.ruff.lint.isort]
known-first-party = ["core_lib", "data_utils", "api_client"]
split-on-trailing-comma = true

[tool.ruff.format]
quote-style = "double"
indent-style = "space"
skip-magic-trailing-comma = false
line-ending = "auto"
```

### הגדרות per-project override

```toml
# packages/legacy-lib/pyproject.toml
# override הגדרות workspace ל-legacy code
[tool.ruff]
extend = "../../pyproject.toml"  # ירש מה-workspace

[tool.ruff.lint]
ignore = [
    "E501",    # legacy code with long lines
    "N",       # legacy naming conventions
    "ANN",     # no type hints in legacy
]
```

### שימוש

```bash
# בדיקה
ruff check .
uv run ruff check .

# תיקון אוטומטי
ruff check --fix .
uv run ruff check --fix .

# Formatting
ruff format .
uv run ruff format .

# בדיקת format בלבד (CI mode)
ruff format --check .

# הצגת כל הכללים הזמינים
ruff linter
```

### שילוב ב-NX project.json

```json
// nx.json targetDefaults
{
  "targetDefaults": {
    "lint": {
      "executor": "nx:run-commands",
      "options": {
        "commands": [
          "uv run ruff check .",
          "uv run ruff format --check ."
        ],
        "parallel": false,
        "cwd": "{projectRoot}"
      },
      "cache": true,
      "inputs": [
        "{projectRoot}/**/*.py",
        "{workspaceRoot}/pyproject.toml",
        "{workspaceRoot}/.ruff.toml"
      ]
    },
    "format": {
      "executor": "nx:run-commands",
      "options": {
        "commands": [
          "uv run ruff format .",
          "uv run ruff check --fix ."
        ],
        "parallel": false,
        "cwd": "{projectRoot}"
      },
      "cache": false
    }
  }
}
```

### הגדרת pre-commit hooks עם Ruff

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.9.0
    hooks:
      - id: ruff
        args: [--fix]
      - id: ruff-format
```

```bash
# התקנת pre-commit
uv add --dev pre-commit
uv run pre-commit install
```

---

## פורמטר מותאם אישית

### ארכיטקטורת הפורמטר

הפורמטר שלכם הוא סקריפט Python שמקבל תיקייה כארגומנט ומחיל formatters על כל הקוד בה.

### מבנה הפורמטר

```python
#!/usr/bin/env python3
# tools/formatter/format.py

"""
Company formatter - applies all formatters to a directory.
Usage: python format.py <directory> [--check] [--verbose]
"""

import argparse
import subprocess
import sys
from pathlib import Path


def run_command(cmd: list[str], cwd: Path, check: bool = False) -> tuple[int, str, str]:
    """Run a command and return (returncode, stdout, stderr)."""
    if check:
        # Check mode - only check, don't fix
        cmd = [c.replace('--fix', '--no-fix') for c in cmd]
        if '--fix' in cmd:
            cmd.remove('--fix')

    result = subprocess.run(
        cmd,
        cwd=cwd,
        capture_output=True,
        text=True
    )
    return result.returncode, result.stdout, result.stderr


def format_directory(directory: str, check_only: bool = False, verbose: bool = False) -> int:
    """Apply all formatters to the given directory."""
    target = Path(directory).resolve()

    if not target.exists():
        print(f"Error: Directory {directory} does not exist", file=sys.stderr)
        return 1

    if not target.is_dir():
        print(f"Error: {directory} is not a directory", file=sys.stderr)
        return 1

    formatters = [
        {
            "name": "ruff-format",
            "cmd": ["uv", "run", "ruff", "format"] + (["--check"] if check_only else []) + ["."],
            "description": "Formatting with Ruff"
        },
        {
            "name": "ruff-lint-fix",
            "cmd": ["uv", "run", "ruff", "check"] + ([] if check_only else ["--fix"]) + ["."],
            "description": "Linting with Ruff" + (" (fix mode)" if not check_only else " (check mode)")
        },
    ]

    all_passed = True

    for formatter in formatters:
        if verbose:
            print(f"\n>>> {formatter['description']} in {target}")

        returncode, stdout, stderr = run_command(formatter["cmd"], target)

        if returncode != 0:
            all_passed = False
            print(f"FAILED: {formatter['name']} in {directory}", file=sys.stderr)
            if stdout:
                print(stdout, file=sys.stderr)
            if stderr:
                print(stderr, file=sys.stderr)
        elif verbose:
            print(f"OK: {formatter['name']}")
            if stdout:
                print(stdout)

    return 0 if all_passed else 1


def main() -> int:
    parser = argparse.ArgumentParser(
        description="Apply company formatters to a directory"
    )
    parser.add_argument(
        "directory",
        help="Directory to format"
    )
    parser.add_argument(
        "--check",
        action="store_true",
        help="Only check formatting, don't apply changes"
    )
    parser.add_argument(
        "--verbose", "-v",
        action="store_true",
        help="Verbose output"
    )

    args = parser.parse_args()
    return format_directory(args.directory, args.check, args.verbose)


if __name__ == "__main__":
    sys.exit(main())
```

### שילוב הפורמטר ב-NX

```json
// nx.json targetDefaults - הוספת format target
{
  "targetDefaults": {
    "format-check": {
      "executor": "nx:run-commands",
      "options": {
        "command": "python {workspaceRoot}/tools/formatter/format.py {projectRoot} --check",
        "cwd": "{workspaceRoot}"
      },
      "cache": true,
      "inputs": [
        "{projectRoot}/**/*.py",
        "{workspaceRoot}/tools/formatter/format.py",
        "{workspaceRoot}/pyproject.toml"
      ]
    },
    "format": {
      "executor": "nx:run-commands",
      "options": {
        "command": "python {workspaceRoot}/tools/formatter/format.py {projectRoot} --verbose",
        "cwd": "{workspaceRoot}"
      },
      "cache": false
    }
  }
}
```

```bash
# הרצה על פרויקט ספציפי
nx format my-python-lib

# בדיקה בלבד (CI)
nx format-check my-python-lib

# על כל הפרויקטים שהשתנו
nx affected -t format-check

# format הכל
nx run-many -t format
```

---

## תוכנית מעבר שלב-אחר-שלב

### Phase 1: NX (שבועות 1-3)

#### שבוע 1: הכנה וניסוי

**Tasks:**
- [ ] בחרו 2-3 ספריות Python לניסוי (לא קריטיות ל-production)
- [ ] התקינו NX בשורש המונורפו
- [ ] צרו `nx.json` ו-`project.json` לספריות הניסוי
- [ ] הריצו `nx graph` וודאו שהתלויות נכונות
- [ ] הריצו `nx test <lib>` וודאו שהכל עובד

```bash
# שלב הניסוי
npm init -y
npm install --save-dev nx
echo '{"defaultBase": "main"}' > nx.json

# לכל ספרייה בניסוי
cat > packages/core-lib/project.json << 'EOF'
{
  "name": "core-lib",
  "root": "packages/core-lib",
  "projectType": "library",
  "targets": {
    "test": {
      "executor": "nx:run-commands",
      "options": {
        "command": "poetry run pytest",
        "cwd": "{projectRoot}"
      }
    }
  }
}
EOF

npx nx test core-lib
npx nx graph
```

**בעיות צפויות בשבוע 1:**
- NX לא מזהה תלויות → הוסיפו `implicitDependencies`
- Cache לא עובד → בדקו `inputs` definitions
- פקודה נכשלת → בדקו `cwd` ו-`env`

#### שבוע 2: הרחבה לכל הפרויקטים

**Tasks:**
- [ ] צרו `project.json` לכל ספריות ה-Python
- [ ] צרו `project.json` לכל אפליקציות ה-Vue.js
- [ ] הגדירו `namedInputs` גלובליים ב-`nx.json`
- [ ] הגדירו `targetDefaults` גלובליים
- [ ] הריצו `nx graph` ווידאו שהגרף שלם ונכון

**Scripts לאוטומציה:**

```python
# tools/scripts/generate-project-json.py
"""
Script to auto-generate project.json files for all Python packages
"""
import json
from pathlib import Path
import toml

WORKSPACE_ROOT = Path(__file__).parent.parent.parent

def generate_project_json(package_dir: Path) -> dict:
    pyproject_path = package_dir / "pyproject.toml"
    if not pyproject_path.exists():
        return None

    pyproject = toml.loads(pyproject_path.read_text())
    name = pyproject.get("project", {}).get("name", package_dir.name)

    return {
        "name": name,
        "root": str(package_dir.relative_to(WORKSPACE_ROOT)),
        "sourceRoot": str((package_dir / "src").relative_to(WORKSPACE_ROOT)),
        "projectType": "library",
        "tags": ["lang:python"],
        "targets": {
            "build": {
                "executor": "nx:run-commands",
                "options": {
                    "command": "uv build --no-sources",
                    "cwd": "{projectRoot}"
                },
                "cache": True,
                "outputs": ["{projectRoot}/dist"]
            },
            "test": {
                "executor": "nx:run-commands",
                "options": {
                    "command": "uv run pytest tests/ -v",
                    "cwd": "{projectRoot}"
                },
                "cache": True
            },
            "lint": {
                "executor": "nx:run-commands",
                "options": {
                    "command": "uv run ruff check .",
                    "cwd": "{projectRoot}"
                },
                "cache": True
            }
        }
    }

for package_dir in (WORKSPACE_ROOT / "packages").iterdir():
    if package_dir.is_dir():
        config = generate_project_json(package_dir)
        if config:
            project_json_path = package_dir / "project.json"
            if not project_json_path.exists():  # לא מדרס על קיים
                project_json_path.write_text(json.dumps(config, indent=2))
                print(f"Created: {project_json_path}")
```

#### שבוע 3: CI/CD

**Tasks:**
- [ ] עדכנו `.gitlab-ci.yml` לשימוש ב-`nx affected`
- [ ] הגדירו `NX_BASE` ו-`NX_HEAD` נכון
- [ ] הגדירו cache ב-GitLab runners
- [ ] בדקו שה-pipeline מריץ רק את מה שצריך

```yaml
# .gitlab-ci.yml
image: node:20-slim

variables:
  CI: "true"
  NX_HEAD: "$CI_COMMIT_SHA"
  NX_BASE: "${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}"

before_script:
  - apt-get update -qq && apt-get install -y -qq python3 curl git
  - curl -LsSf https://astral.sh/uv/install.sh | sh
  - source $HOME/.cargo/env
  - npm ci --silent
  # עדיין עם Poetry בשלב זה:
  - pip install -q poetry
  - poetry install --all-extras

cache:
  key: "$CI_COMMIT_REF_SLUG"
  paths:
    - .nx/cache/
    - node_modules/

test:
  script:
    - npx nx affected -t test --parallel=4 --nxBail
  only:
    - main
    - merge_requests
```

### Phase 2: UV (שבועות 4-6)

#### שבוע 4: Pilot Migration

**Tasks:**
- [ ] בחרו 3-5 ספריות לניסוי
- [ ] המירו `pyproject.toml` שלהן לפורמט UV
- [ ] צרו UV workspace ב-root
- [ ] בדקו שכל הבדיקות עוברות
- [ ] תעדו בעיות שנתקלתם

```bash
# Migration script per-library
cd packages/core-lib

# גיבוי
cp pyproject.toml pyproject.toml.poetry.bak

# המרה ידנית (ראו סעיף "המרת pyproject.toml" למעלה)
# ...

# בדיקה
uv sync
uv run pytest
```

#### שבוע 5: Full Migration

**Tasks:**
- [ ] העבירו את כל הספריות
- [ ] צרו `uv.lock` מרכזי
- [ ] עדכנו CI/CD ל-UV
- [ ] מחקו `poetry.lock` ו-`poetry.toml` files

```bash
# בשורש workspace
uv lock
git add uv.lock
git rm poetry.lock
git rm -r poetry-related-files/
```

#### שבוע 6: Validation

**Tasks:**
- [ ] ריצו את כל הבדיקות
- [ ] בדקו pipelines ב-CI
- [ ] מדדו שיפור בזמן installation
- [ ] תעדו את השינויים

### Phase 3: Ruff (שבוע 7)

**Tasks:**
- [ ] הוסיפו `ruff` ל-dev dependencies
- [ ] הגדירו `[tool.ruff]` ב-`pyproject.toml` המרכזי
- [ ] הריצו `ruff check . --fix` על הכל
- [ ] עברו על כל ה-warnings ותקנו
- [ ] הוסיפו `lint` target ל-NX לכל הפרויקטים
- [ ] הוסיפו lint ל-CI pipeline

```bash
# הרצה ראשונה - רק בדיקה
uv run ruff check . --statistics  # ראו כמה issues

# תיקון אוטומטי
uv run ruff check . --fix

# בדיקה מה נשאר
uv run ruff check .
```

### Phase 4: Formatter (שבוע 8)

**Tasks:**
- [ ] כתבו/הגדירו את סקריפט הפורמטר
- [ ] שלבו ב-NX כ-target
- [ ] הוסיפו ל-CI כ-check
- [ ] אמנו את הצוות

```bash
# בדיקת הפורמטר
python tools/formatter/format.py packages/core-lib --check --verbose
python tools/formatter/format.py packages/ --verbose  # כל הפרויקטים

# ב-NX
nx affected -t format-check
```

---

## בעיות צפויות ופתרונות

### בעיות NX

#### בעיה: `nx` command not found

```bash
# פתרון: הריצו דרך npx
npx nx <command>

# או התקינו גלובלית
npm install -g nx
```

#### בעיה: NX Version Mismatch

```
Error: Your global Nx CLI version X.Y.Z is greater than the local version Z.Y.X
```

```bash
# עדכנו NX בפרויקט
npx nx migrate latest
npx nx migrate --run-migrations
```

#### בעיה: Cache לא עובד בCI

**סימפטום:** כל run ב-CI מריץ הכל מחדש.

**פתרון:**
```yaml
# .gitlab-ci.yml
cache:
  key: "nx-$CI_COMMIT_REF_SLUG"
  paths:
    - .nx/cache/
  # חשוב: policy: pull-push
```

#### בעיה: `nx graph` לא מציג תלויות Python

**פתרון:**
1. הוסיפו `implicitDependencies` ב-`project.json`
2. ווידאו ש-`[tool.uv.sources]` מוגדר עם `workspace = true`

### בעיות UV

#### בעיה: UV לא מוצא Python

```bash
# התקנת Python דרך UV
uv python install 3.12
uv python pin 3.12  # בתיקיית הפרויקט
```

#### בעיה: Resolution conflicts בין workspace members

**סימפטום:**
```
error: package A requires B>=2.0, but C requires B<2.0
```

**פתרון:**
```toml
# pyproject.toml שורש - constraint מפורש
[tool.uv]
constraint-dependencies = [
    "package-B==2.1.0"  # pin גרסה ספציפית
]
```

#### בעיה: Private Registry לא נגיש

**פתרון:**
```bash
# ווידאו שה-URL נגיש
curl -v https://pypi.internal.company.com/simple

# ווידאו שה-credentials נכונים
uv pip install --index-url https://user:pass@pypi.internal.company.com/simple test-package

# ב-.env
UV_INDEX_INTERNAL_USERNAME=user
UV_INDEX_INTERNAL_PASSWORD=token
```

#### בעיה: `uv lock` לוקח הרבה זמן

**פתרון:**
```bash
# הפעילו cache
export UV_CACHE_DIR=/path/to/shared/cache

# ב-CI - cache את UV
cache:
  paths:
    - .uv-cache/
variables:
  UV_CACHE_DIR: ".uv-cache"
```

### בעיות Ruff

#### בעיה: Ruff conflicts עם Black/isort קיימים

**פתרון:**
```bash
# הסירו Black ו-isort מה-dependencies
uv remove black isort

# ב-pyproject.toml - הגדירו ruff כ-formatter
[tool.ruff.format]
# רוב ה-settings תואמים Black כברירת מחדל
```

#### בעיה: יותר מדי false positives

**פתרון:**
```toml
[tool.ruff.lint]
ignore = [
    "ANN101",  # missing type annotation for self
    "D100",    # missing docstring in public module
]
```

#### בעיה: Ruff שובר legacy code

**פתרון:** השתמשו ב-`noqa` comments זמנית:
```python
import old_module  # noqa: F401

def legacy_function(self, x, y, z=None, **kwargs):  # noqa: ANN
    pass
```

ואז עבדו לנקות בהדרגה.

### בעיות GitLab CI

#### בעיה: `NX_BASE` שגוי ב-pipeline

**סימפטום:** `nx affected` מריץ הכל.

**פתרון:**
```yaml
variables:
  NX_HEAD: "$CI_COMMIT_SHA"
  NX_BASE: "${CI_MERGE_REQUEST_DIFF_BASE_SHA:-$CI_COMMIT_BEFORE_SHA}"

# Debug - הדפיסו את הגרסאות
script:
  - echo "NX_BASE: $NX_BASE"
  - echo "NX_HEAD: $NX_HEAD"
  - git log --oneline $NX_BASE..$NX_HEAD
  - npx nx affected --print-affected -t build
```

#### בעיה: UV installation איטית ב-CI

**פתרון:**
```dockerfile
# השתמשו ב-Docker image עם UV מותקן מראש
FROM ghcr.io/astral-sh/uv:0.6-python3.12-slim
```

#### בעיה: Release גם ב-MR pipelines

**פתרון:** שמרו release רק ל-main:
```yaml
release:
  only:
    - main
  when: manual  # ידני
```

---

## נספח A: Cheat Sheet - פקודות שימושיות

### NX

```bash
# מידע
npx nx show projects                    # רשימת פרויקטים
npx nx show project my-lib --web        # פרטי פרויקט
npx nx graph                            # גרף תלויות
npx nx list                             # plugins זמינים

# הרצה
nx test my-lib                          # test ספציפי
nx run-many -t test                     # test לכולם
nx affected -t test                     # test למושפעים
nx affected -t test --parallel=5        # עם 5 workers
nx affected -t lint,test,build          # מספר targets

# Release
nx release --dry-run                    # preview
nx release --projects=my-lib --dry-run  # ספרייה ספציפית
nx release --groups=python-libs         # קבוצה

# Maintenance
nx reset                                # מחק cache
nx migrate latest                       # עדכן NX
```

### UV

```bash
# ניהול dependencies
uv add requests                         # הוסף תלות
uv add --dev pytest ruff                # תלות פיתוח
uv remove requests                      # הסר תלות
uv sync                                 # sync environment
uv lock                                 # עדכן lockfile
uv lock --upgrade                       # עדכן כל התלויות
uv lock --upgrade-package requests      # עדכן ספרייה ספציפית

# הרצה
uv run pytest                           # הרץ בvenv
uv run python script.py                 # הרץ script
uv run --package my-lib pytest          # הרץ ב-package ספציפי

# Build & Publish
uv build                                # בנה
uv build --no-sources                   # בנה ללא workspace sources
uv publish --index internal             # פרסם

# Python versions
uv python install 3.12                  # התקן Python
uv python list                          # גרסאות זמינות
uv python pin 3.12                      # pin גרסה
```

### Ruff

```bash
# Linting
ruff check .                            # בדוק
ruff check . --fix                      # בדוק ותקן
ruff check . --statistics               # סטטיסטיקה
ruff check . --select E,F               # כללים ספציפיים

# Formatting
ruff format .                           # פרמט
ruff format --check .                   # בדוק format
ruff format --diff .                    # הצג שינויים

# מידע
ruff linter                             # כל הכללים
ruff rule E501                          # הסבר כלל ספציפי
```

---

## נספח B: מבנה Workspace מלא לדוגמה

```
company-monorepo/
├── .gitlab-ci.yml
├── .gitignore
├── .nxignore
├── .python-version          # "3.12"
├── .ruff.toml               # Ruff config (או ב-pyproject.toml)
├── nx.json                  # NX workspace config
├── package.json             # NX + Node.js deps
├── package-lock.json
├── pyproject.toml           # UV workspace root
├── uv.lock                  # UV lockfile
├── tools/
│   ├── formatter/
│   │   └── format.py        # Custom formatter
│   └── scripts/
│       ├── release.ts       # Programmatic release
│       └── generate-project-json.py
├── packages/                # Python libraries
│   ├── core-lib/
│   │   ├── project.json
│   │   ├── pyproject.toml
│   │   ├── README.md
│   │   ├── CHANGELOG.md
│   │   ├── src/
│   │   │   └── core_lib/
│   │   │       ├── __init__.py
│   │   │       └── ...
│   │   └── tests/
│   ├── data-utils/
│   │   ├── project.json
│   │   ├── pyproject.toml
│   │   └── ...
│   └── api-client/
│       └── ...
├── apps/                    # Applications
│   ├── backend-api/         # Python service
│   │   ├── project.json
│   │   ├── pyproject.toml
│   │   └── src/
│   └── frontend-dashboard/  # Vue.js app
│       ├── project.json
│       ├── package.json
│       ├── vite.config.ts
│       ├── tsconfig.json
│       └── src/
└── libs/                    # Shared Vue/TS libs
    └── ui-components/
        ├── project.json
        ├── package.json
        └── src/
```

---

## נספח C: Security Considerations לסביבה סגורה

### אימות ב-Registry

```bash
# הגדירו ב-GitLab CI Variables (Masked):
# UV_INDEX_INTERNAL_USERNAME = __token__
# UV_INDEX_INTERNAL_PASSWORD = <deploy_token>

# ב-pipeline
script:
  - export UV_INDEX_INTERNAL_PASSWORD="$INTERNAL_PYPI_TOKEN"
  - uv sync
```

### ביטול גישה ל-PyPI הציבורי

```toml
# pyproject.toml
[[tool.uv.index]]
name = "internal"
url = "https://pypi.internal.company.com/simple"
default = true

[tool.uv]
no-binary = false
```

```bash
# ב-CI - וודאו שאין גישה חיצונית
export UV_NO_INDEX=false  # false אבל עם internal default
# או
export UV_INDEX_URL="https://pypi.internal.company.com/simple"
```

### NX Cloud בסביבה סגורה

אם אתם לא יכולים להשתמש ב-NX Cloud:

```yaml
# השתמשו ב-GitLab cache במקום
cache:
  key: "nx-$CI_COMMIT_REF_SLUG"
  paths:
    - .nx/cache/
  policy: pull-push
```

---

## סיכום - מפת הדרכים

```
שבוע 1-2: NX Pilot
  ✓ התקנת NX
  ✓ project.json ל-2-3 ספריות
  ✓ הרצת tests דרך NX
  ✓ אימות cache עובד

שבוע 3: NX CI/CD
  ✓ עדכון .gitlab-ci.yml
  ✓ nx affected ב-pipeline
  ✓ cache ב-runners

שבוע 4: NX Full Rollout
  ✓ project.json לכל הפרויקטים
  ✓ dependency graph מלא
  ✓ nx release הגדרה

שבוע 5-6: UV Migration
  ✓ pilot: 3-5 ספריות
  ✓ full migration
  ✓ uv.lock מרכזי

שבוע 7: Ruff
  ✓ התקנה והגדרה
  ✓ תיקון קוד קיים
  ✓ CI integration

שבוע 8: Formatter
  ✓ סקריפט פורמטר
  ✓ NX target
  ✓ CI check
```

**הסוף:** workspace מנוהל, מהיר, עם release סלקטיבי מדויק וכלי quality ברמה גבוהה.

---

*מסמך זה נוצר עבור צוות הנדסה של ארגון גדול. יש לעדכנו בהתאם לגרסאות הכלים הספציפיות בסביבתכם.*
