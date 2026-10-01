# Module 5: Pandas II — Analysis & Aggregation

Module 4 was about getting data clean and shaped correctly. Module 5 is about actually *answering questions* with it — grouping, aggregating, combining multiple tables, working with time series, and building the kind of cohort/retention analysis that shows up constantly in real analytics work.

Every lesson continues using the `ecommerce` schema. Several lessons deliberately mirror business questions already answered in the companion SQL course's Aggregation and Window Functions modules — so you can compare the same answer produced two different ways.

## Lessons

| # | Lesson | File |
|---|--------|------|
| 5.1 | `groupby()` Mechanics | `lesson-5.1-groupby-mechanics.md` |
| 5.2 | Aggregation Functions & `.agg()` | `lesson-5.2-aggregation-functions.md` |
| 5.3 | Pivot Tables & `crosstab()` | `lesson-5.3-pivot-tables.md` |
| 5.4 | Merging & Joining DataFrames | `lesson-5.4-merging-and-joining.md` |
| 5.5 | Concatenating & Appending | `lesson-5.5-concatenating.md` |
| 5.6 | Time Series Basics | `lesson-5.6-time-series-basics.md` |
| 5.7 | Window Functions (`.rolling()`, `.expanding()`) | `lesson-5.7-window-functions.md` |
| 5.8 | Cohort Analysis Patterns | `lesson-5.8-cohort-analysis.md` |

## Setup

Same as Module 4 — pandas and NumPy:

```bash
pip install pandas numpy
```

```python
import pandas as pd
import numpy as np
```

## Lesson Format

Same six-part structure as prior modules: CONCEPT → WHY IT MATTERS → SYNTAX REFERENCE → WORKED EXAMPLE → COMMON MISTAKES → PRACTICE EXERCISES, with hidden hints/solutions and a Data Engineer Tip per lesson.

**Data Engineer Tip (module-level):** Every technique here has a SQL equivalent you've already learned: `groupby()` is `GROUP BY`, `.merge()` is `JOIN`, `.rolling()`/`.expanding()` are window functions, `pd.concat()` is `UNION ALL`. A genuinely important data engineering judgment call — touched on throughout this module — is *where* this work should actually happen: pushed down into SQL (closer to the data, often faster on large volumes) versus done in pandas (more flexible, easier to iterate on). Neither is universally "correct" — the right answer depends on data volume, where the data already lives, and how the result will be reused.

---

**Previous module:** [Module 4 — Pandas I: Data Wrangling](../module-4-pandas-1-data-wrangling/README.md)
**Next module:** [Module 6 — Data Visualization](../module-6-data-visualization/README.md)
