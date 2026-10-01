# Lesson 8.3 — Dependency Management with Poetry

## CONCEPT

**Poetry** is a modern Python tool that combines what Module 0 covered separately with `venv` and `pip freeze` into one integrated workflow: it manages your virtual environment, tracks your project's dependencies with exact, reproducible versions (a **lockfile**), and packages your project — all through one configuration file, `pyproject.toml`.

```toml
# pyproject.toml
[tool.poetry]
name = "ecommerce-pipeline"
version = "0.1.0"

[tool.poetry.dependencies]
python = "^3.12"
pandas = "^2.2"
sqlalchemy = "^2.0"
```

```bash
poetry install        # creates a virtual environment AND installs exact dependency versions
poetry add requests     # adds a new dependency, updating both pyproject.toml and the lockfile
poetry run python main.py   # runs a command inside the project's managed environment
```

## WHY IT MATTERS

Module 0's `venv` + `pip freeze > requirements.txt` workflow works, but has a real gap: `requirements.txt` only records what you happened to have installed at the moment you ran `pip freeze` — it's easy to forget to regenerate it, and it doesn't distinguish "a package I directly need" from "a package that got installed as someone else's dependency." Poetry's lockfile (`poetry.lock`) captures the *exact* dependency tree automatically and precisely, which is exactly the reproducibility guarantee a production pipeline needs: the same `poetry install` produces an identical environment on your laptop, in CI, and in a deployed container, every time.

## SYNTAX REFERENCE

```bash
# Installing Poetry itself (a one-time setup, separate from any single project)
curl -sSL https://install.python-poetry.org | python3 -
poetry --version           # confirm it installed correctly

# Starting a new project
poetry new ecommerce-pipeline      # creates a new folder with a standard structure
cd ecommerce-pipeline

# OR: adding Poetry to an existing project folder
cd existing-project
poetry init                          # interactively creates a pyproject.toml

# Adding dependencies
poetry add pandas                     # adds a regular dependency
poetry add pytest --group dev          # adds a DEV-only dependency (testing tools, not needed in production)
poetry remove pandas                    # removes a dependency

# Installing from an existing pyproject.toml + poetry.lock (e.g. after cloning a repo)
poetry install

# Running commands inside the project's managed environment
poetry run python main.py
poetry run pytest

# Activating a shell inside the environment (alternative to prefixing every command with `poetry run`)
poetry shell

# Viewing the dependency tree
poetry show --tree

# Updating dependencies to their latest allowed versions (respecting version constraints)
poetry update
```

```toml
# A typical pyproject.toml structure
[tool.poetry]
name = "ecommerce-pipeline"
version = "0.1.0"
description = "ETL pipeline for e-commerce order data"

[tool.poetry.dependencies]
python = "^3.12"
pandas = "^2.2"
sqlalchemy = "^2.0"
psycopg2-binary = "^2.9"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0"
mypy = "^1.10"
ruff = "^0.4"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

## WORKED EXAMPLE

Setting up a new project with Poetry, from nothing to a working, reproducible environment:

```bash
# 1. Create a new project (Poetry sets up the folder structure automatically)
poetry new ecommerce-pipeline
cd ecommerce-pipeline

# Poetry created:
# ecommerce-pipeline/
# ├── pyproject.toml
# ├── README.md
# ├── ecommerce_pipeline/
# │   └── __init__.py
# └── tests/
#     └── __init__.py

# 2. Add the dependencies this project actually needs
poetry add pandas sqlalchemy psycopg2-binary python-dotenv

# 3. Add dev-only tools — needed while developing, NOT needed to actually run the pipeline
poetry add pytest mypy ruff --group dev

# 4. Poetry automatically created a virtual environment and a poetry.lock file
#    recording the EXACT versions of everything, including indirect dependencies

# 5. Run code inside the managed environment
poetry run python -c "import pandas; print(pandas.__version__)"

# 6. Confirm the dependency tree
poetry show --tree
# ecommerce-pipeline
# ├── pandas 2.2.2
# │   ├── numpy >=1.26.0
# │   ├── python-dateutil >=2.8.2
# │   └── ...
# ├── sqlalchemy 2.0.30
# └── ...

# 7. Commit BOTH pyproject.toml AND poetry.lock to version control
#    (unlike requirements.txt, which only shows top-level packages by convention,
#    poetry.lock pins the ENTIRE dependency tree exactly)

# --- Simulating a teammate cloning this repo later ---
# They run just ONE command to get an identical environment:
poetry install
# This reads poetry.lock (if present) and installs EXACTLY those versions —
# not just "pandas 2.x, whatever the latest 2.x happens to be today"
```

## COMMON MISTAKES

**1. Forgetting to commit `poetry.lock` to version control**
```bash
git add pyproject.toml     # committed the requirements...
# forgot: git add poetry.lock   # but not the EXACT locked versions!
```
Fix: always commit both files together. `pyproject.toml` alone only specifies version *ranges* (like `^2.2`, meaning "2.2 or any compatible newer version") — without the lockfile, two different `poetry install` runs on different days could genuinely install different exact versions.

**2. Mixing `pip install` and `poetry add` in the same project**
```bash
poetry add pandas
pip install requests   # NOT tracked by Poetry at all — invisible to pyproject.toml/poetry.lock!
```
Fix: once a project uses Poetry, add every dependency through `poetry add` (or `poetry add ... --group dev` for dev tools) exclusively — mixing in raw `pip install` calls creates an untracked, unreproducible gap in your declared dependencies.

**3. Not distinguishing regular dependencies from dev dependencies**
```bash
poetry add pytest    # added as a REGULAR dependency — now pytest installs even in production!
```
Fix: testing and linting tools (`pytest`, `mypy`, `ruff`) belong in `--group dev` — they're needed while developing and in CI, but a production deployment running the actual pipeline doesn't need them, and keeping them separate keeps the production install smaller and more focused.

**4. Misunderstanding version constraint syntax**
```toml
pandas = "^2.2"    # means "2.2.0 or higher, but LESS than 3.0.0" — allows 2.3, 2.9, NOT 3.0
pandas = "~2.2"     # means "2.2.0 or higher, but less than 2.3.0" — a much narrower range
pandas = "2.2.0"     # means EXACTLY 2.2.0, nothing else — usually too restrictive
```
Fix: `^` (caret) is the most common and usually the right default — it allows compatible updates while protecting against breaking major-version changes; reserve exact pins for cases where you've specifically confirmed a newer version causes a problem.

**5. Running `python main.py` directly instead of `poetry run python main.py`**
```bash
python main.py   # uses whatever Python is on your system PATH — may NOT be the Poetry-managed one!
```
Fix: always prefix commands with `poetry run`, or use `poetry shell` to activate the managed environment for your whole terminal session — otherwise you might accidentally run code against a different (system, or another project's) Python environment entirely, silently using the wrong package versions.

**Data Engineer Tip:** The `pyproject.toml` + `poetry.lock` pair genuinely matters once a pipeline moves from "runs on my laptop" to "runs in CI, and gets deployed to a container." A Dockerfile that runs `poetry install` reproduces the *exact* dependency tree every time an image gets built — eliminating an entire category of "works on my machine, breaks in production" bugs caused by a subtle package version mismatch. This is precisely why Poetry (or an equivalent modern lockfile-based tool) is the standard expectation on most professional data engineering teams today, rather than a loose `requirements.txt`.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Install Poetry on your machine (if you haven't already) and confirm it's working with `poetry --version`.

<details>
<summary>Hint</summary>
`curl -sSL https://install.python-poetry.org | python3 -`, then restart your terminal if the command isn't found immediately.
</details>

<details>
<summary>Solution</summary>
No code — verification step. Expected output resembles:
```
Poetry (version 1.8.3)
```
(the exact version number will vary depending on when you install it).
</details>

---

**Exercise 2 (Easy):** Create a new Poetry project called `practice-project`, and inspect the folder structure it generates.

<details>
<summary>Hint</summary>
`poetry new practice-project`
</details>

<details>
<summary>Solution</summary>

```bash
poetry new practice-project
cd practice-project
ls -R
```
Expected structure:
```
practice-project/
├── pyproject.toml
├── README.md
├── practice_project/
│   └── __init__.py
└── tests/
    └── __init__.py
```
</details>

---

**Exercise 3 (Medium):** Inside `practice-project`, add `requests` as a regular dependency and `pytest` as a dev dependency. Then run `poetry show --tree` and confirm both appear, and inspect `pyproject.toml` to confirm they were added under the correct sections.

<details>
<summary>Hint</summary>
`poetry add requests` and `poetry add pytest --group dev`
</details>

<details>
<summary>Solution</summary>

```bash
poetry add requests
poetry add pytest --group dev
poetry show --tree
cat pyproject.toml
```
`pyproject.toml` should now show `requests` under `[tool.poetry.dependencies]` and `pytest` under `[tool.poetry.group.dev.dependencies]`.
</details>

---

**Exercise 4 (Medium):** Write a tiny script `hello.py` inside `practice-project/practice_project/` that imports `requests` and prints `requests.__version__`. Run it using `poetry run`, and confirm it executes correctly using the Poetry-managed environment (not a system-wide `requests` install, if you happen to have one).

<details>
<summary>Hint</summary>
`poetry run python practice_project/hello.py`
</details>

<details>
<summary>Solution</summary>

```python
# practice_project/hello.py
import requests
print(requests.__version__)
```
```bash
poetry run python practice_project/hello.py
```
</details>

---

**Exercise 5 (Hard):** Simulate the "teammate clones this repo" scenario from the worked example: delete the `.venv`/virtual environment Poetry created for `practice-project` (find it with `poetry env info --path`, or simply remove the project's cache via `poetry env remove python`), then run `poetry install` from scratch using only the existing `pyproject.toml` and `poetry.lock` files, and confirm `poetry run python practice_project/hello.py` still works identically afterward — demonstrating that the lockfile alone is sufficient to reproduce the environment.

<details>
<summary>Hint</summary>
`poetry env info --path` shows you where the environment lives; `poetry env remove python` removes it entirely (rather than manually deleting folders, which is safer and Poetry-aware). Then `poetry install` rebuilds it from the lockfile.
</details>

<details>
<summary>Solution</summary>

```bash
# Find and remove the existing environment
poetry env info --path
poetry env remove python

# Reinstall from scratch, using ONLY pyproject.toml + poetry.lock
poetry install

# Confirm it works identically
poetry run python practice_project/hello.py
```
This demonstrates the core value proposition: you (or a teammate, or a CI system, or a Docker build) never need to remember which exact packages were installed by hand — `poetry install` alone, run against the committed `poetry.lock`, reproduces the exact same environment every time.
</details>

---

**Previous lesson:** [8.2 — Structuring a Project: Modules & Packages](lesson-8.2-project-structure.md)
**Next lesson:** [8.4 — Robust Error Handling & Custom Exceptions](lesson-8.4-custom-exceptions.md)
