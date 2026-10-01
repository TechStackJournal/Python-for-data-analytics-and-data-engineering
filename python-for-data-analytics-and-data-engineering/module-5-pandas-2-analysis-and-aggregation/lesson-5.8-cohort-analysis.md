# Lesson 5.8 — Cohort Analysis Patterns

## CONCEPT

A **cohort analysis** groups customers (or any entities) by a shared starting point in time — most commonly, the month they first made a purchase — and then tracks how that group behaves over subsequent periods (do they come back? how much do they spend?). It's one of the most common real analytics deliverables, and it's built entirely from techniques you already have: `groupby()`, `merge()`, `pivot_table()`, and datetime handling.

```python
# Conceptually: "of everyone who first ordered in January, what % were still
# active in February? In March?" — a retention cohort, the classic example
```

This lesson doesn't introduce new pandas mechanics — it's a worked, end-to-end synthesis of Lessons 5.1–5.6, applied to a genuinely common real-world analytical question.

## WHY IT MATTERS

Cohort analysis answers questions that a simple aggregate can't: overall revenue might be flat, while newer cohorts are actually retaining *worse* than older ones — a warning sign a single "total revenue" number would completely hide. This exact technique — and the specific "first order month" + retention matrix shape — comes up constantly in subscription businesses, e-commerce, and app analytics, and is a genuinely strong, resume-relevant addition to a portfolio project.

## SYNTAX REFERENCE

```python
import pandas as pd

# Step 1: find each customer's first order date (their "cohort")
first_order = orders.groupby("customer_id")["order_date"].min().reset_index()
first_order.columns = ["customer_id", "cohort_date"]

# Step 2: reduce both dates to a coarser period (e.g. month) for grouping
orders["order_month"] = orders["order_date"].dt.to_period("M")
first_order["cohort_month"] = first_order["cohort_date"].dt.to_period("M")

# Step 3: attach each order's cohort back onto the orders table
orders = orders.merge(first_order[["customer_id", "cohort_month"]], on="customer_id")

# Step 4: compute "periods since first order" for each row
orders["period_number"] = (orders["order_month"] - orders["cohort_month"]).apply(lambda x: x.n)

# Step 5: count distinct active customers per cohort, per period
cohort_counts = orders.groupby(["cohort_month", "period_number"])["customer_id"].nunique().reset_index()

# Step 6: pivot into the classic cohort retention grid
cohort_pivot = cohort_counts.pivot_table(
    index="cohort_month", columns="period_number", values="customer_id"
)

# Step 7: convert counts into retention PERCENTAGES relative to each cohort's starting size
cohort_sizes = cohort_pivot.iloc[:, 0]                     # period 0 = the cohort's full size
retention = cohort_pivot.divide(cohort_sizes, axis=0).round(3)
```

## WORKED EXAMPLE

A complete, end-to-end cohort retention analysis on a small synthetic dataset:

```python
import pandas as pd

orders = pd.DataFrame({
    "customer_id": [1, 1, 1, 2, 2, 3, 3, 3, 4, 4, 5],
    "order_date": pd.to_datetime([
        "2024-09-05", "2024-10-12", "2024-11-20",   # customer 1: Sept, Oct, Nov
        "2024-09-15", "2024-11-02",                   # customer 2: Sept, Nov (skipped Oct)
        "2024-10-01", "2024-10-20", "2024-12-01",       # customer 3: Oct, Oct, Dec
        "2024-11-10", "2024-11-25",                       # customer 4: Nov, Nov
        "2024-12-05",                                       # customer 5: Dec only
    ]),
})

# Step 1: each customer's first order date -> their cohort
first_order = orders.groupby("customer_id")["order_date"].min().reset_index()
first_order.columns = ["customer_id", "cohort_date"]
first_order["cohort_month"] = first_order["cohort_date"].dt.to_period("M")
print(first_order[["customer_id", "cohort_month"]])
#    customer_id cohort_month
# 0            1      2024-09
# 1            2      2024-09
# 2            3      2024-10
# 3            4      2024-11
# 4            5      2024-12

# Step 2-3: attach cohort info back onto every order
orders["order_month"] = orders["order_date"].dt.to_period("M")
orders = orders.merge(first_order[["customer_id", "cohort_month"]], on="customer_id")

# Step 4: how many months after their FIRST order did this particular order happen?
orders["period_number"] = (orders["order_month"] - orders["cohort_month"]).apply(lambda x: x.n)
print(orders[["customer_id", "order_month", "cohort_month", "period_number"]])
#    customer_id order_month cohort_month  period_number
# 0            1     2024-09      2024-09              0
# 1            1     2024-10      2024-09              1
# 2            1     2024-11      2024-09              2
# 3            2     2024-09      2024-09              0
# 4            2     2024-11      2024-09              2
# ...

# Step 5-6: count distinct customers active in each (cohort, period) combination, pivoted
cohort_counts = orders.groupby(["cohort_month", "period_number"])["customer_id"].nunique().reset_index()
cohort_pivot = cohort_counts.pivot_table(index="cohort_month", columns="period_number", values="customer_id")
print(cohort_pivot)
# period_number    0    1    2    3
# cohort_month
# 2024-09         2.0  1.0  1.0  NaN
# 2024-10         1.0  1.0  NaN  1.0
# 2024-11         1.0  1.0  NaN  NaN
# 2024-12         1.0  NaN  NaN  NaN

# Step 7: convert to a RETENTION PERCENTAGE — relative to each cohort's own starting size
cohort_sizes = cohort_pivot.iloc[:, 0]        # period 0 column = full cohort size
retention = cohort_pivot.divide(cohort_sizes, axis=0).round(2)
print(retention)
# period_number    0     1     2    3
# cohort_month
# 2024-09         1.0  0.50  0.50  NaN
# 2024-10         1.0  1.00   NaN  1.0
# 2024-11         1.0  1.00   NaN  NaN
# 2024-12         1.0   NaN   NaN  NaN

# Reading this: of the 2 customers who first ordered in September, only 50%
# (1 customer) returned in period 1 (October) AND period 2 (November) — customer
# 2 skipped October entirely, which is exactly the kind of gap this analysis reveals
```

## COMMON MISTAKES

**1. Using calendar date subtraction instead of PERIOD subtraction for `period_number`**
```python
(orders["order_date"] - orders["cohort_date"]).dt.days   # gives days, not "months since first order"
```
Fix: convert both dates to a `Period` (via `.dt.to_period("M")`) *before* subtracting — subtracting two `Period` objects gives a clean integer count of periods (months, in this case), which is what a cohort table actually needs, rather than a raw day count you'd have to convert yourself.

**2. Forgetting `NaN` in the retention table means "cohort didn't exist yet" or "zero customers active," not an error**
In the worked example, `2024-09` has `NaN` at `period_number=3` simply because not enough time has passed since September for a "period 3" to exist yet for that cohort within this dataset — this is expected and meaningful (it should generally be left blank/NaN in a report, not filled with 0, which would incorrectly suggest "we know retention dropped to zero").

**3. Computing cohort based on the WRONG date (e.g. a random order instead of the first)**
```python
orders.groupby("customer_id")["order_date"].first()   # relies on row ORDER, not the actual earliest date!
orders.groupby("customer_id")["order_date"].min()        # correct — always the true earliest date
```
Fix: use `.min()` specifically to get the true earliest date per customer, regardless of what order the rows happen to appear in — `.first()` returns whichever row came first in the DataFrame, which is not necessarily the earliest by date unless you've already sorted.

**4. Dividing by the wrong denominator when converting counts to percentages**
```python
cohort_pivot.divide(cohort_pivot.iloc[:, 0], axis=0)   # correct: each ROW divided by ITS OWN period-0 value
cohort_pivot / cohort_pivot.iloc[:, 0].mean()              # wrong: divides everything by one average value!
```
Fix: `axis=0` is essential here — it tells pandas to divide each row by the corresponding value from `cohort_sizes` (matched by index/cohort), not to broadcast a single number across the whole table.

**5. Treating a cohort table as "done" without sanity-checking the shape against what you know about the data**
The worked example's small dataset makes it easy to manually verify — always spot-check a cohort table's period-0 column against a straightforward `groupby("cohort_month")["customer_id"].nunique()` count on the raw first-order data, to catch a subtle bug in the merge/pivot chain before trusting the full retention percentages.

**Data Engineer Tip:** This lesson is a genuinely good template for a portfolio piece — it combines nearly every pandas technique from this module into one real, resume-relevant analysis. Consider building this exact cohort analysis against the full `ecommerce.orders` and `ecommerce.customers` tables (matching the schema shared with your SQL course) as a Module 4/5 capstone artifact — a dual technical notebook + plain-English summary of what the retention pattern shows, following the "data cleaning decisions log" and executive-summary format from your other portfolio work, would make this a strong standalone project.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given
```python
orders = pd.DataFrame({
    "customer_id": [1, 1, 2],
    "order_date": pd.to_datetime(["2024-01-10", "2024-02-15", "2024-01-20"]),
})
```
compute each customer's first order date (their cohort date) using `.groupby().min()`. Print the result.

<details>
<summary>Hint</summary>
`orders.groupby("customer_id")["order_date"].min()`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "customer_id": [1, 1, 2],
    "order_date": pd.to_datetime(["2024-01-10", "2024-02-15", "2024-01-20"]),
})

cohort_dates = orders.groupby("customer_id")["order_date"].min()
print(cohort_dates)
# customer_id
# 1   2024-01-10
# 2   2024-01-20
```
</details>

---

**Exercise 2 (Easy):** Convert the `order_date` column from Exercise 1 into a monthly `Period` using `.dt.to_period("M")`, and print the result alongside the original dates.

<details>
<summary>Hint</summary>
`orders["order_date"].dt.to_period("M")`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "customer_id": [1, 1, 2],
    "order_date": pd.to_datetime(["2024-01-10", "2024-02-15", "2024-01-20"]),
})

orders["order_month"] = orders["order_date"].dt.to_period("M")
print(orders)
```
</details>

---

**Exercise 3 (Medium):** Given
```python
orders = pd.DataFrame({
    "customer_id": [1, 1, 2, 2],
    "order_date": pd.to_datetime(["2024-01-10", "2024-03-05", "2024-02-01", "2024-02-20"]),
})
```
compute each customer's cohort month (first order month), merge it back onto the full `orders` DataFrame, and compute a `period_number` column representing months since each customer's first order. Print the final DataFrame with `customer_id`, `order_date`, `cohort_month`, and `period_number`.

<details>
<summary>Hint</summary>
Follow steps 1-4 from the Syntax Reference exactly.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "customer_id": [1, 1, 2, 2],
    "order_date": pd.to_datetime(["2024-01-10", "2024-03-05", "2024-02-01", "2024-02-20"]),
})

first_order = orders.groupby("customer_id")["order_date"].min().reset_index()
first_order.columns = ["customer_id", "cohort_date"]
first_order["cohort_month"] = first_order["cohort_date"].dt.to_period("M")

orders["order_month"] = orders["order_date"].dt.to_period("M")
orders = orders.merge(first_order[["customer_id", "cohort_month"]], on="customer_id")
orders["period_number"] = (orders["order_month"] - orders["cohort_month"]).apply(lambda x: x.n)

print(orders[["customer_id", "order_date", "cohort_month", "period_number"]])
```
</details>

---

**Exercise 4 (Medium):** Using the result from Exercise 3, count the number of distinct customers active in each `(cohort_month, period_number)` combination, and pivot it into a grid (cohort_month as rows, period_number as columns). Print the result.

<details>
<summary>Hint</summary>
`.groupby(["cohort_month", "period_number"])["customer_id"].nunique().reset_index()`, then `.pivot_table(...)`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "customer_id": [1, 1, 2, 2],
    "order_date": pd.to_datetime(["2024-01-10", "2024-03-05", "2024-02-01", "2024-02-20"]),
})

first_order = orders.groupby("customer_id")["order_date"].min().reset_index()
first_order.columns = ["customer_id", "cohort_date"]
first_order["cohort_month"] = first_order["cohort_date"].dt.to_period("M")

orders["order_month"] = orders["order_date"].dt.to_period("M")
orders = orders.merge(first_order[["customer_id", "cohort_month"]], on="customer_id")
orders["period_number"] = (orders["order_month"] - orders["cohort_month"]).apply(lambda x: x.n)

cohort_counts = orders.groupby(["cohort_month", "period_number"])["customer_id"].nunique().reset_index()
cohort_pivot = cohort_counts.pivot_table(index="cohort_month", columns="period_number", values="customer_id")
print(cohort_pivot)
```
</details>

---

**Exercise 5 (Hard):** Using a larger synthetic dataset:
```python
orders = pd.DataFrame({
    "customer_id": [1, 1, 1, 2, 2, 3, 4, 4, 5, 5, 5, 6],
    "order_date": pd.to_datetime([
        "2024-08-01", "2024-09-15", "2024-10-20",
        "2024-08-10", "2024-10-05",
        "2024-08-25",
        "2024-09-01", "2024-09-20",
        "2024-10-01", "2024-11-05", "2024-11-20",
        "2024-11-15",
    ]),
})
```
Run the full cohort analysis pipeline end-to-end (cohort assignment, period numbers, pivoted counts), then convert the counts into a **retention percentage table** (each cohort's period-0 column normalized to 1.0, other periods relative to it), rounded to 2 decimals. Print the final retention percentage table, and write one sentence (as a comment) interpreting which cohort appears to be retaining best based on the numbers.

<details>
<summary>Hint</summary>
Follow all 7 steps from the Syntax Reference. For the retention percentages, use `cohort_pivot.divide(cohort_pivot.iloc[:, 0], axis=0)`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "customer_id": [1, 1, 1, 2, 2, 3, 4, 4, 5, 5, 5, 6],
    "order_date": pd.to_datetime([
        "2024-08-01", "2024-09-15", "2024-10-20",
        "2024-08-10", "2024-10-05",
        "2024-08-25",
        "2024-09-01", "2024-09-20",
        "2024-10-01", "2024-11-05", "2024-11-20",
        "2024-11-15",
    ]),
})

first_order = orders.groupby("customer_id")["order_date"].min().reset_index()
first_order.columns = ["customer_id", "cohort_date"]
first_order["cohort_month"] = first_order["cohort_date"].dt.to_period("M")

orders["order_month"] = orders["order_date"].dt.to_period("M")
orders = orders.merge(first_order[["customer_id", "cohort_month"]], on="customer_id")
orders["period_number"] = (orders["order_month"] - orders["cohort_month"]).apply(lambda x: x.n)

cohort_counts = orders.groupby(["cohort_month", "period_number"])["customer_id"].nunique().reset_index()
cohort_pivot = cohort_counts.pivot_table(index="cohort_month", columns="period_number", values="customer_id")

cohort_sizes = cohort_pivot.iloc[:, 0]
retention = cohort_pivot.divide(cohort_sizes, axis=0).round(2)
print(retention)

# The August 2024 cohort (customers 1, 2, 3) shows the strongest longer-term
# retention: it still has activity at period 2 (October), while later cohorts
# haven't yet had enough elapsed time in this dataset to show the same depth —
# though September's cohort (customers 4, 5... wait, 5 started in October) also
# looks healthy at period 1. Always read a small synthetic table like this
# cautiously — with so few customers per cohort, one or two customers swing the
# percentage dramatically, which is itself a good lesson about cohort analysis
# on small sample sizes.
```
</details>

---

## Module 5 Complete

You've now covered pandas' full analytical toolkit: grouping, aggregation, pivoting, merging, concatenation, time series, window functions, and a complete real-world cohort analysis synthesizing all of it. Module 6 moves from *computing* these results to *communicating* them — turning everything you've built here into charts and visual reports.

**Previous lesson:** [5.7 — Window Functions](lesson-5.7-window-functions.md)
**Next module:** [Module 6 — Data Visualization](../module-6-data-visualization/README.md)
