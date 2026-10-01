# Lesson 8.9 — Code Style, Linting & Pre-commit Hooks

## CONCEPT

A **linter** analyzes your code for style issues, likely bugs, and bad practices without running it — catching things like unused imports, undefined variables, and inconsistent formatting. A **formatter** automatically rewrites your code into a consistent style (spacing, quote style, line length) so you never have to think about it manually. **`ruff`** is a fast, modern linter (and increasingly a formatter too) that's become a standard choice in the Python ecosystem; **`black`** is the most widely used dedicated formatter. A **pre-commit hook** runs these tools automatically every time you attempt a Git commit, blocking the commit if something fails.

```bash
ruff check .          # lint the whole project, report issues
black .                 # auto-format the whole project
```

## WHY IT MATTERS

Inconsistent formatting and small, avoidable bugs (an unused import, a variable used before assignment, a bare `except:`) create noise that makes real code review harder and erodes trust in a codebase over time. Automating this — so style is simply never a discussion, and basic mistakes are caught before a human reviewer ever sees them — is standard practice on essentially every professional Python team. Setting this up now, on your own portfolio projects, is a direct, visible signal of production-readiness to anyone reviewing your code.

## SYNTAX REFERENCE

```bash
# Installing the tools (if not already installed in Module 8's setup)
pip install ruff black pre-commit

# Running ruff — a fast linter (checks for issues, doesn't fix formatting)
ruff check .                  # check the whole project
ruff check path/to/file.py      # check a single file
ruff check . --fix               # auto-fix issues ruff knows how to fix safely

# Running black — a formatter (rewrites code into a consistent style)
black .                        # format the whole project in place
black --check .                  # check WITHOUT modifying — useful in CI, exits non-zero if changes needed
black --diff .                    # show what WOULD change, without modifying

# Setting up pre-commit (one-time, per project)
# 1. Create a .pre-commit-config.yaml file in your project root:
```
```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
  - repo: https://github.com/psf/black
    rev: 24.8.0
    hooks:
      - id: black
```
```bash
# 2. Install the git hook (one-time, per project clone)
pre-commit install

# 3. From now on, every `git commit` automatically runs ruff + black first
git commit -m "add order cleaning function"
# -> ruff and black run automatically; if they modify files or find issues,
#    the commit is blocked until you review/re-stage the changes

# Running pre-commit manually against ALL files (useful the first time, on an existing project)
pre-commit run --all-files
```

## WORKED EXAMPLE

Taking a messy, inconsistently-styled file and cleaning it up with these tools, then wiring up pre-commit so it never regresses:

```python
# messy_pipeline.py — BEFORE cleanup
import pandas as pd
import numpy as np
import os  # unused import — ruff will flag this

def clean_orders(df):
    df['total']=df['total'].astype(float)
    df["status"] = df["status"].str.strip( ).str.lower()
    unused_variable = 42  # ruff flags this too — assigned but never used
    return df



def load_and_clean(  path ):
    df = pd.read_csv(path)
    return clean_orders(df)
```

Running the linter first, to see what it catches:

```bash
ruff check messy_pipeline.py
```
```
messy_pipeline.py:3:8: F401 'os' imported but unused
messy_pipeline.py:9:5: F841 local variable 'unused_variable' is assigned to but never used
Found 2 errors.
```

Fixing the real issues (ruff's `--fix` handles some automatically, but an unused variable assignment needs a human decision — delete it or use it), then running the formatter:

```bash
black messy_pipeline.py
```

```python
# messy_pipeline.py — AFTER black formatting (whitespace/quotes/spacing normalized)
import pandas as pd
import numpy as np


def clean_orders(df):
    df["total"] = df["total"].astype(float)
    df["status"] = df["status"].str.strip().str.lower()
    return df


def load_and_clean(path):
    df = pd.read_csv(path)
    return clean_orders(df)
```

Notice black fixed the inconsistent quote style, spacing around `=`, extra spaces inside `strip( )`, and extra blank lines — all without changing what the code actually does. The unused `os` import and `unused_variable` needed to be removed by hand, since removing them is a judgment call ruff flags but doesn't force.

Now wiring up pre-commit so this stays true going forward:

```bash
# Create .pre-commit-config.yaml (content shown in Syntax Reference above)
pre-commit install

# Test it against the whole project once, immediately
pre-commit run --all-files
```
```
ruff.....................................................................Passed
black....................................................................Passed
```

From this point on, every `git commit` in this project automatically re-runs both checks — a teammate (or future you) literally cannot commit inconsistently-formatted code or an obviously unused import without the commit being blocked first.

## COMMON MISTAKES

**1. Treating linter warnings as optional noise to ignore**
```
F841 local variable 'unused_variable' is assigned to but never used
```
This specific warning often points to a real bug — a value you meant to use (in a `return`, a log message, a later calculation) but forgot to. Fix: treat every linter warning as worth a moment's thought, even when the eventual fix is just deleting an unused line — it's one of the cheapest bug-finding tools available, precisely because it runs before you even execute the code.

**2. Running `black` without first committing or reviewing existing work**
`black` can rewrite a large number of lines across many files the first time it's run on an existing, previously-unformatted project — if you haven't committed your actual logic changes yet, it becomes hard to tell "real" changes apart from pure formatting changes in a diff. Fix: commit your functional changes first, *then* run `black` as a separate, dedicated "formatting" commit — keeps the git history readable.

**3. Forgetting `pre-commit install` is per-clone, not just per-project**
```bash
git clone https://github.com/you/your-pipeline
cd your-pipeline
git commit -m "fix bug"   # hooks DON'T run — pre-commit install was never run in this clone!
```
Fix: `pre-commit install` needs to be run once in *every* local clone of the repository (including a teammate's machine, or a fresh CI environment) — the `.pre-commit-config.yaml` file itself is what's shared via Git, but the actual Git hook has to be installed locally in each clone.

**4. Letting `ruff --fix` or `black` silently change code behavior**
In rare cases, an auto-fix (especially with more aggressive linter rules enabled) can change behavior, not just style — e.g., removing what looks like a genuinely unused import that was actually needed for a side effect. Fix: always review what an auto-fix changed (`git diff` after running `--fix`) before committing, rather than blindly trusting every automated change.

**5. Not pinning tool versions in `.pre-commit-config.yaml`**
Without a specific `rev:` pinned for each hook, different contributors (or different points in time) might run different versions of ruff/black, producing inconsistent results between runs. Fix: always pin a specific version (as shown in the Syntax Reference), and update it deliberately, the same way you'd update any other dependency.

**Data Engineer Tip:** CI/CD pipelines (platforms like GitHub Actions, Azure DevOps Pipelines) very commonly run `ruff check .` and `black --check .` as an automated step on every pull request — using `--check` (which fails loudly instead of silently reformatting) rather than letting CI modify your code directly. Setting up pre-commit locally, as in this lesson, means your code almost always already passes that CI check by the time you open a pull request, since the same checks already ran — and were already enforced — on your own machine before the commit even happened.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Install `ruff` and `black` in your project's virtual environment, and run `ruff check .` against any existing `.py` file from an earlier module. Report (in a comment or just by reading the output) how many issues, if any, it finds.

<details>
<summary>Hint</summary>
`pip install ruff black`, then `ruff check path/to/file.py`.
</details>

<details>
<summary>Solution</summary>
No single code solution — this is a verification/exploration step. Expected: `ruff check` either reports zero issues (clean file) or a short list of specific, named issues (e.g., unused imports, undefined names), each with a file/line reference.
</details>

---

**Exercise 2 (Easy):** Take this intentionally messy snippet, save it to a file, and run `black` on it. Compare the before/after.
```python
def   add(a,b):
    return a+b
x=add(1,2)
print( x )
```

<details>
<summary>Hint</summary>
`black messy.py`, then view the file again to see the formatting changes.
</details>

<details>
<summary>Solution</summary>

```bash
black messy.py
```
Resulting file:
```python
def add(a, b):
    return a + b


x = add(1, 2)
print(x)
```
`black` normalized the spacing around the function definition, the `+` operator, the assignment, and removed the unnecessary space inside `print( x )` — all without changing what the code does.
</details>

---

**Exercise 3 (Medium):** Create a `.pre-commit-config.yaml` file for a small project (content from the Syntax Reference is fine as a starting point), run `pre-commit install`, then make a trivial, intentionally badly-formatted change to a tracked Python file and attempt to commit it. Describe what happens.

<details>
<summary>Hint</summary>
After `pre-commit install`, a `git commit` with a badly formatted file should trigger ruff/black automatically, and either auto-fix and require you to re-stage, or block the commit until fixed.
</details>

<details>
<summary>Solution</summary>
No single code solution — this is a verification step. Expected behavior: running `git commit` after `pre-commit install` with a mis-formatted file triggers both hooks to run; `black` will likely auto-reformat the file and report "files were modified by this hook," which causes the commit to abort so you can review and re-`git add` the now-reformatted file before committing again successfully.
</details>

---

**Exercise 4 (Medium):** Given this snippet, intentionally containing an unused import and an unused variable, run `ruff check` on it and record the exact warnings produced, then fix both issues by hand and confirm `ruff check` passes cleanly afterward.
```python
import json
import sys

def process(data):
    result = data.upper()
    unused = 100
    return result
```

<details>
<summary>Hint</summary>
`ruff check file.py` should flag the unused `json` and `sys` imports (if `sys` genuinely isn't used elsewhere) and the unused `unused` variable.
</details>

<details>
<summary>Solution</summary>

```bash
ruff check file.py
```
Expected output (approximate):
```
file.py:1:8: F401 'json' imported but unused
file.py:2:8: F401 'sys' imported but unused
file.py:6:5: F841 local variable 'unused' is assigned to but never used
```
Fixed version:
```python
def process(data):
    result = data.upper()
    return result
```
Running `ruff check file.py` again now reports no issues.
</details>

---

**Exercise 5 (Hard):** Set up a complete pre-commit workflow for a small multi-file project: (1) create at least 2 Python files with a mix of real logic and a few intentional style issues (an unused import in one, inconsistent quote/spacing style in the other); (2) write a `.pre-commit-config.yaml` with both `ruff` (with `--fix`) and `black` configured; (3) run `pre-commit install`; (4) run `pre-commit run --all-files` once manually to clean up the existing issues; (5) make one additional small, deliberately messy edit to one file and attempt a real `git commit`, documenting (in comments, or just narratively) exactly what you observed happen at each step.

<details>
<summary>Hint</summary>
This exercise is meant to be run hands-on in a real project with Git initialized (`git init` if needed) — there's no single "solution code," but walking through all 5 steps end-to-end is the point: see the hooks actually catch and partially auto-fix real issues, and see a commit genuinely get blocked until the files are clean.
</details>

<details>
<summary>Solution</summary>

```bash
# 1. Set up a small project
mkdir precommit_demo && cd precommit_demo
git init

cat > module_a.py << 'EOF'
import os
import pandas as pd

def load(path):
    df = pd.read_csv(path)
    return df
EOF

cat > module_b.py << 'EOF'
def add(a,b):
    return a+b
x=add(1,2)
EOF

# 2. Create the pre-commit config
cat > .pre-commit-config.yaml << 'EOF'
repos:
  - repo: https://github.com/astral-sh/ruff-pre-commit
    rev: v0.6.9
    hooks:
      - id: ruff
        args: [--fix]
  - repo: https://github.com/psf/black
    rev: 24.8.0
    hooks:
      - id: black
EOF

# 3. Install the hook
pre-commit install

# 4. Clean up everything that already exists
pre-commit run --all-files
# Expected: ruff flags/fixes the unused `os` import in module_a.py;
# black reformats module_b.py's spacing; the run reports files were modified

git add .
git commit -m "initial commit, pre-commit configured"
# This commit should now succeed, since the files were already cleaned by
# the manual `--all-files` run above

# 5. Make one more deliberately messy edit and try to commit directly
echo 'y    =    [1,2,3]' >> module_b.py
git add module_b.py
git commit -m "add messy line"
# Expected: black reformats `y    =    [1,2,3]` to `y = [1, 2, 3]` automatically,
# the commit is BLOCKED on this first attempt ("files were modified by this hook"),
# and you need to `git add module_b.py` again and re-run `git commit` to actually
# commit the now-clean version — demonstrating the hook genuinely enforcing style
# before anything reaches the repository's history
```
</details>

---

## Module 8 Complete

You've now covered the full "production-grade" toolkit: classes, project structure, dependency management, robust error handling, logging, testing, type hints, configuration management, and automated code quality enforcement. Every one of these practices comes together in Module 9's capstone — a real, scheduled, tested ETL pipeline that applies all of it at once.

**Previous lesson:** [8.8 — Configuration Management](lesson-8.8-configuration-management.md)
**Next module:** [Module 9 — ETL Automation & Capstone](../module-9-etl-automation-and-capstone/README.md)
