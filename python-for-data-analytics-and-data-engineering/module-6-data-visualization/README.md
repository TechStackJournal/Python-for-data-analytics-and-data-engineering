# Module 6: Data Visualization

Every module so far has been about computing the right numbers. This module is about making those numbers *understandable* to someone else — a stakeholder, a hiring manager reviewing your portfolio, or future you six months from now. Analysis nobody can quickly read isn't finished analysis.

Every lesson continues using the `ecommerce` dataset (orders, products, categories, customers) — the same data you've been aggregating in Module 5, now turned into charts.

## Lessons

| # | Lesson | File |
|---|--------|------|
| 6.1 | matplotlib Fundamentals | `lesson-6.1-matplotlib-fundamentals.md` |
| 6.2 | Line, Bar, Histogram & Scatter Plots | `lesson-6.2-core-chart-types.md` |
| 6.3 | seaborn for Statistical Visualization | `lesson-6.3-seaborn.md` |
| 6.4 | Customization: Labels, Legends, Styling & Saving | `lesson-6.4-customization-and-saving.md` |
| 6.5 | Intro to Interactive Visualization with Plotly | `lesson-6.5-plotly-intro.md` |
| 6.6 | Choosing the Right Chart | `lesson-6.6-choosing-the-right-chart.md` |

## Setup

This module requires matplotlib, seaborn, and Plotly:

```bash
pip install matplotlib seaborn plotly
```

Every lesson assumes these imports at the top of your script:
```python
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
```
(Lesson 6.5 introduces its own Plotly-specific imports.)

## A Note on Running These Lessons

Unlike earlier modules, this one produces *visual* output — a chart window, or a saved image file — rather than printed text. Every worked example and exercise solution includes a comment describing what the resulting chart looks like, but you'll get the most out of this module by actually running the code and looking at the real output. If you're working in a plain `.py` script, add `plt.show()` at the end of any matplotlib example to open the chart in a window; if you're in a Jupyter-style notebook, charts display automatically after the cell runs.

## Lesson Format

Same six-part structure as prior modules: CONCEPT → WHY IT MATTERS → SYNTAX REFERENCE → WORKED EXAMPLE → COMMON MISTAKES → PRACTICE EXERCISES, with hidden hints/solutions and a Data Engineer Tip per lesson.

**Data Engineer Tip (module-level):** Charts you build here are also exactly what Module 9's ETL pipeline capstone can use for a lightweight "did this pipeline run produce sane data?" sanity check — a quick histogram of a key metric after each load is a fast, human-readable way to catch an upstream data quality problem that a pure numeric check might miss.

---

**Previous module:** [Module 5 — Pandas II: Analysis & Aggregation](../module-5-pandas-2-analysis-and-aggregation/README.md)
**Next module:** [Module 7 — Python ↔ Databases & APIs](../module-7-databases-and-apis/README.md)
