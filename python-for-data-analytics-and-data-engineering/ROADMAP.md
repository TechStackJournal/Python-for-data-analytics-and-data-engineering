# Python for Data Analytics & Data Engineering
## A Comprehensive Course Roadmap & Agenda

---

## 0. Course Philosophy & Design

This course follows the same design principles as the companion SQL course:

- **One consistent dataset throughout**: the `ecommerce` schema (`categories`, `products`, `customers`, `orders`, `order_items`, `events`). Learners already familiar with the SQL course will recognize every table — Python is taught as the layer that *extracts, cleans, analyzes, and moves* this same data.
- **Beginner-friendly, zero assumed background**: starts from "what is a variable" and ends at production ETL scripts.
- **Six-part lesson format** (mirrors the SQL course):
  1. **CONCEPT** — plain-language explanation
  2. **WHY IT MATTERS** — the data analytics/engineering angle
  3. **SYNTAX REFERENCE** — quick-lookup syntax box
  4. **WORKED EXAMPLE** — a full example using the `ecommerce` dataset
  5. **COMMON MISTAKES** — beginner pitfalls, with fixes
  6. **PRACTICE EXERCISES** — 3–5 graded exercises with `<details>`-hidden hints/solutions
- **"Data Engineer Tip" callouts** in every lesson connecting the concept to real pipeline work (Azure, dbt, orchestration).
- **Real-world scenarios**, not toy problems: every module ends with a scenario like *"Marketing wants weekly cohort retention"* or *"The nightly load failed at 2am — find out why."*

---

## 1. Prerequisites & Setup

No prior programming experience required. Recommended before Module 1:

- Python 3.12+ installed
- VS Code (or PyCharm Community) with the Python extension
- `venv` or `poetry` for environment management
- Git + GitHub account (for the same repo-publishing workflow used in the SQL course)
- Access to the same PostgreSQL 16.x `ecommerce` database used in the SQL course (Modules 6–8 connect directly to it)

**Setup Lesson (Module 0)**
- Installing Python, virtual environments, `pip` vs `poetry`
- VS Code setup: interpreter selection, linting (`ruff`), formatting (`black`)
- Running your first script vs. using a REPL vs. Jupyter/IPython
- Data Engineer Tip: why production pipelines never rely on a "system Python"

---

## 2. Course Map (9 Modules)

| # | Module | Focus | Est. Time |
|---|--------|-------|-----------|
| 1 | Python Fundamentals | Syntax, variables, control flow, functions | 10–12 hrs |
| 2 | Core Data Structures & Files | Lists, dicts, sets, comprehensions, file I/O | 8–10 hrs |
| 3 | NumPy — Numerical Computing | Arrays, vectorization, broadcasting | 6–8 hrs |
| 4 | Pandas I — Data Wrangling | DataFrames, cleaning, transformation | 12–15 hrs |
| 5 | Pandas II — Analysis & Aggregation | GroupBy, merges, time series, pivoting | 10–12 hrs |
| 6 | Data Visualization | matplotlib, seaborn, plotly | 6–8 hrs |
| 7 | Python ↔ Databases & APIs | SQLAlchemy, psycopg2/pyodbc, REST APIs | 8–10 hrs |
| 8 | Production-Grade Python for DE | OOP, error handling, testing, packaging, logging | 10–12 hrs |
| 9 | ETL Automation & Capstone | Scheduling, config, orchestration, full pipeline project | 10–14 hrs |

**Total: ~80–100 hours**, self-paced (aligns with Phase 1 of your existing 6-month Azure DE roadmap).

---

## 3. Detailed Module Breakdown

### Module 1 — Python Fundamentals
*Goal: comfort with core syntax before touching any data.*

1.1 Variables, data types, and type conversion
1.2 Operators and expressions
1.3 Strings and string methods (parsing raw data — think messy CSV fields)
1.4 Conditional logic (`if`/`elif`/`else`)
1.5 Loops (`for`, `while`) and loop control
1.6 Functions: parameters, return values, default args, `*args`/`**kwargs`
1.7 Scope and namespaces
1.8 Error handling basics (`try`/`except`)

**Real-world scenario:** Write a script that validates a raw list of customer emails (format checks, type coercion, flagging bad rows) — the exact kind of row-level validation that precedes any load into the `ecommerce.customers` table.

---

### Module 2 — Core Data Structures & File Handling
*Goal: represent and manipulate real data before introducing pandas.*

2.1 Lists, tuples, sets — when to use each
2.2 Dictionaries and nested structures (mirroring JSON)
2.3 List/dict/set comprehensions
2.4 Working with `datetime` and time zones
2.5 Reading/writing CSV with the `csv` module
2.6 Reading/writing JSON with the `json` module
2.7 Working with file paths (`pathlib`)
2.8 Intro to regular expressions for text cleaning

**Real-world scenario:** Parse a raw clickstream export (JSON lines) into a clean list of dictionaries, matching the shape of the `ecommerce.events` table — deduplicating and standardizing timestamps.

**Data Engineer Tip:** why comprehensions are preferred over manual loops in pipeline code, and where they hurt readability.

---

### Module 3 — NumPy: Numerical Computing
*Goal: understand vectorized computation, the foundation pandas is built on.*

3.1 Arrays vs. Python lists — why NumPy is faster
3.2 Creating arrays, shapes, dtypes
3.3 Indexing, slicing, boolean masking
3.4 Vectorized operations and broadcasting
3.5 Aggregations (`sum`, `mean`, `std`, `percentile`)
3.6 `NaN` handling basics
3.7 Random sampling (`np.random`) for synthetic test data

**Real-world scenario:** Compute order-value statistics (mean, median, outlier thresholds via IQR) across 500K simulated `order_items` rows, and compare loop-based vs. vectorized runtime.

---

### Module 4 — Pandas I: Data Wrangling
*Goal: pandas as the primary daily tool for a data analyst/engineer.*

4.1 Series and DataFrame fundamentals
4.2 Reading data: CSV, JSON, Excel, SQL (preview of Module 7)
4.3 Inspecting data: `.info()`, `.describe()`, `.head()`, dtypes
4.4 Selecting and filtering: `.loc`, `.iloc`, boolean indexing
4.5 Handling missing data: detection, imputation, dropping
4.6 Data type conversion and cleaning (strings → dates, currency parsing)
4.7 Renaming, reordering, dropping columns
4.8 Detecting and handling duplicates
4.9 Applying functions: `.apply()`, `.map()`, `.replace()`

**Real-world scenario:** Load a messy CSV export of `products` (inconsistent casing, currency symbols, nulls in `category_id`) and produce a cleaned DataFrame ready for loading — directly paralleling the `TRUNCATE`/reseed pattern from the SQL course.

**Common Mistake callout:** chained indexing and `SettingWithCopyWarning` — a top beginner trap.

---

### Module 5 — Pandas II: Analysis & Aggregation
*Goal: answer real business questions with data.*

5.1 `groupby()` mechanics: split-apply-combine
5.2 Aggregation functions and multiple aggregations (`.agg()`)
5.3 Pivot tables and `crosstab()`
5.4 Merging and joining DataFrames (inner/left/right/outer — same logic as SQL joins, deliberately cross-referenced)
5.5 Concatenating and appending datasets
5.6 Time series basics: resampling, rolling windows, `.dt` accessor
5.7 Window functions in pandas (`.rolling()`, `.expanding()`) — parallel to SQL window functions from Module 3 of the SQL course
5.8 Cohort analysis patterns

**Real-world scenario:** Recreate a monthly revenue-by-category report and a customer retention cohort matrix, using the same business questions tackled in the SQL course's Aggregation module — showing the same answer, two tools.

**Data Engineer Tip:** when to push aggregation to the database (SQL) vs. do it in pandas, and why "filter early, in SQL" usually wins on large datasets.

---

### Module 6 — Data Visualization
*Goal: communicate findings, not just compute them.*

6.1 matplotlib fundamentals: figures, axes, subplots
6.2 Line, bar, histogram, and scatter plots
6.3 seaborn for statistical visualization (distributions, boxplots, heatmaps)
6.4 Customization: labels, legends, styling, saving figures
6.5 Intro to interactive visualization with Plotly
6.6 Choosing the right chart for the question being asked

**Real-world scenario:** Build a dashboard-style set of charts — daily order volume trend, category revenue breakdown, funnel drop-off from `events` — as a static report a stakeholder could open directly.

---

### Module 7 — Python ↔ Databases & APIs
*Goal: connect Python directly to your existing PostgreSQL `ecommerce` database.*

7.1 Connecting to PostgreSQL with `psycopg2` / `pyodbc`
7.2 Using SQLAlchemy: engines, connections, `pd.read_sql()`
7.3 Writing DataFrames back to a database (`.to_sql()`, upserts)
7.4 Parameterized queries and SQL injection avoidance
7.5 Managing credentials safely with `python-dotenv`
7.6 Calling REST APIs with `requests` (pagination, auth headers, rate limits)
7.7 Retry logic and resilience with `tenacity`

**Real-world scenario:** Pull the current `ecommerce.orders` table into a DataFrame, enrich it with a mock currency-conversion API call, and write the enriched result back to a new staging table — the same `search_path` gotcha you documented in the SQL course applies here too, called out explicitly.

**Data Engineer Tip:** why credentials belong in `.env`/Key Vault, never hardcoded — direct lead-in to Azure Key Vault in your DE roadmap.

---

### Module 8 — Production-Grade Python for Data Engineering
*Goal: write code that survives contact with a scheduler and a code reviewer.*

8.1 Functions to classes: intro to OOP for data pipelines
8.2 Structuring a project: modules, packages, `__init__.py`
8.3 Environment and dependency management with `poetry`
8.4 Robust error handling and custom exceptions
8.5 Logging (the `logging` module) instead of `print()`
8.6 Writing tests with `pytest` — unit tests for transformation logic
8.7 Type hints and static checking with `mypy`
8.8 Configuration management (YAML/`.env`, avoiding hardcoded paths)
8.9 Code style, linting, and pre-commit hooks

**Real-world scenario:** Refactor the Module 7 script into a proper package: a `transform.py` module with tested, type-hinted functions, a `config.yaml`, and a `pytest` suite — the transition point from "notebook code" to "pipeline code."

---

### Module 9 — ETL Automation & Capstone
*Goal: ship an end-to-end pipeline, portfolio-ready.*

9.1 Designing an ETL job: extract, transform, load stages
9.2 Idempotency and incremental loads (watermarking, `RESTART IDENTITY`-style resets)
9.3 Scheduling basics: cron, and an intro to orchestration (Airflow/ADF concepts)
9.4 Data quality checks (`pandera`/`great_expectations` intro, or manual assertions)
9.5 Logging, alerting, and failure handling in a scheduled job
9.6 Packaging the pipeline for deployment

**Capstone Project:** Build a full ETL pipeline that:
- Extracts raw e-commerce order data (CSV + a mock API source)
- Cleans and validates it with pandas
- Loads it into the PostgreSQL `ecommerce` schema using SQLAlchemy
- Runs a data quality check and logs the result
- Is wrapped in a tested, typed, `poetry`-managed package
- Includes a README documenting the pipeline (same repo-publishing pattern as the SQL course)

This capstone becomes the natural bridge into **Phase 2** of your existing Azure roadmap (dbt + dimensional modeling + Synapse), since the pipeline it produces lands in the same warehouse schema your dbt models will consume.

---

## 4. How This Connects to Your Existing Plans

- **SQL course**: same `ecommerce` schema throughout, so every pandas exercise has a "how would I do this in SQL?" cross-reference already written.
- **Azure DE roadmap Phase 1** (production-grade Python): Module 8 of this course *is* that phase, just fleshed out lesson-by-lesson.
- **DP-203 certification**: Modules 7–9 map directly onto exam domains around data ingestion and transformation.
- **Portfolio project**: the Module 9 capstone is a ready-made GitHub artifact, publishable the same way as `sql-course.zip`.

---

## 5. Suggested Pacing

- **5–10 hrs/week** → course completed in ~10–12 weeks
- Recommended order: strictly sequential through Module 5 (foundations build on each other), then Modules 6–9 can flex based on whether visualization or database connectivity is more urgent for your job search timeline.

---

## 6. Tools & Resources Summary

| Category | Tools |
|---|---|
| Core | Python 3.12+, VS Code, `venv`/`poetry` |
| Data | NumPy, pandas |
| Visualization | matplotlib, seaborn, plotly |
| Database | SQLAlchemy, psycopg2, pyodbc, python-dotenv |
| Reliability | tenacity, logging |
| Quality | pytest, mypy, ruff/black |
| Target DB | PostgreSQL 16.x (`ecommerce` schema, shared with SQL course) |
