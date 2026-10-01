# Module 0: Prerequisites & Setup

Before writing any Python for this course, get your environment in place. This module is short and entirely hands-on — by the end, you'll have Python installed, a project folder set up with a virtual environment, and VS Code configured to write and run code comfortably.

You only need to do this once. If you already have a working Python 3.12+ setup with VS Code, skim this module to confirm your setup matches, then move straight to Module 1.

## Lessons

| # | Lesson | File |
|---|--------|------|
| 0.1 | Installing Python & a Virtual Environment | `lesson-0.1-installing-python.md` |
| 0.2 | VS Code Setup & Running Your First Script | `lesson-0.2-vscode-and-first-script.md` |

## Why This Module Exists

It's tempting to skip setup and jump straight to "real" code — but a broken or inconsistent environment is one of the most common sources of confusing, hard-to-diagnose beginner problems ("it works on the tutorial but not for me"). Getting this right once, carefully, avoids hours of frustration later. It's also your first hands-on encounter with a habit that matters a lot in data engineering: **never work in a "bare" system-wide Python** — always in an isolated, reproducible environment.

**Data Engineer Tip (module-level):** Every pipeline you eventually deploy — locally, in Airflow, in Azure Data Factory, in a Docker container — depends on a controlled, reproducible environment with a known Python version and known package versions. The virtual environment habit you build in Lesson 0.1 today is the same principle behind `poetry` (Module 8) and containerized deployments later in your Azure roadmap. It's a small thing now, but it's foundational.

---

**Next module:** [Module 1 — Python Fundamentals](../module-1-python-fundamentals/README.md)
