# Capstone Project — Full ETL Pipeline

## Overview

Build a complete, production-minded ETL pipeline that extracts e-commerce order data from two different sources, cleans and validates it, loads it into the `ecommerce` PostgreSQL schema, runs an automated data quality check, and is packaged as a tested, typed, documented Python project — ready to publish as a standalone GitHub repository alongside your SQL course work.

This isn't a new lesson with a single right answer — it's a multi-session project that applies everything from Modules 1–9 together. Treat this document as a brief, not a tutorial: it tells you *what* to build and *what good looks like*, not exact code to copy.

## Learning Objectives

By completing this capstone, you'll demonstrate:
- Designing and implementing clear Extract → Transform → Load stages (Lesson 9.1)
- Writing idempotent, safely re-runnable load logic (Lesson 9.2)
- Structuring a real Python project with proper packaging (Lesson 8.2–8.3)
- Cleaning and validating data with pandas (Module 4) and vectorized operations (Module 3)
- Connecting to a database and an external API safely (Module 7)
- Automated data quality gates that can halt a bad load (Lesson 9.4)
- Logging and failure handling suitable for an unattended run (Lesson 9.5, 8.5)
- Unit tests and type hints for the core transformation logic (Lesson 8.6–8.7)
- Clear, professional documentation of the finished project

## Functional Requirements

Your pipeline must:

1. **Extract** data from at least two sources:
   - A CSV file (simulating a raw export — you can generate synthetic order data for this, following the `ecommerce` schema's shape from the SQL course)
   - A mock or real API source (a simple local JSON file served however you like, a public test API like `jsonplaceholder.typicode.com`, or a currency-conversion-style API — the specific choice matters less than demonstrating the `requests` + retry pattern from Module 7)

2. **Transform** the combined data with pandas:
   - Clean and standardize types (Lesson 4.6) — dates, prices, string fields
   - Handle missing values deliberately, not by accident (Lesson 4.5)
   - Remove duplicates using a sensible business key, not a full-row check (Lesson 4.8)
   - Compute at least one derived/aggregated field (e.g., order line totals, a customer-level rollup) using Module 5 techniques

3. **Load** the cleaned data into PostgreSQL, into the same `ecommerce` schema used throughout the SQL course (or a clearly separate staging schema/table if you'd rather not touch the original tables directly):
   - Use SQLAlchemy (Lesson 7.2–7.3), not raw string-built SQL
   - Make the load **idempotent** — running the pipeline twice on the same input should not create duplicate rows or corrupt the target table (Lesson 9.2)

4. **Validate** the data before (or immediately after) loading:
   - At least 3 concrete checks (e.g., no null order totals, no negative prices, row count within an expected range, no duplicate order IDs)
   - The pipeline should clearly **fail loudly** (non-zero exit, a logged error, or a raised exception) if a check fails — not silently continue with bad data

5. **Log** meaningful events throughout the run (Lesson 8.5) — start/end of each stage, row counts processed, any rows skipped or rejected and why, and any failures — using the `logging` module, not `print()`.

## Non-Functional Requirements

- **Structured as a proper package** (Lesson 8.2): separate modules for extract, transform, load, and validation logic — not one long script
- **Dependency-managed with Poetry** (Lesson 8.3), with a committed `pyproject.toml`
- **Configuration externalized** (Lesson 8.8): database connection details and any tunable values (batch size, file paths) come from a config file and/or `.env`, never hardcoded
- **Type-hinted** core functions, passing a `mypy` check (Lesson 8.7)
- **Unit tested** with `pytest` (Lesson 8.6) — focus tests on your transformation and validation logic specifically, since that's where the real business logic lives; you don't need to test the database connection itself
- **Linted and formatted** with `ruff`/`black`, ideally wired into a `pre-commit` hook (Lesson 8.9)
- **Runnable end-to-end** from a single command (a `main.py`, a CLI entry point, or a documented `poetry run` command)

## Suggested Project Structure

A reasonable starting layout — adapt as needed, but keep extract/transform/load/validate logically separated:

```
ecommerce-etl-pipeline/
├── pyproject.toml
├── .env.example              # documents required env vars, without real secrets
├── .gitignore
├── .pre-commit-config.yaml
├── README.md
├── config.yaml
├── src/
│   └── etl_pipeline/
│       ├── __init__.py
│       ├── extract.py         # reading the CSV + calling the API
│       ├── transform.py        # cleaning, type conversion, dedup, derived fields
│       ├── validate.py          # data quality checks
│       ├── load.py               # writing to PostgreSQL via SQLAlchemy
│       ├── config.py              # loads config.yaml + .env into a settings object
│       └── main.py                 # orchestrates extract -> transform -> validate -> load
├── tests/
│   ├── test_transform.py
│   └── test_validate.py
└── data/
    └── raw/
        └── sample_orders.csv    # or a script that generates synthetic data
```

## Suggested Build Order

Don't try to build this top-to-bottom in one sitting. A reasonable sequence:

1. **Scaffold the project** — `poetry init`, folder structure, `.gitignore`, empty modules
2. **Write `extract.py`** — get the CSV reading and the API call working independently, each returning a plain pandas DataFrame; test them manually in a REPL before wiring anything together
3. **Write `transform.py`** — clean and combine the two sources; this is usually the largest, most iterative piece — expect to revisit it as you discover messy edge cases in your synthetic data
4. **Write `validate.py`** — define your data quality checks as clear, named functions that each return a pass/fail (and a reason on failure)
5. **Write `load.py`** — the idempotent write to PostgreSQL; test against a disposable staging table first before pointing it at anything you care about
6. **Write `main.py`** — orchestrate the four stages in order, with logging at each step, and a clear failure path if validation fails
7. **Add tests** for `transform.py` and `validate.py` — these are the easiest and most valuable to test, since they're pure functions with no external dependencies
8. **Add type hints, run `mypy`, run `ruff`/`black`**, and set up pre-commit
9. **Write the README** last, once the pipeline actually works end-to-end

## A Note on Scope

It's tempting to make this capstone bigger than it needs to be — a full orchestration setup with Airflow, multiple environments, a dozen data sources. Resist that. The goal is a **small, complete, genuinely solid** pipeline — every requirement above actually met and working — rather than a large, ambitious pipeline with gaps. A reviewer (or you, six months from now) will trust a small pipeline with real tests and real idempotency far more than an elaborate one that only works when nothing goes wrong.

## Self-Assessment Rubric

Before considering this done, check each of these honestly:

| Area | Check |
|---|---|
| Extract | Pulls from 2 distinct sources (file + API), each in its own function |
| Transform | Handles missing data, duplicates, and type cleaning deliberately, with clear reasoning |
| Load | Writing to PostgreSQL via SQLAlchemy; running the pipeline twice produces the same end state, not duplicated rows |
| Validate | At least 3 real checks; a failed check visibly halts or flags the run, doesn't silently continue |
| Logging | Every stage logs its start, completion, and row counts; failures are logged with enough detail to debug without re-running |
| Structure | Logically separated modules; no single file doing everything |
| Config | No hardcoded credentials or paths; a `.env.example` documents what's needed |
| Tests | `pytest` tests exist for transform/validate logic and actually pass |
| Types | Core functions have type hints; `mypy` runs clean (or documented exceptions) |
| Style | `ruff`/`black` pass cleanly; ideally enforced via pre-commit |
| Docs | A README explains what the pipeline does, how to run it, and what it depends on |

## Optional Stretch Goals

If the core pipeline is solid and you want to push further:
- Add a simple **incremental load** mode using a watermark column (Lesson 9.2), rather than always reprocessing everything
- Wrap the pipeline in a basic **cron** schedule, or sketch out what it would look like as an Airflow DAG (Lesson 9.3) — even just a documented diagram counts as meaningful progress
- Add a **Slack or email alert** (even just a stubbed/logged version) on pipeline failure (Lesson 9.5)
- Containerize the pipeline with a simple `Dockerfile` (a preview of Lesson 9.6's packaging ideas taken further)
- Add a small `pandera` schema (Lesson 9.4) to formalize your validation rules instead of hand-written check functions

## Publishing This as a Portfolio Project

Once finished, this belongs on GitHub as its own repository, following the same practices already used for your `ecommerce` SQL course and `data_jobs` project: a clear README with setup instructions, badges or a short note on test/lint status, and — ideally — a simple CI workflow (e.g., GitHub Actions) that runs `pytest`, `mypy`, and `ruff check` automatically on every push. That last piece in particular is a strong, concrete signal to anyone reviewing your profile that this is genuinely production-minded work, not a one-off tutorial exercise.

This project is also the natural bridge into Phase 2 of your Azure roadmap: the cleaned, loaded `ecommerce` data this pipeline produces is exactly the kind of source table a dbt model or a Synapse pipeline would consume next.

---

**Previous lesson:** [9.6 — Packaging the Pipeline for Deployment](lesson-9.6-packaging-for-deployment.md)

## Course Complete

This capstone closes out the full Python for Data Analytics & Data Engineering course — from `print("hello")` in Module 1 to a tested, idempotent, scheduled-ready ETL pipeline here in Module 9. Combined with the companion SQL course, you now have two complete, portfolio-ready bodies of work covering both the query layer and the pipeline layer of a modern data stack.
