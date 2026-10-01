# Module 4: Pandas I — Data Wrangling

This is the module most of the course has been building toward. **pandas** is the primary tool you'll use for the rest of your data analytics work — a `DataFrame` is essentially a table (rows and columns, like a spreadsheet or a SQL result set) built on top of the NumPy arrays from Module 3, with each column free to have its own dtype.

Every lesson continues using the `ecommerce` schema — now loaded as real DataFrames from CSV/JSON files, exactly as you'd receive them from a database export or an API.

## Lessons

| # | Lesson | File |
|---|--------|------|
| 4.1 | Series & DataFrame Fundamentals | `lesson-4.1-series-and-dataframes.md` |
| 4.2 | Reading Data: CSV, JSON, Excel, SQL | `lesson-4.2-reading-data.md` |
| 4.3 | Inspecting Data | `lesson-4.3-inspecting-data.md` |
| 4.4 | Selecting & Filtering | `lesson-4.4-selecting-and-filtering.md` |
| 4.5 | Handling Missing Data | `lesson-4.5-missing-data.md` |
| 4.6 | Data Type Conversion & Cleaning | `lesson-4.6-type-conversion-and-cleaning.md` |
| 4.7 | Renaming, Reordering & Dropping Columns | `lesson-4.7-column-management.md` |
| 4.8 | Detecting & Handling Duplicates | `lesson-4.8-duplicates.md` |
| 4.9 | Applying Functions | `lesson-4.9-apply-map-replace.md` |

## Setup

This module requires pandas (and, for Lesson 4.2's Excel section, `openpyxl`) in your virtual environment:

```bash
pip install pandas openpyxl
```

Every lesson assumes this import at the top of your script:
```python
import pandas as pd
import numpy as np
```

## Lesson Format

Same six-part structure as prior modules: CONCEPT → WHY IT MATTERS → SYNTAX REFERENCE → WORKED EXAMPLE → COMMON MISTAKES → PRACTICE EXERCISES, with hidden hints/solutions and a Data Engineer Tip per lesson.

**Data Engineer Tip (module-level):** Everything you learned in Module 3 about NumPy applies directly here — a DataFrame column (a `Series`) is a labeled NumPy array, so vectorized operations, boolean masking, and broadcasting all work exactly the same way, just through pandas' interface. If a pandas behavior ever seems confusing, ask "what would the underlying NumPy array do here?" — the answer is usually the explanation.

---

**Previous module:** [Module 3 — NumPy](../module-3-numpy/README.md)
**Next module:** [Module 5 — Pandas II: Analysis & Aggregation](../module-5-pandas-2-analysis-and-aggregation/README.md)
