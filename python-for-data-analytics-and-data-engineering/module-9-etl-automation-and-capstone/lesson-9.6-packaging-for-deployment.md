# Lesson 9.6 — Packaging the Pipeline for Deployment

## CONCEPT

**Packaging** means organizing a pipeline so it can be reliably installed, run, and deployed somewhere other than your own laptop — a teammate's machine, a server, a scheduled environment, or a container. This lesson ties together Module 8's project structure and Poetry lessons specifically for a deployable pipeline: a clean `pyproject.toml`, an entry-point script, environment-specific configuration, and — increasingly standard in real data engineering work — a **Docker container** that packages the code AND its exact runtime environment together.

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY pyproject.toml poetry.lock ./
RUN pip install poetry && poetry install --no-dev
COPY . .
CMD ["python", "-m", "ecommerce_pipeline.main"]
```

## WHY IT MATTERS

"It works on my machine" is the single most common failure mode when moving a pipeline from development into any kind of shared or scheduled environment — a missing dependency, a different Python version, an OS-specific path assumption. Packaging properly, and especially containerizing with Docker, eliminates this entire category of problem: a Docker container runs identically regardless of what's installed on the host machine, which is precisely why containers have become the standard deployment unit across nearly all modern data engineering platforms, including Azure Data Factory's custom activities and Azure Container Instances.

## SYNTAX REFERENCE

```toml
# pyproject.toml — Module 8's Poetry setup, with a proper entry point added
[tool.poetry]
name = "ecommerce-pipeline"
version = "1.0.0"
description = "Daily ETL pipeline for ecommerce order data"
authors = ["Your Name <you@example.com>"]

[tool.poetry.dependencies]
python = "^3.12"
pandas = "^2.2"
sqlalchemy = "^2.0"
psycopg2-binary = "^2.9"
python-dotenv = "^1.0"
pandera = "^0.20"
tenacity = "^9.0"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0"
mypy = "^1.11"
ruff = "^0.6"

[tool.poetry.scripts]
run-pipeline = "ecommerce_pipeline.main:main"   # creates a `run-pipeline` CLI command

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

```dockerfile
# Dockerfile — packages the code AND its exact environment together
FROM python:3.12-slim

WORKDIR /app

# Install dependencies first (better layer caching — rebuilds are faster
# when only your CODE changes, not your dependencies)
COPY pyproject.toml poetry.lock ./
RUN pip install --no-cache-dir poetry \
    && poetry config virtualenvs.create false \
    && poetry install --no-root --no-dev

# Now copy the actual application code
COPY ecommerce_pipeline/ ./ecommerce_pipeline/

# Run as a non-root user (a real security best practice, not just a formality)
RUN useradd --create-home appuser
USER appuser

CMD ["python", "-m", "ecommerce_pipeline.main"]
```

```bash
# Building and running the container
docker build -t ecommerce-pipeline:latest .
docker run --env-file .env ecommerce-pipeline:latest
# --env-file passes your .env variables into the container at runtime,
# WITHOUT ever baking secrets into the image itself (Module 7, Lesson 7.5)

# Tagging and pushing to a registry (e.g. Azure Container Registry), for
# deployment to a scheduled cloud environment
docker tag ecommerce-pipeline:latest myregistry.azurecr.io/ecommerce-pipeline:latest
docker push myregistry.azurecr.io/ecommerce-pipeline:latest
```

## WORKED EXAMPLE

Taking the capstone pipeline's project structure (from Module 8's conventions) and packaging it fully for deployment:

```
ecommerce_pipeline/
├── pyproject.toml
├── poetry.lock
├── Dockerfile
├── .dockerignore
├── .env.example
├── .gitignore
├── README.md
├── ecommerce_pipeline/
│   ├── __init__.py
│   ├── main.py
│   ├── extract.py
│   ├── transform.py
│   ├── load.py
│   ├── validate.py
│   ├── config.py
│   └── exceptions.py
└── tests/
    ├── test_transform.py
    └── test_validate.py
```

```python
# ecommerce_pipeline/main.py — the packaged entry point
import sys
import logging
from ecommerce_pipeline.config import load_settings
from ecommerce_pipeline.extract import extract
from ecommerce_pipeline.transform import transform
from ecommerce_pipeline.validate import validate
from ecommerce_pipeline.load import load
from ecommerce_pipeline.exceptions import DataQualityError

logging.basicConfig(level=logging.INFO, format="%(asctime)s [%(levelname)s] %(message)s")
logger = logging.getLogger(__name__)


def main() -> int:
    settings = load_settings()   # Module 8's config pattern — reads env/YAML
    try:
        raw = extract(settings)
        clean = transform(raw)
        validated = validate(clean)
        load(validated, settings)
        logger.info("Pipeline completed successfully")
        return 0
    except DataQualityError as e:
        logger.error(f"Data quality check failed: {e}")
        return 1
    except Exception:
        logger.exception("Pipeline failed unexpectedly")
        return 1


if __name__ == "__main__":
    sys.exit(main())
```

```
# .dockerignore — keep the image small and avoid leaking secrets into it
.venv/
__pycache__/
*.pyc
.env
.git/
tests/
*.md
```

With this structure, the pipeline can be run three different ways depending on context — locally during development (`poetry run run-pipeline`), as a scheduled cron job calling the same entry point directly (`python -m ecommerce_pipeline.main`), or as a container deployed anywhere Docker runs, including Azure Container Instances or an Azure Data Factory custom activity — all three running the exact same, tested code.

## COMMON MISTAKES

**1. Baking secrets directly into the Docker image**
```dockerfile
ENV DB_PASSWORD=supersecret123   # NEVER — this is baked into the image layer permanently,
                                     # visible to anyone who can pull or inspect the image
```
Fix: pass secrets at *runtime* (`docker run --env-file .env`, or a cloud platform's secret management), never bake them into the image itself — this is the Docker-level version of Module 7, Lesson 7.5's "never hardcode credentials" principle.

**2. Copying the entire project (including `.venv`, `.git`, test files) into the image unnecessarily**
Without a `.dockerignore`, a `COPY . .` instruction copies everything in the build context — including a potentially huge local `.venv` folder and `.git` history — dramatically bloating the image and slowing every build. Fix: always include a `.dockerignore` excluding exactly the categories shown in the worked example.

**3. Not pinning dependency versions, causing a build that worked last month to fail today**
```toml
pandas = "*"   # "any version" — a build run today might get pandas 3.0, silently breaking things
```
Fix: Poetry's `poetry.lock` file (Module 8, Lesson 8.3) already solves this — just make sure it's committed to version control and genuinely used in the Docker build (`poetry install` without regenerating the lock file), so the exact same dependency versions are used every time, everywhere.

**4. Running the container process as root**
```dockerfile
# no USER instruction — the container runs as root by default
CMD ["python", "-m", "ecommerce_pipeline.main"]
```
Fix: create and switch to a non-root user (as shown in the worked example's Dockerfile) — a real, if often overlooked, security practice; if the application is ever compromised, running as a non-root user limits what an attacker could do inside the container.

**5. Forgetting the entry point needs to work the SAME way locally and in the container**
If `main.py` relies on something that only exists in your local dev environment (a specific file path, an assumed working directory), it'll fail inside the container even though it worked on your laptop. Fix: use `pathlib` (Module 2) for all paths, read all configuration through the config/`.env` pattern (never a hardcoded local path), and actually test the containerized version locally (`docker run`) before considering the pipeline genuinely deployment-ready.

**Data Engineer Tip:** Containerizing your capstone project — even if you never actually deploy it to a cloud environment — is a strong, concrete signal on a portfolio repository: it demonstrates you understand deployment concerns, not just "the code runs on my machine." A `Dockerfile` plus a brief "how to run this with Docker" section in your README costs relatively little effort to add and meaningfully strengthens how production-ready the project reads to a reviewer, directly connecting this Python course's capstone to the containerization and deployment concepts you'll encounter continuing into Azure (Container Instances, Azure Kubernetes Service, or Data Factory's custom Docker-based activities).

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Write a minimal `pyproject.toml` for a small project called `my_pipeline`, with `pandas` and `sqlalchemy` as runtime dependencies and `pytest` as a dev dependency, following the structure shown in the Syntax Reference.

<details>
<summary>Hint</summary>
Follow the `[tool.poetry]`, `[tool.poetry.dependencies]`, and `[tool.poetry.group.dev.dependencies]` sections exactly as shown.
</details>

<details>
<summary>Solution</summary>

```toml
[tool.poetry]
name = "my-pipeline"
version = "0.1.0"
description = "A small ETL pipeline"
authors = ["Your Name <you@example.com>"]

[tool.poetry.dependencies]
python = "^3.12"
pandas = "^2.2"
sqlalchemy = "^2.0"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```
</details>

---

**Exercise 2 (Easy):** Write a `.dockerignore` file for a Python project that excludes virtual environments, Python bytecode cache, the `.env` file, Git history, and test files.

<details>
<summary>Hint</summary>
Follow the pattern shown in the worked example's `.dockerignore`.
</details>

<details>
<summary>Solution</summary>

```
.venv/
__pycache__/
*.pyc
.env
.git/
tests/
```
</details>

---

**Exercise 3 (Medium):** Write a minimal, working `Dockerfile` for a simple Python script `hello.py` that just prints `"Pipeline ran successfully"` — no external dependencies needed. It should use a slim Python 3.12 base image, copy the script in, and run it as the container's command.

<details>
<summary>Hint</summary>
`FROM python:3.12-slim`, `WORKDIR`, `COPY`, `CMD`.
</details>

<details>
<summary>Solution</summary>

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY hello.py .
CMD ["python", "hello.py"]
```
Build and run it:
```bash
docker build -t hello-test .
docker run hello-test
# Output: Pipeline ran successfully
```
</details>

---

**Exercise 4 (Medium):** Extend Exercise 3's Dockerfile to add a non-root user (`appuser`) and switch to it before running the command, following the security practice from the Common Mistakes section. Rebuild and confirm it still runs correctly.

<details>
<summary>Hint</summary>
`RUN useradd --create-home appuser` followed by `USER appuser`, placed before the `CMD` line.
</details>

<details>
<summary>Solution</summary>

```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY hello.py .

RUN useradd --create-home appuser
USER appuser

CMD ["python", "hello.py"]
```
```bash
docker build -t hello-test .
docker run hello-test
# Still prints: Pipeline ran successfully — but now as a non-root user inside the container
```
</details>

---

**Exercise 5 (Hard):** Design a complete, deployable package structure for a small pipeline with these requirements: (1) a `pyproject.toml` with `pandas`, `python-dotenv`, and `pandera` as dependencies, and `pytest`/`ruff` as dev dependencies, plus a `[tool.poetry.scripts]` entry point named `run-pipeline` pointing at `my_pipeline.main:main`; (2) a `Dockerfile` that installs dependencies via Poetry, copies only the application code (not tests), creates and switches to a non-root user, and runs the package as a module; (3) a `.dockerignore` excluding the categories from the Common Mistakes section; (4) a one-paragraph README section (just the text, no need for a real file) explaining how to run the pipeline three different ways: locally via Poetry, directly via `python -m`, and via Docker — mirroring the "three ways to run it" idea from the Worked Example.

<details>
<summary>Hint</summary>
This exercise is a synthesis of everything in the lesson — build each of the four pieces following the patterns shown in the Syntax Reference and Worked Example, adjusted for the specific package name `my_pipeline` and the specific dependency list given.
</details>

<details>
<summary>Solution</summary>

**pyproject.toml:**
```toml
[tool.poetry]
name = "my-pipeline"
version = "1.0.0"
description = "A production-grade ETL pipeline"
authors = ["Your Name <you@example.com>"]

[tool.poetry.dependencies]
python = "^3.12"
pandas = "^2.2"
python-dotenv = "^1.0"
pandera = "^0.20"

[tool.poetry.group.dev.dependencies]
pytest = "^8.0"
ruff = "^0.6"

[tool.poetry.scripts]
run-pipeline = "my_pipeline.main:main"

[build-system]
requires = ["poetry-core"]
build-backend = "poetry.core.masonry.api"
```

**Dockerfile:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app

COPY pyproject.toml poetry.lock ./
RUN pip install --no-cache-dir poetry \
    && poetry config virtualenvs.create false \
    && poetry install --no-root --no-dev

COPY my_pipeline/ ./my_pipeline/

RUN useradd --create-home appuser
USER appuser

CMD ["python", "-m", "my_pipeline.main"]
```

**.dockerignore:**
```
.venv/
__pycache__/
*.pyc
.env
.git/
tests/
*.md
```

**README section (text):**
> This pipeline can be run three ways. During local development, use `poetry install` followed by `poetry run run-pipeline`, which uses the entry point defined in `pyproject.toml`. To run it exactly as a scheduler (e.g. cron) would, use `python -m my_pipeline.main` directly from an activated Poetry-managed virtual environment. For a fully isolated, portable run — identical regardless of what's installed on the host machine — build and run the Docker image with `docker build -t my-pipeline . && docker run --env-file .env my-pipeline`, which packages the exact dependency versions from `poetry.lock` alongside the application code itself.
</details>

---

## Module 9 Complete

You've now covered the full lifecycle of a production ETL pipeline: designing clear extract/transform/load stages, building in idempotency and incremental loading, scheduling and orchestration concepts, automated data quality gates, logging and alerting, and packaging for real deployment. The capstone project brings every one of these together, along with everything from Modules 1–8, into one complete, portfolio-ready pipeline.

**Previous lesson:** [9.5 — Logging, Alerting & Failure Handling](lesson-9.5-alerting-and-failure-handling.md)
**Next:** [Capstone Project — Full ETL Pipeline](capstone-project.md)
