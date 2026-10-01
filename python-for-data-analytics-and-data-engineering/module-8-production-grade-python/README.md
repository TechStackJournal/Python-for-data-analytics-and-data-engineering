# Module 8: Production-Grade Python for Data Engineering

Every module so far has focused on getting a task done. This module focuses on something different: writing code that other people — and future you — can trust, extend, and run unattended. This is the module that turns "a script that works on my laptop" into "a pipeline component ready to be scheduled, reviewed, and deployed."

This module is deliberately less tied to the `ecommerce` dataset itself and more focused on *how you organize and harden* the code you've already been writing to work with it throughout Modules 4–7.

## Lessons

| # | Lesson | File |
|---|--------|------|
| 8.1 | From Functions to Classes: Intro to OOP | `lesson-8.1-oop-basics.md` |
| 8.2 | Structuring a Project: Modules & Packages | `lesson-8.2-project-structure.md` |
| 8.3 | Dependency Management with Poetry | `lesson-8.3-poetry.md` |
| 8.4 | Robust Error Handling & Custom Exceptions | `lesson-8.4-custom-exceptions.md` |
| 8.5 | Logging Instead of `print()` | `lesson-8.5-logging.md` |
| 8.6 | Writing Tests with `pytest` | `lesson-8.6-pytest.md` |
| 8.7 | Type Hints & Static Checking with `mypy` | `lesson-8.7-type-hints-and-mypy.md` |
| 8.8 | Configuration Management | `lesson-8.8-configuration-management.md` |
| 8.9 | Code Style, Linting & Pre-commit Hooks | `lesson-8.9-linting-and-pre-commit.md` |

## Setup

This module introduces several new tools. Install them as you reach each lesson (each lesson's own setup notes cover the specifics), or install everything up front:

```bash
pip install pytest mypy ruff black pyyaml pre-commit
```

Lesson 8.3 additionally covers installing Poetry itself, which is a separate, standalone installation rather than a `pip install`.

## Lesson Format

Same six-part structure as prior modules: CONCEPT → WHY IT MATTERS → SYNTAX REFERENCE → WORKED EXAMPLE → COMMON MISTAKES → PRACTICE EXERCISES, with hidden hints/solutions and a Data Engineer Tip per lesson.

**Data Engineer Tip (module-level):** Nothing in this module is about making code "fancier" for its own sake — every practice here exists because it answers a real production question: can this run without a human watching it (logging, error handling)? Can a teammate understand and extend it (structure, type hints)? Can we trust a change didn't break something (tests)? Will it work the same way on a different machine (Poetry, config management)? By the end of this module, the Module 9 capstone ETL pipeline will apply every single one of these practices together in one real project.

---

**Previous module:** [Module 7 — Python ↔ Databases & APIs](../module-7-databases-and-apis/README.md)
**Next module:** [Module 9 — ETL Automation & Capstone](../module-9-etl-automation-and-capstone/README.md)
