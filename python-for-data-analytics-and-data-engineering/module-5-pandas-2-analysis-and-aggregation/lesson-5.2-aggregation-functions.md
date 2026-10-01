# Lesson 5.2 — Aggregation Functions & `.agg()`

## CONCEPT

Lesson 5.1 used single aggregations (`.mean()`, `.sum()`) on a single column. `.agg()` (short for "aggregate") extends this to apply **multiple aggregation functions at once**, and to apply **different aggregations to different columns** in a single call — the pandas equivalent of writing several aggregate functions in one SQL `SELECT` with a `GROUP BY`.

```python
import pandas as pd

df.groupby("category")["price"].agg(["mean", "sum", "count"])
df.groupby("category").agg({"price": "mean", "quantity": "sum"})
```

## WHY IT MATTERS

Real reporting questions rarely need just one number — "average order value, total revenue, AND order count, per category" is a single, extremely common report, not three separate queries. `.agg()` lets you build exactly that in one line, and its flexibility (custom functions, named outputs) covers the vast majority of real summary-table requirements you'll encounter in analytics work.

## SYNTAX REFERENCE

```python
import pandas as pd

# Multiple aggregations on ONE column
df.groupby("category")["price"].agg(["mean", "sum", "count", "min", "max"])

# Different aggregations for DIFFERENT columns
df.groupby("category").agg({
    "price": "mean",
    "quantity": "sum",
    "order_id": "count",
})

# Multiple aggregations per column, using a dict of lists
df.groupby("category").agg({
    "price": ["mean", "min", "max"],
    "quantity": "sum",
})
# result has a MultiIndex column structure: (price, mean), (price, min), etc.

# Named aggregation — cleaner column names, recommended for reports (pandas 0.25+)
df.groupby("category").agg(
    avg_price=("price", "mean"),
    total_quantity=("quantity", "sum"),
    order_count=("order_id", "count"),
)
# result columns are exactly: avg_price, total_quantity, order_count — no MultiIndex

# Custom aggregation function
def price_range(series):
    return series.max() - series.min()

df.groupby("category")["price"].agg(price_range)
df.groupby("category")["price"].agg(lambda s: s.max() - s.min())   # equivalent, inline

# Common built-in aggregation function names (as strings, for use inside .agg())
# 'mean', 'sum', 'count', 'min', 'max', 'std', 'var', 'median', 'nunique', 'first', 'last'
```

## WORKED EXAMPLE

Building a complete category-level summary report — the kind of table that would go directly into a business dashboard:

```python
import pandas as pd

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5, 6, 7, 8],
    "category": ["Electronics", "Home", "Electronics", "Outdoor",
                 "Home", "Electronics", "Outdoor", "Home"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00, 199.99, 22.50, 15.00],
    "quantity": [1, 3, 2, 1, 2, 1, 1, 1],
})

# Named aggregation — the cleanest, most report-ready approach
summary = orders.groupby("category").agg(
    order_count=("order_id", "count"),
    total_revenue=("total", "sum"),
    avg_order_value=("total", "mean"),
    total_items_sold=("quantity", "sum"),
    max_order_value=("total", "max"),
)
print(summary)
#              order_count  total_revenue  avg_order_value  total_items_sold  max_order_value
# category
# Electronics            3         335.97           111.99                 4           199.99
# Home                    3         200.50            66.83                 6           120.50
# Outdoor                 2          56.50            28.25                 2            34.00

# Round the report for cleaner display
summary_rounded = summary.round(2)
print(summary_rounded)

# A custom aggregation: the "spread" (max - min) of order values per category
price_spread = orders.groupby("category")["total"].agg(lambda s: round(s.max() - s.min(), 2))
print(price_spread)
# category
# Electronics    154.00
# Home           105.50
# Outdoor         11.50
# Name: total, dtype: float64

# Flattening a MultiIndex-column result (happens with the dict-of-lists style)
multi_summary = orders.groupby("category").agg({"total": ["mean", "sum"], "quantity": "sum"})
print(multi_summary.columns)
# MultiIndex([( 'total', 'mean'), ( 'total',  'sum'), ('quantity',  'sum')])

# Flatten it into simple, readable column names
multi_summary.columns = ["_".join(col).strip("_") for col in multi_summary.columns]
print(multi_summary.columns.tolist())
# ['total_mean', 'total_sum', 'quantity_sum']
```

## COMMON MISTAKES

**1. Preferring the dict-of-lists style over named aggregation, then fighting the resulting MultiIndex columns**
```python
result = df.groupby("category").agg({"price": ["mean", "sum"]})
result["price"]   # this works, but result["mean"] does NOT — the structure is nested
```
Fix: for anything destined for a report or further use, prefer **named aggregation** (`avg_price=("price", "mean")`) — it produces flat, directly usable column names from the start, avoiding the MultiIndex flattening step entirely.

**2. Using a string aggregation name that doesn't apply to the column's dtype**
```python
df.groupby("category")["status"].agg("mean")   # error or nonsense — can't average text!
```
Fix: match the aggregation to the column type — `mean`/`sum`/`std` for numeric columns, `count`/`nunique`/`first`/`last` for any column including text.

**3. Forgetting `.count()` (inside `.agg()` or standalone) excludes missing values, while `size` doesn't**
This is the same trap from Lesson 5.1 — worth repeating here since it's easy to miss when it's buried inside a larger `.agg()` call with several other aggregations that don't have this quirk.

**4. Writing an overly complex custom lambda when a built-in string name already does the job**
```python
df.groupby("category")["price"].agg(lambda s: s.sum())   # works, but...
df.groupby("category")["price"].agg("sum")                  # ...simpler and slightly faster
```
Fix: reach for a custom function only when there's genuinely no built-in equivalent (like the `price_range` example) — built-in aggregation names are implemented more efficiently.

**5. Applying `.round()` before aggregating instead of after**
```python
df["price"].round(2).groupby(df["category"]).mean()   # rounds individual prices FIRST — usually not intended
df.groupby("category")["price"].mean().round(2)          # rounds the final AGGREGATED result — usually correct
```
Fix: think about whether you want to round raw input values (rarely right) or the final summary numbers (almost always right, purely for display).

**Data Engineer Tip:** The named-aggregation summary in the worked example is exactly equivalent to `SELECT category, COUNT(order_id) AS order_count, SUM(total) AS total_revenue, AVG(total) AS avg_order_value FROM orders GROUP BY category` in SQL — same logic, same resulting column names even, just different syntax. When you're deciding where this aggregation should actually run in a real pipeline (see the module-level Data Engineer Tip), a report like this one is often a good candidate to push upstream into SQL if the source data already lives in a database — it reduces the amount of raw data that needs to travel into Python at all.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `df = pd.DataFrame({"category": ["A", "B", "A", "B"], "value": [10, 20, 30, 40]})`, use `.agg()` to compute both the sum and mean of `value` per category in one call. Print the result.

<details>
<summary>Hint</summary>
`df.groupby("category")["value"].agg(["sum", "mean"])`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"category": ["A", "B", "A", "B"], "value": [10, 20, 30, 40]})
result = df.groupby("category")["value"].agg(["sum", "mean"])
print(result)
#            sum  mean
# category
# A           40    20
# B           60    30
```
</details>

---

**Exercise 2 (Easy):** Using the same DataFrame, use **named aggregation** to produce columns `total_value` (sum) and `avg_value` (mean). Print the result.

<details>
<summary>Hint</summary>
`df.groupby("category").agg(total_value=("value", "sum"), avg_value=("value", "mean"))`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"category": ["A", "B", "A", "B"], "value": [10, 20, 30, 40]})
result = df.groupby("category").agg(
    total_value=("value", "sum"),
    avg_value=("value", "mean"),
)
print(result)
#           total_value  avg_value
# category
# A                  40         20
# B                  60         30
```
</details>

---

**Exercise 3 (Medium):** Given
```python
orders = pd.DataFrame({
    "customer": ["Maria", "James", "Maria", "Priya", "James"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00],
    "order_id": [1, 2, 3, 4, 5],
})
```
build a named-aggregation summary per customer with `order_count` (count of order_id), `total_spent` (sum of total), and `avg_order` (mean of total). Print the result, rounded to 2 decimals.

<details>
<summary>Hint</summary>
Chain `.round(2)` after the `.agg()` call.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "customer": ["Maria", "James", "Maria", "Priya", "James"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00],
    "order_id": [1, 2, 3, 4, 5],
})

summary = orders.groupby("customer").agg(
    order_count=("order_id", "count"),
    total_spent=("total", "sum"),
    avg_order=("total", "mean"),
).round(2)

print(summary)
```
</details>

---

**Exercise 4 (Medium):** Given the same `orders` DataFrame, write a custom aggregation function (or lambda) that computes the range (max - min) of `total` per customer, and apply it with `.agg()`. Print the result.

<details>
<summary>Hint</summary>
`orders.groupby("customer")["total"].agg(lambda s: s.max() - s.min())`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "customer": ["Maria", "James", "Maria", "Priya", "James"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00],
    "order_id": [1, 2, 3, 4, 5],
})

spread = orders.groupby("customer")["total"].agg(lambda s: s.max() - s.min())
print(spread)
```
</details>

---

**Exercise 5 (Hard):** Given
```python
orders = pd.DataFrame({
    "category": ["Electronics", "Home", "Electronics", "Outdoor", "Home", "Electronics"],
    "status": ["shipped", "shipped", "cancelled", "shipped", "pending", "shipped"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00, 199.99],
    "quantity": [1, 3, 2, 1, 2, 1],
})
```
Build a single named-aggregation report per category with: `order_count` (total rows), `shipped_count` (hint: you'll need a helper column or a lambda that counts where status == "shipped" within each group), `total_revenue` (sum of total, ALL statuses), and `avg_items_per_order` (mean of quantity). Print the final report, rounded to 2 decimals where relevant.

<details>
<summary>Hint</summary>
For `shipped_count`, one clean approach: create a boolean helper column first (`orders["is_shipped"] = orders["status"] == "shipped"`), then aggregate it with `"sum"` (since `True` counts as 1) inside the same named-aggregation call — `shipped_count=("is_shipped", "sum")`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "category": ["Electronics", "Home", "Electronics", "Outdoor", "Home", "Electronics"],
    "status": ["shipped", "shipped", "cancelled", "shipped", "pending", "shipped"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00, 199.99],
    "quantity": [1, 3, 2, 1, 2, 1],
})

orders["is_shipped"] = orders["status"] == "shipped"

report = orders.groupby("category").agg(
    order_count=("category", "count"),
    shipped_count=("is_shipped", "sum"),
    total_revenue=("total", "sum"),
    avg_items_per_order=("quantity", "mean"),
).round(2)

print(report)
#              order_count  shipped_count  total_revenue  avg_items_per_order
# category
# Electronics            3              2         335.97                 1.33
# Home                    2              1         185.50                 2.50
# Outdoor                 1              1          34.00                 1.00
```
</details>

---

**Previous lesson:** [5.1 — groupby() Mechanics](lesson-5.1-groupby-mechanics.md)
**Next lesson:** [5.3 — Pivot Tables & crosstab()](lesson-5.3-pivot-tables.md)
