# Lesson 5.3 — Pivot Tables & `crosstab()`

## CONCEPT

A **pivot table** reshapes long-format data (one row per observation) into a wide grid — one value spread across rows, another across columns, with an aggregation filling each cell. `pivot_table()` is pandas' tool for this; `crosstab()` is a closely related, simpler tool specifically for counting combinations of two categorical variables.

```python
import pandas as pd

pd.pivot_table(df, values="total", index="category", columns="status", aggfunc="sum")
pd.crosstab(df["category"], df["status"])   # counts, by default
```

If you've used a pivot table in Excel or Google Sheets, this is the exact same concept — pandas' version is just programmatic rather than click-driven.

## WHY IT MATTERS

Many real reports are naturally two-dimensional: "revenue by category, broken down by month," "order count by status, broken down by region." A `groupby(["category", "month"])` (Lesson 5.1) technically answers this, but the result is a tall, narrow table — a pivot table reshapes it into the wide grid format that's actually readable as a report and matches what a stakeholder expects to see (rows = one dimension, columns = another, values in the grid).

## SYNTAX REFERENCE

```python
import pandas as pd

# Basic pivot table
pd.pivot_table(
    df,
    values="total",       # the column to aggregate
    index="category",      # becomes the ROWS
    columns="status",       # becomes the COLUMNS
    aggfunc="sum",            # how to aggregate (default is 'mean')
)

# Multiple values / multiple aggregations
pd.pivot_table(df, values=["total", "quantity"], index="category", aggfunc="sum")
pd.pivot_table(df, values="total", index="category", aggfunc=["sum", "mean"])

# Filling missing combinations (where no data exists for a cell) with a specific value
pd.pivot_table(df, values="total", index="category", columns="status",
                aggfunc="sum", fill_value=0)

# Adding row/column totals
pd.pivot_table(df, values="total", index="category", columns="status",
                aggfunc="sum", margins=True, margins_name="Total")

# Multiple index levels (nested rows)
pd.pivot_table(df, values="total", index=["category", "region"], aggfunc="sum")

# crosstab — specifically for counting combinations of two categorical columns
pd.crosstab(df["category"], df["status"])                  # counts of each combination
pd.crosstab(df["category"], df["status"], normalize=True)    # proportions instead of counts
pd.crosstab(df["category"], df["status"], margins=True)        # with row/column totals

# The equivalent groupby()-based approach, for comparison (Lesson 5.1/5.2 style)
df.groupby(["category", "status"])["total"].sum().unstack()
# .unstack() converts the innermost grouped index level into columns — this is
# essentially what pivot_table() does automatically under the hood
```

## WORKED EXAMPLE

Building a revenue-by-category-and-status report, and a separate order-count crosstab, from the same order data:

```python
import pandas as pd

orders = pd.DataFrame({
    "category": ["Electronics", "Home", "Electronics", "Outdoor",
                 "Home", "Electronics", "Outdoor", "Home"],
    "status": ["shipped", "shipped", "pending", "shipped",
               "cancelled", "shipped", "shipped", "pending"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00, 199.99, 22.50, 15.00],
})

# Revenue by category (rows) and status (columns), with missing combos filled as 0
revenue_pivot = pd.pivot_table(
    orders,
    values="total",
    index="category",
    columns="status",
    aggfunc="sum",
    fill_value=0,
)
print(revenue_pivot)
# status       cancelled  pending  shipped
# category
# Electronics       0.00    89.99   245.98
# Home              65.00    15.00   120.50
# Outdoor            0.00     0.00    56.50

# Add row and column totals — useful for a stakeholder-facing report
revenue_pivot_with_totals = pd.pivot_table(
    orders,
    values="total",
    index="category",
    columns="status",
    aggfunc="sum",
    fill_value=0,
    margins=True,
    margins_name="Total",
)
print(revenue_pivot_with_totals)
# status       cancelled  pending  shipped   Total
# category
# Electronics       0.00    89.99   245.98  335.97
# Home              65.00    15.00   120.50  200.50
# Outdoor            0.00     0.00    56.50   56.50
# Total             65.00   104.99   422.98  593.47

# crosstab — a simpler question: how many orders (just COUNTS) fall into each combination?
order_counts = pd.crosstab(orders["category"], orders["status"])
print(order_counts)
# status       cancelled  pending  shipped
# category
# Electronics          0        1        2
# Home                  1        1        1
# Outdoor                0        0        2

# crosstab with normalize — what PROPORTION of all orders does each cell represent?
proportions = pd.crosstab(orders["category"], orders["status"], normalize=True)
print(proportions.round(3))
# status       cancelled  pending  shipped
# category
# Electronics       0.000    0.125    0.250
# Home              0.125    0.125    0.125
# Outdoor            0.000    0.000    0.250
# (every cell in the full table sums to 1.0 across the entire grid)
```

## COMMON MISTAKES

**1. Forgetting `fill_value=0` and getting `NaN` for combinations with no data**
```python
pd.pivot_table(df, values="total", index="category", columns="status", aggfunc="sum")
# a category/status combination with zero matching rows shows NaN, not 0
```
Fix: `fill_value=0` is almost always what you want for a revenue/count-style report — `NaN` here means "no orders in this combination," which is meaningfully different from "zero revenue," but for most reporting purposes, displaying `0` is clearer.

**2. Using the wrong `aggfunc` for the question being asked**
```python
pd.pivot_table(df, values="total", index="category", columns="status")
# default aggfunc is 'mean' — easy to accidentally build an AVERAGE table when you meant a SUM table
```
Fix: always specify `aggfunc` explicitly rather than relying on the default, since `mean` vs `sum` vs `count` produce very different-looking (and easily confused) reports.

**3. Confusing `pivot_table()` (aggregates, handles duplicates) with `pivot()` (reshapes, requires unique index/column combinations)**
```python
df.pivot(index="category", columns="status", values="total")
# ValueError if there's more than one row per category/status combination —
# pivot() does NOT aggregate, it assumes each combination is already unique
```
Fix: `pivot_table()` is the safer default for real data (which usually has multiple rows per combination) — reach for the plain `pivot()` only when you're certain each index/column combination appears exactly once.

**4. Not realizing `crosstab()` takes Series arguments directly, not column name strings**
```python
pd.crosstab(orders, "category", "status")            # wrong — crosstab wants Series, not a DataFrame + names
pd.crosstab(orders["category"], orders["status"])     # correct
```

**5. Misreading `margins=True` totals as being pre-aggregated correctly when the aggfunc is `mean`**
```python
pd.pivot_table(df, values="total", index="category", aggfunc="mean", margins=True)
# the "Total" row/column for a MEAN pivot is the mean of ALL underlying rows,
# NOT a simple average of the category-level means shown above it — this is
# mathematically correct but easy to misread at a glance
```
Fix: be careful interpreting margin totals on anything other than `sum` — for `mean`, the margin is computed from the raw underlying data, not from averaging the displayed group means.

**Data Engineer Tip:** A pivot table is a presentation-layer transformation — it's meant for a human to read a summary, not typically for further programmatic processing (the wide, spread-out shape is awkward to filter or join further). In a real pipeline, aggregation and grouping (Lessons 5.1–5.2) usually happen upstream to produce clean, long-format summary tables that feed a dashboard tool (which then handles the pivoting/visual layout itself) — reach for `pivot_table()`/`crosstab()` specifically when you need a quick, readable summary directly in your analysis, not as a stored intermediate dataset.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given
```python
df = pd.DataFrame({
    "category": ["A", "A", "B", "B"],
    "region": ["East", "West", "East", "West"],
    "sales": [100, 150, 200, 130],
})
```
build a pivot table with `category` as rows, `region` as columns, and `sales` summed in each cell. Print it.

<details>
<summary>Hint</summary>
`pd.pivot_table(df, values="sales", index="category", columns="region", aggfunc="sum")`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({
    "category": ["A", "A", "B", "B"],
    "region": ["East", "West", "East", "West"],
    "sales": [100, 150, 200, 130],
})

pivot = pd.pivot_table(df, values="sales", index="category", columns="region", aggfunc="sum")
print(pivot)
```
</details>

---

**Exercise 2 (Easy):** Using the same DataFrame, build a `crosstab()` showing the count of rows for each `category`/`region` combination.

<details>
<summary>Hint</summary>
`pd.crosstab(df["category"], df["region"])`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({
    "category": ["A", "A", "B", "B"],
    "region": ["East", "West", "East", "West"],
    "sales": [100, 150, 200, 130],
})

counts = pd.crosstab(df["category"], df["region"])
print(counts)
```
</details>

---

**Exercise 3 (Medium):** Given
```python
orders = pd.DataFrame({
    "category": ["Electronics", "Home", "Electronics", "Home", "Outdoor"],
    "status": ["shipped", "pending", "shipped", "shipped", "cancelled"],
    "total": [45.99, 120.50, 89.99, 65.00, 34.00],
})
```
build a pivot table with `category` as rows and `status` as columns, summing `total`, with missing combinations filled as `0`. Print the result.

<details>
<summary>Hint</summary>
Add `fill_value=0` to the `pivot_table()` call.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "category": ["Electronics", "Home", "Electronics", "Home", "Outdoor"],
    "status": ["shipped", "pending", "shipped", "shipped", "cancelled"],
    "total": [45.99, 120.50, 89.99, 65.00, 34.00],
})

pivot = pd.pivot_table(orders, values="total", index="category", columns="status",
                        aggfunc="sum", fill_value=0)
print(pivot)
```
</details>

---

**Exercise 4 (Medium):** Using the same `orders` DataFrame, add row and column totals to the pivot table from Exercise 3 using `margins=True`, labeling the totals `"Grand Total"`. Print the result.

<details>
<summary>Hint</summary>
`margins=True, margins_name="Grand Total"`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "category": ["Electronics", "Home", "Electronics", "Home", "Outdoor"],
    "status": ["shipped", "pending", "shipped", "shipped", "cancelled"],
    "total": [45.99, 120.50, 89.99, 65.00, 34.00],
})

pivot = pd.pivot_table(orders, values="total", index="category", columns="status",
                        aggfunc="sum", fill_value=0, margins=True, margins_name="Grand Total")
print(pivot)
```
</details>

---

**Exercise 5 (Hard):** Given
```python
orders = pd.DataFrame({
    "category": ["Electronics", "Home", "Electronics", "Outdoor", "Home", "Electronics", "Home"],
    "tier": ["gold", "silver", "gold", "bronze", "gold", "silver", "silver"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00, 199.99, 15.00],
})
```
Build a pivot table with `category` as rows and `tier` as columns, showing BOTH the sum and the mean of `total` in each cell (hint: pass a list to `aggfunc`), with missing combinations filled as `0`. Then, separately, use `crosstab()` with `normalize=True` to show what proportion of all orders each category/tier combination represents, rounded to 3 decimals. Print both results.

<details>
<summary>Hint</summary>
For multiple aggfuncs: `aggfunc=["sum", "mean"]` — the result will have a MultiIndex on the columns (tier x aggfunc). For the crosstab: `pd.crosstab(orders["category"], orders["tier"], normalize=True).round(3)`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "category": ["Electronics", "Home", "Electronics", "Outdoor", "Home", "Electronics", "Home"],
    "tier": ["gold", "silver", "gold", "bronze", "gold", "silver", "silver"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00, 199.99, 15.00],
})

pivot = pd.pivot_table(
    orders,
    values="total",
    index="category",
    columns="tier",
    aggfunc=["sum", "mean"],
    fill_value=0,
)
print(pivot)

proportions = pd.crosstab(orders["category"], orders["tier"], normalize=True).round(3)
print(proportions)
```
</details>

---

**Previous lesson:** [5.2 — Aggregation Functions & .agg()](lesson-5.2-aggregation-functions.md)
**Next lesson:** [5.4 — Merging & Joining DataFrames](lesson-5.4-merging-and-joining.md)
