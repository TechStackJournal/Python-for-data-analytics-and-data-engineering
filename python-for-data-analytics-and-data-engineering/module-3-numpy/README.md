# Module 3: NumPy — Numerical Computing

NumPy is the foundation nearly every data science and analytics library in Python is built on — pandas, scikit-learn, and matplotlib all use NumPy arrays under the hood. This module shifts your mental model from "loop over values one at a time" (Modules 1–2) to **vectorized thinking**: applying an operation to an entire array at once, which is both faster and more concise.

Every lesson continues using e-commerce data — now represented as NumPy arrays of prices, quantities, and ratings drawn from the `ecommerce` schema.

## Lessons

| # | Lesson | File |
|---|--------|------|
| 3.1 | Arrays vs. Python Lists | `lesson-3.1-arrays-vs-lists.md` |
| 3.2 | Creating Arrays, Shapes & Dtypes | `lesson-3.2-creating-arrays.md` |
| 3.3 | Indexing, Slicing & Boolean Masking | `lesson-3.3-indexing-and-masking.md` |
| 3.4 | Vectorized Operations & Broadcasting | `lesson-3.4-vectorization-and-broadcasting.md` |
| 3.5 | Aggregations | `lesson-3.5-aggregations.md` |
| 3.6 | Handling Missing Data (`NaN`) | `lesson-3.6-nan-handling.md` |
| 3.7 | Random Sampling | `lesson-3.7-random-sampling.md` |

## Setup

This module requires NumPy installed in your virtual environment (see Module 0):

```bash
pip install numpy
```

Every lesson assumes this import at the top of your script:
```python
import numpy as np
```

## Lesson Format

Same six-part structure as prior modules: CONCEPT → WHY IT MATTERS → SYNTAX REFERENCE → WORKED EXAMPLE → COMMON MISTAKES → PRACTICE EXERCISES, with hidden hints/solutions and a Data Engineer Tip per lesson.

**Data Engineer Tip (module-level):** Every pandas operation you'll learn in Modules 4–5 is, underneath, a NumPy operation on the array backing each DataFrame column. When pandas behaves unexpectedly — a slow `.apply()`, a surprising `NaN`, an unexpected data type — the explanation is almost always found at the NumPy level. This module is what makes pandas *make sense* later, rather than feeling like memorized syntax.

---

**Previous module:** [Module 2 — Core Data Structures & File Handling](../module-2-data-structures-and-files/README.md)
**Next module:** [Module 4 — Pandas I: Data Wrangling](../module-4-pandas-1-data-wrangling/README.md)
