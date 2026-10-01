# Module 7: Python ↔ Databases & APIs

Every module so far has worked with data that was already sitting in a variable or a local file. Module 7 is about *getting* data from where it actually lives — a database, or an external API — and about doing so safely: with proper credential handling, protection against SQL injection, and resilience against network failures.

This module connects directly to the same `ecommerce` PostgreSQL database used throughout the companion SQL course. If you haven't set that database up, revisit the SQL course's setup instructions before starting Lesson 7.1.

## Lessons

| # | Lesson | File |
|---|--------|------|
| 7.1 | Connecting to PostgreSQL | `lesson-7.1-connecting-to-postgresql.md` |
| 7.2 | SQLAlchemy: Engines & `pd.read_sql()` | `lesson-7.2-sqlalchemy-and-read-sql.md` |
| 7.3 | Writing DataFrames Back to a Database | `lesson-7.3-writing-to-a-database.md` |
| 7.4 | Parameterized Queries & SQL Injection | `lesson-7.4-parameterized-queries.md` |
| 7.5 | Managing Credentials with `python-dotenv` | `lesson-7.5-credentials-and-dotenv.md` |
| 7.6 | Calling REST APIs with `requests` | `lesson-7.6-calling-rest-apis.md` |
| 7.7 | Retry Logic & Resilience with `tenacity` | `lesson-7.7-retry-logic-and-resilience.md` |

## Setup

This module requires several new packages:

```bash
pip install psycopg2-binary sqlalchemy python-dotenv requests tenacity
```

You'll also need a running PostgreSQL instance with the `ecommerce` schema loaded — the same database from the SQL course. Connection details (host, port, database name, username, password) should never be hardcoded in your scripts; Lesson 7.5 covers exactly how to handle them safely, and every later lesson in this module follows that pattern.

## Lesson Format

Same six-part structure as prior modules: CONCEPT → WHY IT MATTERS → SYNTAX REFERENCE → WORKED EXAMPLE → COMMON MISTAKES → PRACTICE EXERCISES, with hidden hints/solutions and a Data Engineer Tip per lesson.

**Data Engineer Tip (module-level):** This module is where the course stops being "Python for analysis" and starts being genuinely "Python for data engineering." Every lesson here — safe credentials, parameterized queries, retry logic — exists because pipeline code runs unattended, against real production systems, often on a schedule with nobody watching. The habits in this module are exactly what separates a script that works once on your laptop from one that can be trusted to run in production.

---

**Previous module:** [Module 6 — Data Visualization](../module-6-data-visualization/README.md)
**Next module:** [Module 8 — Production-Grade Python for Data Engineering](../module-8-production-grade-python/README.md)
