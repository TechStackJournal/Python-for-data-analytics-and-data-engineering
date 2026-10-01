# Lesson 4.8 — Detecting & Handling Duplicates

## CONCEPT

**Duplicate rows** — records that are identical (or identical on the fields that matter) — are extremely common in real data: a customer submitted the same form twice, an ETL job re-ran and re-inserted rows, a join accidentally fanned out. pandas provides direct tools to detect and remove them.

```python
import pandas as pd

df.duplicated()             # a boolean Series — True where a row is a duplicate of an earlier one
df.drop_duplicates()          # returns a new DataFrame with duplicates removed
```

By default, "duplicate" means every column matches exactly — but you can also check duplication on just a subset of columns (e.g., "same customer, same email" even if other fields differ).

## WHY IT MATTERS

Duplicate rows silently distort nearly every downstream calculation: a `sum()` counts a duplicated order twice, a `mean()` skews toward whatever got duplicated, a customer count comes out too high. Unlike a missing value (Lesson 4.5), which is usually *visible* as a gap, a duplicate is invisible unless you specifically check for it — making this one of the most important, and most often skipped, data quality checks in a real pipeline. This directly parallels the `DISTINCT` keyword and `GROUP BY`-based duplicate detection from the SQL course.

## SYNTAX REFERENCE

```python
import pandas as pd

# Detecting duplicates
df.duplicated()                       # True/False per row — is this row a repeat of an earlier one?
df.duplicated().sum()                  # count of duplicate rows
df[df.duplicated()]                     # view just the duplicate rows

# Duplicated based on specific columns only
df.duplicated(subset=["customer_id", "email"])

# Controlling which occurrence is marked as the "duplicate"
df.duplicated(keep="first")    # default — first occurrence is NOT flagged, later ones are
df.duplicated(keep="last")      # last occurrence is NOT flagged, earlier ones are
df.duplicated(keep=False)        # ALL occurrences are flagged, including the first

# Removing duplicates
df.drop_duplicates()                          # keeps the first occurrence of each duplicate set
df.drop_duplicates(subset=["email"])            # dedupe based on email only
df.drop_duplicates(subset=["email"], keep="last")  # keep the LAST occurrence instead
df.drop_duplicates(inplace=True)                  # modify df directly rather than returning a copy

# Counting unique values (a related, very common check)
df["customer_id"].nunique()          # count of DISTINCT values in a column
df["customer_id"].unique()            # array of the distinct values themselves
```

## WORKED EXAMPLE

Detecting and resolving duplicate order records — including the more subtle case of "same order, different timestamp" from a re-run pipeline:

```python
import pandas as pd

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 2, 4, 3],
    "customer_id": [101, 102, 103, 102, 104, 103],
    "total": [45.99, 120.50, 89.99, 120.50, 15.00, 89.99],
    "loaded_at": ["2024-11-01 09:00", "2024-11-01 09:01", "2024-11-01 09:02",
                  "2024-11-01 09:15", "2024-11-01 09:03", "2024-11-01 09:16"],
})
print(orders)
#    order_id  customer_id   total          loaded_at
# 0         1          101   45.99  2024-11-01 09:00
# 1         2          102  120.50  2024-11-01 09:01
# 2         3          103   89.99  2024-11-01 09:02
# 3         2          102  120.50  2024-11-01 09:15
# 4         4          104   15.00  2024-11-01 09:03
# 5         3          103   89.99  2024-11-01 09:16

# 1. Full-row duplicate check — finds nothing, because loaded_at differs for every row!
print("Full-row duplicates:", orders.duplicated().sum())   # 0 — misleading

# 2. The REAL duplicates are on order_id — a re-run pipeline reloaded orders 2 and 3
print("Order ID duplicates:", orders.duplicated(subset=["order_id"]).sum())   # 2

duplicate_rows = orders[orders.duplicated(subset=["order_id"], keep=False)]
print(duplicate_rows)
#    order_id  customer_id   total          loaded_at
# 1         2          102  120.50  2024-11-01 09:01
# 2         3          103   89.99  2024-11-01 09:02
# 3         2          102  120.50  2024-11-01 09:15
# 5         3          103   89.99  2024-11-01 09:16
# (keep=False shows BOTH occurrences of each duplicated order_id, for inspection)

# 3. Deduplicate, keeping the LATEST load for each order_id (a common ETL pattern —
#    trust the most recent version of a record)
orders_sorted = orders.sort_values("loaded_at")
deduped = orders_sorted.drop_duplicates(subset=["order_id"], keep="last")
print(deduped.sort_values("order_id"))
#    order_id  customer_id   total          loaded_at
# 0         1          101   45.99  2024-11-01 09:00
# 3         2          102  120.50  2024-11-01 09:15
# 5         3          103   89.99  2024-11-01 09:16
# 4         4          104   15.00  2024-11-01 09:03

print(f"Original rows: {len(orders)}, After dedup: {len(deduped)}")
# Original rows: 6, After dedup: 4

# 4. A related check: how many DISTINCT customers placed these orders?
print("Distinct customers:", orders["customer_id"].nunique())   # 4
```

## COMMON MISTAKES

**1. Checking for full-row duplicates when the real duplicate key is narrower**
As the worked example shows, a `loaded_at` timestamp (or any metadata column) that differs between an original and its re-inserted copy makes `df.duplicated()` (checking *every* column) find nothing, even though the business-meaningful data (`order_id`) clearly repeats. Fix: always think about what actually defines "the same record" for your specific data — usually a business key (`order_id`, `email`), not every column — and pass that explicitly via `subset=`.

**2. Using `drop_duplicates()` without sorting first, when "which copy to keep" matters**
```python
df.drop_duplicates(subset=["order_id"])   # keeps whichever row happened to come first in the DataFrame
```
Fix: if you specifically want to keep the *newest* or *most complete* version of a duplicated record, sort deliberately first (by a timestamp, a "last updated" column, or completeness) so `keep="first"`/`"last"` selects the version you actually intend.

**3. Forgetting `drop_duplicates()` returns a new DataFrame by default**
```python
df.drop_duplicates()     # df itself is UNCHANGED unless you reassign or use inplace=True
print(df.duplicated().sum())   # still shows the original duplicate count!
```
Fix: `df = df.drop_duplicates()` to reassign, or `df.drop_duplicates(inplace=True)` to modify directly — the same pattern as most pandas cleaning methods (Lesson 4.5, 4.6).

**4. Confusing `.duplicated()` (which row is a REPEAT) with `.nunique()`/`.unique()` (how many DISTINCT values)**
These answer related but different questions: `.duplicated()` operates at the row level and tells you *which* rows repeat; `.nunique()` operates on a single column/Series and tells you *how many distinct values* exist, regardless of which rows they're in.

**5. Not realizing near-duplicates (different only by whitespace/casing) won't be caught**
```python
pd.Series(["maria@example.com", "Maria@Example.com "]).duplicated()   # [False, False] — NOT caught!
```
Fix: this is exactly why Lesson 4.6's cleaning (stripping whitespace, standardizing case) should generally happen *before* duplicate detection — `.duplicated()` only catches values that are already byte-for-byte identical.

**Data Engineer Tip:** The "reload with a newer timestamp, then dedupe keeping the latest" pattern in the worked example is exactly how many real pipelines handle **idempotent loads** — a job that can safely re-run without creating true duplicates, because the load step always resolves down to "one row per business key, latest version wins." This is one of the specific engineering quality signals mentioned in your portfolio principles — a pipeline that isn't idempotent will slowly accumulate duplicate data every time it re-runs (e.g., after a failure and retry), which is a very real and common production bug.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `df = pd.DataFrame({"id": [1, 2, 2, 3], "value": ["a", "b", "b", "c"]})`, use `.duplicated()` to identify duplicate rows, and print the result alongside the original DataFrame.

<details>
<summary>Hint</summary>
`df.duplicated()`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"id": [1, 2, 2, 3], "value": ["a", "b", "b", "c"]})
print(df)
print(df.duplicated())
# 0    False
# 1    False
# 2     True
# 3    False
```
</details>

---

**Exercise 2 (Easy):** Using the same DataFrame, remove duplicate rows with `.drop_duplicates()` and print the result, confirming it now has 3 rows.

<details>
<summary>Hint</summary>
`df.drop_duplicates()`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"id": [1, 2, 2, 3], "value": ["a", "b", "b", "c"]})
deduped = df.drop_duplicates()
print(deduped)
print(len(deduped))   # 3
```
</details>

---

**Exercise 3 (Medium):** Given
```python
customers = pd.DataFrame({
    "customer_id": [1, 2, 3, 4],
    "email": ["maria@example.com", "james@example.com", "maria@example.com", "priya@example.com"],
    "signup_date": ["2024-01-10", "2024-02-15", "2024-03-01", "2024-04-20"],
})
```
find duplicate customers based on `email` only (note the `customer_id` and `signup_date` differ, but the email repeats — a sign of an accidental double-signup), showing both occurrences.

<details>
<summary>Hint</summary>
`customers.duplicated(subset=["email"], keep=False)`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

customers = pd.DataFrame({
    "customer_id": [1, 2, 3, 4],
    "email": ["maria@example.com", "james@example.com", "maria@example.com", "priya@example.com"],
    "signup_date": ["2024-01-10", "2024-02-15", "2024-03-01", "2024-04-20"],
})

dupes = customers[customers.duplicated(subset=["email"], keep=False)]
print(dupes)
#    customer_id               email signup_date
# 0            1  maria@example.com  2024-01-10
# 2            3  maria@example.com  2024-03-01
```
</details>

---

**Exercise 4 (Medium):** Using the same `customers` DataFrame, deduplicate by `email`, keeping the **earliest** `signup_date` for each duplicated email (hint: sort first, then use the appropriate `keep` value). Print the result.

<details>
<summary>Hint</summary>
Sort by `signup_date` ascending, then `drop_duplicates(subset=["email"], keep="first")`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

customers = pd.DataFrame({
    "customer_id": [1, 2, 3, 4],
    "email": ["maria@example.com", "james@example.com", "maria@example.com", "priya@example.com"],
    "signup_date": ["2024-01-10", "2024-02-15", "2024-03-01", "2024-04-20"],
})

sorted_customers = customers.sort_values("signup_date")
deduped = sorted_customers.drop_duplicates(subset=["email"], keep="first")
print(deduped.sort_values("customer_id"))
#    customer_id               email signup_date
# 0            1  maria@example.com  2024-01-10
# 1            2  james@example.com  2024-02-15
# 3            4  priya@example.com  2024-04-20
```
</details>

---

**Exercise 5 (Hard):** Given a simulated "pipeline re-run" scenario:
```python
orders = pd.DataFrame({
    "order_id": [101, 102, 103, 101, 104, 103],
    "status": ["pending", "shipped", "pending", "shipped", "shipped", "delivered"],
    "total": [45.99, 120.50, 89.99, 45.99, 15.00, 89.99],
    "updated_at": ["2024-11-01", "2024-11-01", "2024-11-01", "2024-11-03", "2024-11-01", "2024-11-04"],
})
```
Notice order 101's status changed from "pending" to "shipped" on a later date, and order 103 went from "pending" to "delivered." Write code that: (1) identifies which `order_id`s appear more than once, (2) deduplicates by `order_id`, always keeping the row with the LATEST `updated_at` (the current true status), and (3) prints the final deduplicated DataFrame sorted by `order_id`, confirming order 101 shows "shipped" and order 103 shows "delivered."

<details>
<summary>Hint</summary>
Use `.duplicated(subset=["order_id"], keep=False)` to find the repeated IDs first (for inspection/logging). Then sort by `updated_at` ascending and `drop_duplicates(subset=["order_id"], keep="last")` to keep the most recent status per order.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "order_id": [101, 102, 103, 101, 104, 103],
    "status": ["pending", "shipped", "pending", "shipped", "shipped", "delivered"],
    "total": [45.99, 120.50, 89.99, 45.99, 15.00, 89.99],
    "updated_at": ["2024-11-01", "2024-11-01", "2024-11-01", "2024-11-03", "2024-11-01", "2024-11-04"],
})

repeated_ids = orders[orders.duplicated(subset=["order_id"], keep=False)]["order_id"].unique()
print("Orders with multiple updates:", repeated_ids)   # [101 103]

sorted_orders = orders.sort_values("updated_at")
current_status = sorted_orders.drop_duplicates(subset=["order_id"], keep="last")
print(current_status.sort_values("order_id"))
#    order_id     status   total updated_at
# 0       101    shipped   45.99 2024-11-03
# 1       102    shipped  120.50 2024-11-01
# 5       103  delivered   89.99 2024-11-04
# 4       104    shipped   15.00 2024-11-01
```
</details>

---

**Previous lesson:** [4.7 — Renaming, Reordering & Dropping Columns](lesson-4.7-column-management.md)
**Next lesson:** [4.9 — Applying Functions](lesson-4.9-apply-map-replace.md)
