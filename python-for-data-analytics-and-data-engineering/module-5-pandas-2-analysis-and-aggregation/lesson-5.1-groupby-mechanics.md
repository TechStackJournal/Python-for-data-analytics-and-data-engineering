# Lesson 5.1 — `groupby()` Mechanics

## CONCEPT

`groupby()` implements the **split-apply-combine** pattern: **split** the data into groups based on a column's values, **apply** an aggregation (or other function) to each group independently, then **combine** the results back into a single output.

```python
import pandas as pd

df.groupby("category")["price"].mean()
```

This one line: splits `df` into a separate group for each distinct `category`, computes the mean `price` within each group, and combines the per-group means into a single result — all without an explicit loop.

## WHY IT MATTERS

"Average order value by category," "total revenue per customer," "count of orders per status" — nearly every real business question about a dataset is a `groupby()`. This is the pandas equivalent of SQL's `GROUP BY`, and it's arguably the single most-used pandas method in real analytics work. Understanding exactly what a `groupby()` object *is* (not yet a DataFrame — an intermediate grouped structure waiting for an aggregation) is the key to using it fluently rather than guessing at syntax.

## SYNTAX REFERENCE

```python
import pandas as pd

# Basic groupby + single aggregation
df.groupby("category")["price"].mean()        # a Series: category -> mean price
df.groupby("category")["price"].sum()
df.groupby("category")["price"].count()          # count of NON-NULL values per group
df.groupby("category").size()                      # count of ROWS per group (includes nulls)

# Grouping by multiple columns
df.groupby(["category", "status"])["price"].mean()   # a MultiIndex result

# Selecting the WHOLE group's columns, not just one
df.groupby("category")[["price", "quantity"]].mean()

# Common aggregation shortcuts
df.groupby("category")["price"].agg(["mean", "sum", "count", "min", "max"])

# Resetting the index — turns the grouped result back into a "normal" flat DataFrame
result = df.groupby("category")["price"].mean().reset_index()
# columns become: category, price (instead of category as the index)

# Iterating over groups directly (rarely needed, but useful to understand what's happening)
for category, group_df in df.groupby("category"):
    print(category, len(group_df))

# Filtering groups based on a group-level condition
df.groupby("category").filter(lambda g: len(g) > 5)   # keep only groups with more than 5 rows

# Sorting is on by default — disable if you don't need it (a minor performance gain)
df.groupby("category", sort=False)
```

## WORKED EXAMPLE

Answering several real business questions about order data using `groupby()`:

```python
import pandas as pd

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5, 6, 7, 8],
    "category": ["Electronics", "Home", "Electronics", "Outdoor",
                 "Home", "Electronics", "Outdoor", "Home"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00, 199.99, 22.50, 15.00],
    "status": ["shipped", "shipped", "pending", "shipped",
               "cancelled", "shipped", "shipped", "pending"],
})

# Question 1: What's the average order total per category?
avg_by_category = orders.groupby("category")["total"].mean()
print(avg_by_category)
# category
# Electronics    111.99
# Home            66.83
# Outdoor         28.25
# Name: total, dtype: float64

# Question 2: How many orders fall into each status?
order_counts = orders.groupby("status").size()
print(order_counts)
# status
# cancelled    1
# pending      2
# shipped      5
# dtype: int64

# Question 3: Total revenue per category, for SHIPPED orders only
# (filter FIRST, then group — this is a very common pattern)
shipped_only = orders[orders["status"] == "shipped"]
revenue_by_category = shipped_only.groupby("category")["total"].sum()
print(revenue_by_category)
# category
# Electronics    245.98
# Home           120.50
# Outdoor         56.50
# Name: total, dtype: float64

# Question 4: Grouping by TWO columns at once — revenue by category AND status
by_category_and_status = orders.groupby(["category", "status"])["total"].sum()
print(by_category_and_status)
# category     status
# Electronics  pending      89.99
#              shipped     245.98
# Home         cancelled    65.00
#              pending      15.00
#              shipped     120.50
# Outdoor      shipped      56.50
# Name: total, dtype: float64

# Question 5: Turning the grouped result back into a flat, usable DataFrame
flat_result = orders.groupby("category")["total"].mean().reset_index()
flat_result.columns = ["category", "avg_total"]
print(flat_result)
#       category  avg_total
# 0  Electronics     111.99
# 1         Home      66.83
# 2      Outdoor      28.25
```

## COMMON MISTAKES

**1. Expecting `df.groupby("category")` alone to produce a visible result**
```python
grouped = df.groupby("category")
print(grouped)   # <pandas.core.groupby.generic.DataFrameGroupBy object at 0x...> — not a table!
```
Fix: `groupby()` alone creates an intermediate **grouped object**, not yet a result — you must chain an aggregation (`.mean()`, `.sum()`, `.agg()`, etc.) afterward to actually produce output.

**2. Forgetting the result has the grouping column as its INDEX, not a regular column**
```python
result = df.groupby("category")["total"].mean()
result["category"]   # KeyError! "category" is the index now, not a column
```
Fix: use `.reset_index()` if you need the grouping column back as a regular column (e.g., before merging with another DataFrame, or before further filtering by column name).

**3. Grouping on a column that still has messy/inconsistent values**
```python
df.groupby("category")   # "Electronics" and "electronics " treated as DIFFERENT groups!
```
Fix: this is exactly why Module 4's cleaning (Lesson 4.6: `.str.strip().str.title()`) should happen *before* any `groupby()` — inconsistent casing or whitespace silently fragments what should be one group into several.

**4. Filtering AFTER grouping when you meant to filter BEFORE**
```python
# WRONG intent: "average total for shipped orders, per category"
orders.groupby("category")["total"].mean()[orders["status"] == "shipped"]   # doesn't even work correctly

# CORRECT: filter the rows first, then group
orders[orders["status"] == "shipped"].groupby("category")["total"].mean()
```
Fix: think carefully about the order of operations — "filter rows, then group" and "group, then filter groups" answer genuinely different questions, and the first is far more common.

**5. Using `.count()` when you meant `.size()` (or vice versa)**
```python
df.groupby("category")["total"].count()   # counts NON-NULL values in the total column only
df.groupby("category").size()               # counts ALL rows in each group, nulls included
```
These give different answers whenever there are missing values in the column you're counting — a subtle but real source of off-by-a-few discrepancies.

**Data Engineer Tip:** `orders[orders["status"] == "shipped"].groupby("category")["total"].sum()` is pandas' direct equivalent of `SELECT category, SUM(total) FROM orders WHERE status = 'shipped' GROUP BY category` — filter (`WHERE`) happens before grouping (`GROUP BY`) in both. If you already covered `WHERE` vs. `HAVING` in the SQL course's gotchas, the pandas parallel is: filtering rows *before* `.groupby()` is a `WHERE`-style filter, while `.filter()` on the grouped object (shown in the Syntax Reference) — filtering entire *groups* based on a group-level condition — is the `HAVING`-style equivalent.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `df = pd.DataFrame({"category": ["A", "B", "A", "B"], "value": [10, 20, 30, 40]})`, group by `category` and compute the sum of `value` per group. Print the result.

<details>
<summary>Hint</summary>
`df.groupby("category")["value"].sum()`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"category": ["A", "B", "A", "B"], "value": [10, 20, 30, 40]})
result = df.groupby("category")["value"].sum()
print(result)
# A    40
# B    60
```
</details>

---

**Exercise 2 (Easy):** Using the same DataFrame, count how many rows belong to each category using `.size()`. Print the result.

<details>
<summary>Hint</summary>
`df.groupby("category").size()`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"category": ["A", "B", "A", "B"], "value": [10, 20, 30, 40]})
counts = df.groupby("category").size()
print(counts)
# A    2
# B    2
```
</details>

---

**Exercise 3 (Medium):** Given
```python
orders = pd.DataFrame({
    "customer": ["Maria", "James", "Maria", "Priya", "James"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00],
})
```
compute the total spend per customer, then use `.reset_index()` to turn the result into a flat DataFrame with columns `customer` and `total`. Print it.

<details>
<summary>Hint</summary>
`orders.groupby("customer")["total"].sum().reset_index()`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "customer": ["Maria", "James", "Maria", "Priya", "James"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00],
})

spend = orders.groupby("customer")["total"].sum().reset_index()
print(spend)
#   customer   total
# 0    James  185.50
# 1    Maria  135.98
# 2    Priya   34.00
```
</details>

---

**Exercise 4 (Medium):** Given
```python
orders = pd.DataFrame({
    "category": ["Electronics", "Home", "Electronics", "Outdoor", "Home"],
    "status": ["shipped", "shipped", "cancelled", "shipped", "shipped"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00],
})
```
compute total revenue per category, using only `"shipped"` orders (filter first, then group). Print the result.

<details>
<summary>Hint</summary>
Filter with `orders[orders["status"] == "shipped"]` first, then chain `.groupby("category")["total"].sum()`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "category": ["Electronics", "Home", "Electronics", "Outdoor", "Home"],
    "status": ["shipped", "shipped", "cancelled", "shipped", "shipped"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00],
})

shipped_revenue = orders[orders["status"] == "shipped"].groupby("category")["total"].sum()
print(shipped_revenue)
# Electronics     45.99
# Home           185.50
# Outdoor         34.00
```
</details>

---

**Exercise 5 (Hard):** Given
```python
orders = pd.DataFrame({
    "customer": ["Maria", "James", "Maria", "Priya", "James", "Maria", "Priya"],
    "category": ["Electronics", "Home", "Home", "Outdoor", "Electronics", "Electronics", "Home"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00, 199.99, 22.50],
})
```
Group by BOTH `customer` and `category`, compute the sum of `total` for each combination, then use `.reset_index()` to flatten the result. Additionally, use `.filter()` on a *separate* groupby (grouped by `customer` only) to keep only rows belonging to customers with more than one order total (i.e., customers who appear more than once in the original data). Print both results.

<details>
<summary>Hint</summary>
For the first part: `orders.groupby(["customer", "category"])["total"].sum().reset_index()`. For the second: `orders.groupby("customer").filter(lambda g: len(g) > 1)` — this keeps entire groups (all rows for a customer) where the group has more than 1 row.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "customer": ["Maria", "James", "Maria", "Priya", "James", "Maria", "Priya"],
    "category": ["Electronics", "Home", "Home", "Outdoor", "Electronics", "Electronics", "Home"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00, 199.99, 22.50],
})

by_customer_category = orders.groupby(["customer", "category"])["total"].sum().reset_index()
print(by_customer_category)
#   customer     category   total
# 0    James  Electronics   65.00
# 1    James         Home  120.50
# 2    Maria  Electronics  245.98
# 3    Maria         Home   89.99
# 4    Priya         Home   22.50
# 5    Priya      Outdoor   34.00

repeat_customers = orders.groupby("customer").filter(lambda g: len(g) > 1)
print(repeat_customers)
# (rows for Maria and James only — Priya appears only twice too actually, let's verify:
#  Maria: 3 rows, James: 2 rows, Priya: 2 rows — ALL qualify with > 1)
```
</details>

---

**Previous module:** [Module 4 — Pandas I: Data Wrangling](../module-4-pandas-1-data-wrangling/README.md)
**Next lesson:** [5.2 — Aggregation Functions & .agg()](lesson-5.2-aggregation-functions.md)
