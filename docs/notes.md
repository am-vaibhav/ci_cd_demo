CI/CD with GitHub Actions
=========================

This project demonstrates Continuous Integration (CI) — automatically checking code quality and running tests every time you push to GitHub. No manual testing needed.

## What is CI/CD?

```text
CI (Continuous Integration)              CD (Continuous Delivery/Deployment)
───────────────────────────              ──────────────────────────────────
Push code → auto lint → auto test        Auto deploy to staging/production
"Does my code work?"                     "Ship it to users"
```

This project covers **CI only** — automated linting and testing on every push to `main`.

## Why CI matters for ML

Without CI:
- You push broken code → teammates pull it → their environment breaks
- A teammate changes `app.py` but doesn't run tests → bug reaches production

With CI:
- Every push triggers lint + test automatically
- Broken code is caught immediately → GitHub shows a red X
- No one merges code that doesn't pass

---

## 1. The Application

**File:** `app.py`

```python
def add(a, b):
    return a + b

def subtract(a, b):
    return a - b
```

Simple functions used to demonstrate testing. In a real ML project, these would be data processing functions, feature engineering helpers, or model inference endpoints.

---

## 2. Unit Tests

**File:** `test_app.py`

```python
import unittest
from app import add, subtract

class TestMathFunctions(unittest.TestCase):
    def test_add(self):
        self.assertEqual(add(4, 5), 9)
        self.assertEqual(add(-1, 1), 0)

    def test_sub(self):
        self.assertEqual(subtract(4, 5), -1)
        self.assertEqual(subtract(-1, -1), 0)
```

What this means:
- `unittest` is Python's built-in testing framework (no install needed)
- Each `test_*` method is one test case
- `assertEqual(actual, expected)` fails the test if values don't match
- `python -m unittest discover` auto-finds all files matching `test_*.py` and runs them

Run locally before pushing:
```bash
python -m unittest discover
```

```text
..                           ← two dots = two tests passed
----------------------------------------------------------------------
Ran 2 tests in 0.001s
OK
```

---

## 3. GitHub Actions Workflow — Explained Line by Line

**File:** `.github/workflows/ci.yaml`

### Trigger — When does CI run?

```yaml
on:
  push:
    branches:
      - main
```

What this means:
- CI runs only on pushes to the `main` branch
- Pushes to other branches (feature branches, dev) don't trigger it
- To also trigger on pull requests, add `pull_request:` under `on:`

### Job 1: Linting

```yaml
linting:
  runs-on: ${{ matrix.os }}
  strategy:
    matrix:
      os: [ubuntu-latest, macos-latest]
      python-versions: ['3.10', '3.11', '3.12']
```

What this means:
- `runs-on` sets which operating system the job runs on
- `strategy.matrix` creates **all combinations** of OS and Python version
- 2 OS x 3 Python versions = **6 parallel jobs**

```text
Matrix expands to 6 runners:

ubuntu  + Python 3.10  ──→  Run flake8
ubuntu  + Python 3.11  ──→  Run flake8
ubuntu  + Python 3.12  ──→  Run flake8
macos   + Python 3.10  ──→  Run flake8
macos   + Python 3.11  ──→  Run flake8
macos   + Python 3.12  ──→  Run flake8
```

Why test on multiple OS/versions:
- Code that works on your machine might break on a different Python version (e.g., f-strings changed in 3.12)
- Cross-OS testing catches platform-specific bugs (file paths, line endings)

#### Steps inside the linting job:

```yaml
steps:
  - name: Code checkout
    uses: actions/checkout@v4

  - name: Setup Python
    uses: actions/setup-python@v5
    with:
      python-version: ${{ matrix.python-versions }}

  - name: Install Flake8
    run: |
      python -m pip install --upgrade pip
      pip install flake8

  - name: Run Flake8
    run: |
      flake8 app.py
```

What each step does:
1. **Code checkout** — clones your repo into the GitHub runner. Without this, the runner has an empty workspace
2. **Setup Python** — installs the Python version from the matrix. Runners come with Python pre-installed but not necessarily the version you need
3. **Install Flake8** — installs the linting tool. Each runner starts clean — no packages carry over between runs
4. **Run Flake8** — checks `app.py` for style violations, syntax errors, and unused imports

### Job 2: Testing

```yaml
testing:
  needs: linting
  runs-on: ubuntu-latest
  steps:
    - name: Code checkout
      uses: actions/checkout@v4

    - name: Setup Python
      uses: actions/setup-python@v5
      with:
        python-version: '3.11'

    - name: Run unit test
      run: |
        python -m unittest discover
```

What this means:
- `needs: linting` — this job **waits** for all 6 linting jobs to pass before running
- If any linting job fails, testing is **skipped entirely**
- Only runs on `ubuntu-latest` with Python 3.11 (no matrix — one environment is enough for unit tests)
- `unittest discover` auto-finds `test_*.py` files and runs all `test_*` methods

---

## 4. Sequential vs Parallel Jobs

```yaml
# The key line:
needs: linting
```

```text
WITH `needs: linting`:              WITHOUT `needs`:

linting (6 runners)                 linting (6 runners) ──┐
    │                               testing ──────────────┘
    │ all must pass                  (both run at the same time)
    ▼
testing (1 runner)
```

What this means:
- `needs` creates a dependency — "don't start until X finishes"
- Without `needs`, GitHub runs all jobs in parallel by default
- You want sequential when testing depends on linting — no point running tests if the code has syntax errors
- You want parallel when jobs are independent — e.g., linting Python + linting JavaScript

---

## 5. Flake8 — What It Checks

Flake8 is a linting tool that checks Python code for common issues:

| Category | What It Catches | Example |
|----------|-----------------|---------|
| **Style (PEP 8)** | Wrong indentation, missing whitespace, long lines | `x=1` → `x = 1` |
| **Syntax errors** | Invalid Python that would crash at runtime | `def foo(` missing `)` |
| **Unused imports** | Importing modules you never use | `import os` but never using `os` |
| **Undefined names** | Using variables that don't exist | Typo: `pritn(x)` instead of `print(x)` |
| **Complexity** | Too many nested if/else blocks | Deeply nested function |

Run locally:
```bash
pip install flake8
flake8 app.py           # check one file
flake8 .                # check all Python files in current directory
flake8 --max-line-length=120 app.py   # allow longer lines
```

---

## 6. GitHub Actions — Key Concepts

### Where to put workflow files

```text
.github/
└── workflows/
    └── ci.yaml       ← GitHub auto-detects any .yml/.yaml here
```

What this means:
- GitHub scans `.github/workflows/` for YAML files
- Each file is a separate workflow — you can have `ci.yaml`, `deploy.yaml`, `test.yaml` etc.
- GitHub runs them automatically based on their `on:` triggers

### Common Actions (pre-built steps)

| Action | What It Does |
|--------|-------------|
| `actions/checkout@v4` | Clones your repo into the runner |
| `actions/setup-python@v5` | Installs a specific Python version |
| `actions/setup-node@v4` | Installs Node.js (for JS projects) |
| `actions/cache@v4` | Caches dependencies between runs (faster CI) |

What this means:
- `uses:` references a pre-built action from GitHub's marketplace
- `@v4` is the version — pin it so your CI doesn't break when the action updates
- `with:` passes configuration to the action

### Matrix Strategy — Explained

```yaml
strategy:
  matrix:
    os: [ubuntu-latest, macos-latest]
    python-versions: ['3.10', '3.11', '3.12']
```

What this means:
- Creates the cartesian product (all combinations) of the values
- Each combination runs as a separate, parallel job
- If one combination fails, the others continue running (unless you set `fail-fast: true`)
- Reference values in the job with `${{ matrix.os }}` and `${{ matrix.python-versions }}`

---

## 7. Full Workflow

```text
Developer pushes to main
        │
        ▼
GitHub Actions triggered (.github/workflows/ci.yaml)
        │
        ▼
┌─────────────────────────────────────────────┐
│ Linting (6 parallel jobs)                   │
│                                             │
│ ubuntu + 3.10 ──→ flake8 app.py ──→ pass/fail│
│ ubuntu + 3.11 ──→ flake8 app.py ──→ pass/fail│
│ ubuntu + 3.12 ──→ flake8 app.py ──→ pass/fail│
│ macos  + 3.10 ──→ flake8 app.py ──→ pass/fail│
│ macos  + 3.11 ──→ flake8 app.py ──→ pass/fail│
│ macos  + 3.12 ──→ flake8 app.py ──→ pass/fail│
└──────────────────────┬──────────────────────┘
                       │ all 6 must pass
                       ▼
┌─────────────────────────────────────────────┐
│ Testing (1 job)                             │
│                                             │
│ ubuntu + 3.11 ──→ unittest discover ──→ pass/fail│
└──────────────────────┬──────────────────────┘
                       │
                       ▼
              Green check ✓ or Red X ✗ on GitHub
```

## 8. Common Gotchas

| Problem | Cause | Fix |
|---------|-------|-----|
| CI passes locally but fails on GitHub | Different Python version or OS | Add the failing version to your matrix |
| `ModuleNotFoundError` in CI | Missing `pip install` step | Add dependencies to the install step |
| Flake8 fails on line length | Lines > 79 chars (PEP 8 default) | Add `--max-line-length=120` or fix the lines |
| Tests not found | File not named `test_*.py` | `unittest discover` only finds `test_*.py` |
| Job runs but shouldn't | Missing `needs:` dependency | Add `needs: <job-name>` to enforce order |
