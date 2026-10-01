# Lesson 5.5 — Concatenating & Appending

## CONCEPT

**`pd.concat()`** stacks multiple DataFrames together — either **vertically** (adding more rows, when the DataFrames share the same columns) or **horizontally** (adding more columns, when they share the same index). This is fundamentally different from merging (Lesson 5.4): a merge *combines* rows based on matching key values; concatenation just *stacks* datasets together, matching (or not) purely on position/column names.

```python
import pandas as pd

pd.concat([orders_january, orders_february])   # vertical stack — more rows
```

## WHY IT MATTERS

Real data frequently arrives in separate chunks that need to be combined before analysis: monthly export files, data from different regions or systems with the same structure, results from several API pages. This is the pandas equivalent of SQL's `UNION ALL` — and it's an extremely common first step in any pipeline that ingests data delivered in batches rather than as one complete file.

## SYNTAX REFERENCE

```python
import pandas as pd

# Vertical concatenation (stacking rows) — the most common use case
combined = pd.concat([df1, df2, df3])            # stacks rows; index values may repeat!
combined = pd.concat([df1, df2], ignore_index=True)   # resets to a clean 0, 1, 2... index

# Horizontal concatenation (stacking columns) — less common, requires matching index
combined = pd.concat([df1, df2], axis=1)

# Concatenating DataFrames with slightly different columns
pd.concat([df1, df2])   # missing columns in either DataFrame become NaN for those rows
pd.concat([df1, df2], join="inner")   # keeps ONLY columns present in BOTH DataFrames

# Adding a label identifying which original DataFrame each row came from
pd.concat([df1, df2], keys=["january", "february"])
# creates a MultiIndex — useful for tracing rows back to their source after combining

# Concatenating a list built programmatically (a very common real pattern)
monthly_files = ["jan.csv", "feb.csv", "mar.csv"]
all_months = pd.concat([pd.read_csv(f) for f in monthly_files], ignore_index=True)

# The older .append() method existed in pandas but is REMOVED as of pandas 2.0 —
# always use pd.concat() instead, even for adding a single row/DataFrame
```

## WORKED EXAMPLE

Combining monthly order exports into one dataset, then handling a schema mismatch between two of the files:

```python
import pandas as pd

orders_jan = pd.DataFrame({
    "order_id": [1, 2, 3],
    "total": [45.99, 120.50, 89.99],
    "status": ["shipped", "shipped", "pending"],
})

orders_feb = pd.DataFrame({
    "order_id": [4, 5],
    "total": [34.00, 65.00],
    "status": ["shipped", "cancelled"],
})

# Basic vertical concat — stack the two months together
all_orders = pd.concat([orders_jan, orders_feb], ignore_index=True)
print(all_orders)
#    order_id   total     status
# 0         1   45.99    shipped
# 1         2  120.50    shipped
# 2         3   89.99    pending
# 3         4   34.00    shipped
# 4         5   65.00  cancelled

# ignore_index=True gave us a clean 0-4 index instead of two separate 0-2, 0-1 ranges
# WITHOUT ignore_index=True, the index would have duplicate values (0, 1, 2, 0, 1) — a
# common source of confusing bugs if later code assumes the index is unique

# A realistic complication: March's file has an EXTRA column ("region") that Jan/Feb don't
orders_mar = pd.DataFrame({
    "order_id": [6, 7],
    "total": [15.00, 199.99],
    "status": ["shipped", "shipped"],
    "region": ["East", "West"],
})

# Default concat behavior: missing columns become NaN
all_orders_v2 = pd.concat([orders_jan, orders_feb, orders_mar], ignore_index=True)
print(all_orders_v2)
#    order_id   total     status region
# 0         1   45.99    shipped    NaN
# 1         2  120.50    shipped    NaN
# 2         3   89.99    pending    NaN
# 3         4   34.00    shipped    NaN
# 4         5   65.00  cancelled    NaN
# 5         6   15.00    shipped   East
# 6         7  199.99    shipped   West

# Tracking WHICH source file each row came from — useful for debugging data issues
labeled = pd.concat(
    [orders_jan, orders_feb, orders_mar],
    keys=["january", "february", "march"],
)
print(labeled.head(4))
#                   order_id   total   status region
# january  0               1   45.99  shipped    NaN
#          1               2  120.50  shipped    NaN
#          2               3   89.99  pending    NaN
# february 0               4   34.00  shipped    NaN

# The common real-world loop pattern: reading and combining several files at once
# (simulated here without actual files, for illustration)
monthly_dataframes = [orders_jan, orders_feb, orders_mar]
combined_from_loop = pd.concat(monthly_dataframes, ignore_index=True)
print(f"Total combined rows: {len(combined_from_loop)}")   # 7
```

## COMMON MISTAKES

**1. Forgetting `ignore_index=True` and ending up with duplicate index values**
```python
combined = pd.concat([df1, df2])   # index might be [0,1,2,0,1] — NOT unique!
combined.loc[0]   # returns MULTIPLE rows, not one — a classic downstream bug
```
Fix: use `ignore_index=True` whenever the original index values don't carry meaningful information you need to preserve (usually the case for simple row-stacking).

**2. Using `pd.concat()` when you actually meant `.merge()`**
```python
pd.concat([orders, customers], axis=1)   # stacks side-by-side by POSITION, not by matching keys!
```
This silently produces nonsense if the two DataFrames aren't already perfectly row-aligned. Fix: if you're combining datasets based on a *shared key* (like `customer_id`), you want `.merge()` (Lesson 5.4), not `concat(axis=1)` — concat's horizontal mode is for the much rarer case of two DataFrames that are already guaranteed to be in the same row order.

**3. Not checking column consistency before concatenating**
```python
pd.concat([df1, df2])   # if df2 has "Total" (capitalized) and df1 has "total", pandas
                          # treats them as TWO DIFFERENT columns — mostly NaN result!
```
Fix: verify column names match exactly (case, spelling) across all DataFrames being concatenated — this is exactly why standardizing column names early (Lesson 4.7) pays off before a later concatenation step.

**4. Using `join="outer"` (the default) when you actually want only the shared columns**
```python
pd.concat([df1, df2])   # default: keeps ALL columns from EITHER DataFrame, NaN-filling gaps
pd.concat([df1, df2], join="inner")   # keeps ONLY columns present in BOTH
```
Fix: think about whether missing columns in some files should become `NaN` (usually right, so you know data is incomplete) or should be dropped entirely (`join="inner"`, appropriate when you specifically only care about the fields every source has in common).

**5. Calling `.append()`, which no longer exists in modern pandas**
```python
df.append(new_row)   # AttributeError in pandas 2.0+ — .append() was removed entirely
```
Fix: always use `pd.concat([df, new_row_df])` instead — this is a common point of confusion for anyone learning from older tutorials or Stack Overflow answers written before pandas 2.0.

**Data Engineer Tip:** The "combine a list of monthly files" pattern in the worked example is exactly how a real batch pipeline often ingests data that arrives as separate daily or monthly drops — `pd.concat([pd.read_csv(f) for f in file_list], ignore_index=True)` is a genuinely common line of real production code. The `keys=` parameter, tagging each row with its source file, is also a practical debugging tool: when a data quality issue turns up in the combined dataset, being able to trace a bad row back to "it came from the March file" narrows down the investigation immediately.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `df1 = pd.DataFrame({"id": [1, 2], "value": ["a", "b"]})` and `df2 = pd.DataFrame({"id": [3, 4], "value": ["c", "d"]})`, concatenate them with a clean, reset index, and print the result.

<details>
<summary>Hint</summary>
`pd.concat([df1, df2], ignore_index=True)`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df1 = pd.DataFrame({"id": [1, 2], "value": ["a", "b"]})
df2 = pd.DataFrame({"id": [3, 4], "value": ["c", "d"]})

combined = pd.concat([df1, df2], ignore_index=True)
print(combined)
```
</details>

---

**Exercise 2 (Easy):** Given the same two DataFrames, concatenate them WITHOUT `ignore_index=True`, and demonstrate the duplicate-index problem by showing that `.loc[0]` now returns more than one row.

<details>
<summary>Hint</summary>
`pd.concat([df1, df2])` (no `ignore_index`), then `.loc[0]`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df1 = pd.DataFrame({"id": [1, 2], "value": ["a", "b"]})
df2 = pd.DataFrame({"id": [3, 4], "value": ["c", "d"]})

combined = pd.concat([df1, df2])
print(combined.loc[0])
# returns TWO rows (id=1 from df1, id=3 from df2) — both have index label 0
```
</details>

---

**Exercise 3 (Medium):** Given three small DataFrames representing three days of sales:
```python
day1 = pd.DataFrame({"item": ["A", "B"], "sales": [10, 20]})
day2 = pd.DataFrame({"item": ["A", "C"], "sales": [15, 5]})
day3 = pd.DataFrame({"item": ["B", "C"], "sales": [25, 8]})
```
Concatenate all three into one DataFrame with a clean index, then use `.groupby("item")["sales"].sum()` to get total sales per item across all three days. Print both the combined DataFrame and the grouped totals.

<details>
<summary>Hint</summary>
`pd.concat([day1, day2, day3], ignore_index=True)`, then group as usual.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

day1 = pd.DataFrame({"item": ["A", "B"], "sales": [10, 20]})
day2 = pd.DataFrame({"item": ["A", "C"], "sales": [15, 5]})
day3 = pd.DataFrame({"item": ["B", "C"], "sales": [25, 8]})

combined = pd.concat([day1, day2, day3], ignore_index=True)
print(combined)

totals = combined.groupby("item")["sales"].sum()
print(totals)
# A    25
# B    45
# C    13
```
</details>

---

**Exercise 4 (Medium):** Given
```python
region_a = pd.DataFrame({"order_id": [1, 2], "total": [45.99, 89.99]})
region_b = pd.DataFrame({"order_id": [3, 4], "total": [120.50, 34.00], "notes": ["rush", "gift"]})
```
(note `region_b` has an extra `notes` column), concatenate them and print the result — confirm `region_a`'s rows show `NaN` for `notes`.

<details>
<summary>Hint</summary>
Default `pd.concat()` behavior fills missing columns with `NaN` automatically.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

region_a = pd.DataFrame({"order_id": [1, 2], "total": [45.99, 89.99]})
region_b = pd.DataFrame({"order_id": [3, 4], "total": [120.50, 34.00], "notes": ["rush", "gift"]})

combined = pd.concat([region_a, region_b], ignore_index=True)
print(combined)
#    order_id   total notes
# 0         1   45.99   NaN
# 1         2   89.99   NaN
# 2         3  120.50  rush
# 3         4   34.00  gift
```
</details>

---

**Exercise 5 (Hard):** Simulate combining data from 3 regional exports, each with a slightly different set of columns:
```python
east = pd.DataFrame({"order_id": [1, 2], "total": [45.99, 89.99], "rep": ["Alice", "Bob"]})
west = pd.DataFrame({"order_id": [3, 4], "total": [120.50, 34.00]})
central = pd.DataFrame({"order_id": [5, 6], "total": [65.00, 199.99], "rep": ["Carol", "Dan"], "priority": [True, False]})
```
Concatenate all three using `keys=` to tag each row with its source region (`"east"`, `"west"`, `"central"`), with a clean combined structure. Then, using the resulting MultiIndex, print just the rows that came from `"central"` (hint: use `.loc["central"]`). Finally, compute the total revenue per source region using `.groupby(level=0)["total"].sum()` (grouping by the OUTER index level, which holds the region labels).

<details>
<summary>Hint</summary>
`pd.concat([east, west, central], keys=["east", "west", "central"])`. For extracting one region: `.loc["central"]`. For grouping by the outer MultiIndex level: `.groupby(level=0)`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

east = pd.DataFrame({"order_id": [1, 2], "total": [45.99, 89.99], "rep": ["Alice", "Bob"]})
west = pd.DataFrame({"order_id": [3, 4], "total": [120.50, 34.00]})
central = pd.DataFrame({"order_id": [5, 6], "total": [65.00, 199.99], "rep": ["Carol", "Dan"], "priority": [True, False]})

combined = pd.concat([east, west, central], keys=["east", "west", "central"])
print(combined)

central_only = combined.loc["central"]
print(central_only)

region_totals = combined.groupby(level=0)["total"].sum()
print(region_totals)
# central    264.99
# east        135.98
# west        154.50
```
</details>

---

**Previous lesson:** [5.4 — Merging & Joining DataFrames](lesson-5.4-merging-and-joining.md)
**Next lesson:** [5.6 — Time Series Basics](lesson-5.6-time-series-basics.md)
