# Python for Data Analytics & Data Engineering

A complete, beginner-to-advanced Python course built around a single running `ecommerce` dataset — the same schema used in the companion [SQL course](#) — taking a learner from `print("hello")` through a tested, idempotent, production-minded ETL pipeline.

This course was written from scratch as a self-study and portfolio project while transitioning into data engineering. It's structured for anyone learning Python specifically for data analytics and data engineering work, not general-purpose software development.

## Why This Course Exists

Most "learn Python" material either stays too generic (never touching real data workflows) or jumps straight to pandas without building solid fundamentals first. This course is deliberately sequenced the other way: solid core Python first, then NumPy, then pandas, then the production practices (testing, logging, packaging) that separate a script from a pipeline — with every single lesson's examples and exercises drawn from one consistent, realistic e-commerce dataset rather than disconnected toy problems.

## Course Structure

| Module | Topic | Lessons |
|---|---|---|
| [0](module-0-setup/README.md) | Prerequisites & Setup | 2 |
| [1](module-1-python-fundamentals/README.md) | Python Fundamentals | 8 |
| [2](module-2-data-structures-and-files/README.md) | Core Data Structures & File Handling | 8 |
| [3](module-3-numpy/README.md) | NumPy — Numerical Computing | 7 |
| [4](module-4-pandas-1-data-wrangling/README.md) | Pandas I — Data Wrangling | 9 |
| [5](module-5-pandas-2-analysis-and-aggregation/README.md) | Pandas II — Analysis & Aggregation | 8 |
| [6](module-6-data-visualization/README.md) | Data Visualization | 6 |
| [7](module-7-databases-and-apis/README.md) | Python ↔ Databases & APIs | 7 |
| [8](module-8-production-grade-python/README.md) | Production-Grade Python for Data Engineering | 9 |
| [9](module-9-etl-automation-and-capstone/README.md) | ETL Automation & Capstone | 6 + capstone project |

See [ROADMAP.md](ROADMAP.md) for the full planning document, including pacing guidance, tool/library summary, and how each module maps onto a typical 6-month data engineering transition timeline.

## How Each Lesson Is Structured

Every lesson in Modules 0–9 follows the same six-part format:

1. **CONCEPT** — plain-language explanation of the idea
2. **WHY IT MATTERS** — the data analytics/engineering angle specifically
3. **SYNTAX REFERENCE** — a quick-lookup syntax box
4. **WORKED EXAMPLE** — a full example using the `ecommerce` dataset
5. **COMMON MISTAKES** — beginner pitfalls, with fixes
6. **PRACTICE EXERCISES** — 3–5 graded exercises (easy → hard), each with a hidden hint and full solution

Throughout, lessons include **"Data Engineer Tip"** callouts connecting the immediate concept to real pipeline work — and, from Module 4 onward, explicit cross-references to the SQL equivalent of whatever pandas operation is being taught, since the two courses share the same dataset.

Module 9 closes with a **capstone project**: a complete ETL pipeline — extract from a CSV and an API, clean and validate with pandas, idempotent load into PostgreSQL via SQLAlchemy, automated data quality checks, logging, and a tested, typed, Poetry-managed package — meant to be published as its own standalone portfolio repository.

## The `ecommerce` Dataset

Every lesson and exercise in this course draws on the same simulated e-commerce schema: `categories`, `products`, `customers`, `orders`, `order_items`, and `events`. This is the identical schema used throughout the companion SQL course, so a concept taught there (a `JOIN`, a `GROUP BY`, a window function) has a direct, nameable pandas equivalent here — and vice versa.

## Prerequisites

None — Module 0 starts from a blank machine with no Python installed. If you already have a working Python 3.12+ environment and an editor set up, you can skip straight to Module 1.

## Setup

```bash
git clone <this-repo-url>
cd python-for-data-analytics-and-data-engineering
python3 -m venv .venv
source .venv/bin/activate   # .venv\Scripts\activate on Windows
pip install -r requirements.txt
```

Each module's own README also lists the specific packages that module introduces, in case you'd rather install incrementally as you progress rather than all at once.

## Tech Stack Covered

Python 3.12+ · NumPy · pandas · matplotlib · seaborn · Plotly · SQLAlchemy · psycopg2 · python-dotenv · requests · tenacity · pytest · mypy · ruff · black · pre-commit · Poetry · PostgreSQL

## Companion Projects

- **SQL course** — the same `ecommerce` schema, taught from PostgreSQL fundamentals through window functions and query optimization
- **`data_jobs` EDA project** — a separate portfolio project applying SQL skills to a real job-postings dataset

## License

This course is released under the MIT License — see [LICENSE](LICENSE).
