# Module 9: ETL Automation & Capstone

This is the final module — where everything from Modules 1–8 comes together into one real, scheduled, tested pipeline. You've already written every individual piece: reading and cleaning data (Module 4), aggregating it (Module 5), connecting to a database and an API (Module 7), and writing production-grade code with proper structure, logging, and tests (Module 8). This module is about assembling those pieces into something that can genuinely run unattended, on a schedule, and be trusted to tell you when something's wrong.

The module ends with a capstone project: a complete ETL pipeline you'll build, test, and document — ready to publish as a portfolio piece alongside your SQL course work.

## Lessons

| # | Lesson | File |
|---|--------|------|
| 9.1 | Designing an ETL Job: Extract, Transform, Load | `lesson-9.1-etl-design.md` |
| 9.2 | Idempotency & Incremental Loads | `lesson-9.2-idempotency-and-incremental-loads.md` |
| 9.3 | Scheduling & an Intro to Orchestration | `lesson-9.3-scheduling-and-orchestration.md` |
| 9.4 | Data Quality Checks | `lesson-9.4-data-quality-checks.md` |
| 9.5 | Logging, Alerting & Failure Handling | `lesson-9.5-alerting-and-failure-handling.md` |
| 9.6 | Packaging the Pipeline for Deployment | `lesson-9.6-packaging-for-deployment.md` |
| Capstone | Full ETL Pipeline Project | `capstone-project.md` |

## Setup

This module draws on every tool installed across Modules 4–8: pandas, SQLAlchemy, psycopg2, python-dotenv, requests, tenacity, pytest, mypy, and a YAML config library. One new addition for data quality checks:

```bash
pip install pandera
```

## Lesson Format

Lessons 9.1–9.6 follow the same six-part structure as every prior module: CONCEPT → WHY IT MATTERS → SYNTAX REFERENCE → WORKED EXAMPLE → COMMON MISTAKES → PRACTICE EXERCISES, with hidden hints/solutions and a Data Engineer Tip per lesson. The capstone document is structured differently — as a project brief with requirements, a suggested architecture, and a grading rubric — since it's meant to be built over several sessions rather than worked through as a single lesson.

**Data Engineer Tip (module-level):** This module — and the capstone specifically — is the piece of this course most worth publishing to GitHub as a standalone portfolio repository, following the same practices already used for the `ecommerce` SQL course and the `data_jobs` project: a clear README, a documented design decisions log, idempotency and data quality checks built in from the start, and CI running tests automatically on every push. A reviewer skimming your GitHub profile should be able to tell within thirty seconds that this is production-minded work, not a tutorial copy.

---

**Previous module:** [Module 8 — Production-Grade Python for Data Engineering](../module-8-production-grade-python/README.md)
**This is the final module of the course.**
