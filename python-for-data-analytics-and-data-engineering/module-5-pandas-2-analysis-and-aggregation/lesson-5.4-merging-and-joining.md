# Lesson 5.4 — Merging & Joining DataFrames

## CONCEPT

**`.merge()`** combines two DataFrames based on matching values in one or more shared columns — the pandas equivalent of a SQL `JOIN`. Real data is almost never in one flat table; the `ecommerce` schema alone has six related tables, and answering most real questions requires combining at least two of them.

```python
import pandas as pd

orders.merge(customers, on="customer_id", how="left")
```

pandas supports the same join types you already know from SQL: inner, left, right, and outer/full.

## WHY IT MATTERS

"Show me each order with the customer's name and tier" requires combining `orders` with `customers`. "Which products have never been ordered?" requires combining `products` with `order_items` and specifically looking for non-matches. Merging is unavoidable in any analysis touching more than one table — and since this maps directly onto SQL joins you've already learned, this lesson is largely about translating a skill you already have into pandas syntax.

## SYNTAX REFERENCE

```python
import pandas as pd

# Basic merge — matches on a column present in BOTH DataFrames
orders.merge(customers, on="customer_id")                # inner join (default)
orders.merge(customers, on="customer_id", how="left")      # keep all orders, matched or not
orders.merge(customers, on="customer_id", how="right")       # keep all customers
orders.merge(customers, on="customer_id", how="outer")         # keep everything from both sides

# Merging when the key columns have DIFFERENT names in each DataFrame
orders.merge(customers, left_on="cust_id", right_on="customer_id")

# Merging on the INDEX instead of a column
orders.merge(customers, left_index=True, right_index=True)

# Merging on multiple columns at once
order_items.merge(products, on=["product_id", "warehouse_id"])

# Handling overlapping column names (other than the join key) — pandas auto-suffixes
orders.merge(customers, on="customer_id", suffixes=("_order", "_customer"))

# Checking which rows matched, using the 'indicator' parameter — very useful for auditing
merged = orders.merge(customers, on="customer_id", how="left", indicator=True)
merged["_merge"].value_counts()   # 'both', 'left_only', 'right_only'

# Finding NON-matches specifically (e.g. "orders with no matching customer" — a data quality check)
unmatched = orders.merge(customers, on="customer_id", how="left", indicator=True)
unmatched = unmatched[unmatched["_merge"] == "left_only"]
```

## WORKED EXAMPLE

Combining orders, customers, and order items to answer a realistic multi-table question — and specifically auditing for unmatched records, a common data quality check:

```python
import pandas as pd

customers = pd.DataFrame({
    "customer_id": [101, 102, 103, 104],
    "name": ["Maria Chen", "James Lee", "Priya Patel", "Sam Torres"],
    "tier": ["gold", "silver", "gold", "bronze"],
})

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5],
    "customer_id": [101, 102, 101, 999, 103],   # note: customer_id 999 doesn't exist!
    "total": [45.99, 120.50, 89.99, 15.00, 65.00],
})

# Basic left join: keep every order, attach customer info where it exists
orders_with_customers = orders.merge(customers, on="customer_id", how="left")
print(orders_with_customers)
#    order_id  customer_id   total        name  tier
# 0         1          101   45.99  Maria Chen  gold
# 1         2          102  120.50   James Lee  silver
# 2         3          101   89.99  Maria Chen  gold
# 3         4          999   15.00         NaN   NaN
# 4         5          103   65.00  Priya Patel  gold
# Order 4's customer_id (999) had no match — name/tier came back as NaN,
# exactly like a LEFT JOIN in SQL

# Data quality check: find orders referencing a customer_id that doesn't exist
audit = orders.merge(customers, on="customer_id", how="left", indicator=True)
orphaned_orders = audit[audit["_merge"] == "left_only"]
print(f"Orders with no matching customer: {len(orphaned_orders)}")
print(orphaned_orders[["order_id", "customer_id", "total"]])
#    order_id  customer_id  total
# 3         4          999   15.0

# Inner join — only orders that DO have a valid customer match
valid_orders = orders.merge(customers, on="customer_id", how="inner")
print(f"Valid orders: {len(valid_orders)} out of {len(orders)} total")
# Valid orders: 4 out of 5 total

# Now combine with a THIRD table — order_items, to compute total spend per customer
order_items = pd.DataFrame({
    "order_id": [1, 1, 2, 3, 5],
    "product": ["Mouse", "Cable", "Lamp", "Keyboard", "Mat"],
    "line_total": [24.99, 12.50, 120.50, 59.99, 19.99],
})

# Chain multiple merges: order_items -> orders -> customers
full_picture = (
    order_items
    .merge(orders, on="order_id", how="left")
    .merge(customers, on="customer_id", how="left")
)
print(full_picture[["order_id", "product", "line_total", "name", "tier"]])
#    order_id   product  line_total        name  tier
# 0         1     Mouse       24.99  Maria Chen  gold
# 1         1     Cable       12.50  Maria Chen  gold
# 2         2      Lamp      120.50   James Lee  silver
# 3         3  Keyboard       59.99  Maria Chen  gold
# 4         5       Mat       19.99  Priya Patel  gold

# Revenue by customer TIER, using the fully joined data
revenue_by_tier = full_picture.groupby("tier")["line_total"].sum()
print(revenue_by_tier)
# tier
# gold      217.47
# silver    120.50
```

## COMMON MISTAKES

**1. Defaulting to inner join without realizing rows silently disappear**
```python
orders.merge(customers, on="customer_id")   # default how="inner" — the orphaned order 999 VANISHES
```
Fix: `how="inner"` is the default, and it silently drops any row without a match on *either* side — often not what you want when the goal is "keep all my orders and attach customer info where available." Use `how="left"` explicitly when you want to preserve every row from your primary table.

**2. Not checking for duplicated join keys before merging**
```python
# if "customer_id" isn't unique in customers (e.g. a data quality issue upstream),
# merging can silently produce MORE rows than either original table — a "fan-out"
```
Fix: before merging, it's good practice to check `customers["customer_id"].is_unique` — an unexpected duplicate key on the "one" side of a supposed one-to-many relationship is a common and very confusing source of inflated row counts after a merge.

**3. Merging on columns with mismatched types**
```python
orders["customer_id"].dtype    # int64
customers["customer_id"].dtype  # object (string) — e.g. "101" instead of 101
orders.merge(customers, on="customer_id")   # returns ZERO matches — no error, just nothing matches!
```
Fix: check both key columns' dtypes match before merging (Lesson 4.6's type conversion) — a silent type mismatch produces an empty or incomplete result with no error message to flag it.

**4. Forgetting overlapping non-key column names get auto-suffixed**
```python
# if BOTH orders and customers happen to have a "status" column,
# the merged result gets "status_x" and "status_y" automatically
```
Fix: either rename ahead of time, or use the `suffixes=` parameter to choose more meaningful labels than the default `_x`/`_y`.

**5. Chaining many merges without checking row counts at each step**
```python
result = a.merge(b, on="id").merge(c, on="id").merge(d, on="id")
# if any of the joins fans out unexpectedly, the final row count can be wildly wrong
# with no obvious clue about WHICH merge caused it
```
Fix: for a complex multi-table merge, check `len(df)` after each individual `.merge()` step while developing, to catch an unexpected row-count change at the exact step it happens — the worked example's chained merge is simple enough not to need this, but it becomes essential once you're joining four or five tables together.

**Data Engineer Tip:** The `indicator=True` audit pattern in the worked example — finding rows that failed to match a join — is a genuinely important real-world data quality check, often called a **referential integrity check**: "does every `customer_id` in `orders` actually exist in `customers`?" In a production pipeline, this exact check (usually run in SQL against the actual database, using `LEFT JOIN ... WHERE right.id IS NULL`) is one of the most common automated data quality gates, catching broken foreign keys or late-arriving dimension data before they corrupt downstream reports.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given
```python
products = pd.DataFrame({"product_id": [1, 2, 3], "name": ["Mouse", "Keyboard", "Monitor"]})
prices = pd.DataFrame({"product_id": [1, 2, 3], "price": [24.99, 59.99, 199.00]})
```
merge them on `product_id` and print the result.

<details>
<summary>Hint</summary>
`products.merge(prices, on="product_id")`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

products = pd.DataFrame({"product_id": [1, 2, 3], "name": ["Mouse", "Keyboard", "Monitor"]})
prices = pd.DataFrame({"product_id": [1, 2, 3], "price": [24.99, 59.99, 199.00]})

result = products.merge(prices, on="product_id")
print(result)
```
</details>

---

**Exercise 2 (Easy):** Given
```python
orders = pd.DataFrame({"order_id": [1, 2, 3], "customer_id": [101, 102, 103]})
customers = pd.DataFrame({"customer_id": [101, 102], "name": ["Maria", "James"]})
```
(note: customer 103 is missing from `customers`), perform a LEFT join keeping all orders, and print the result — confirm the row for customer 103 shows `NaN` for `name`.

<details>
<summary>Hint</summary>
`orders.merge(customers, on="customer_id", how="left")`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({"order_id": [1, 2, 3], "customer_id": [101, 102, 103]})
customers = pd.DataFrame({"customer_id": [101, 102], "name": ["Maria", "James"]})

result = orders.merge(customers, on="customer_id", how="left")
print(result)
#    order_id  customer_id   name
# 0         1          101  Maria
# 1         2          102  James
# 2         3          103    NaN
```
</details>

---

**Exercise 3 (Medium):** Using the same `orders` and `customers` DataFrames from Exercise 2, use `indicator=True` to find which order(s) have no matching customer record, and print just those rows.

<details>
<summary>Hint</summary>
`merged[merged["_merge"] == "left_only"]`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({"order_id": [1, 2, 3], "customer_id": [101, 102, 103]})
customers = pd.DataFrame({"customer_id": [101, 102], "name": ["Maria", "James"]})

merged = orders.merge(customers, on="customer_id", how="left", indicator=True)
unmatched = merged[merged["_merge"] == "left_only"]
print(unmatched)
```
</details>

---

**Exercise 4 (Medium):** Given
```python
orders = pd.DataFrame({"order_id": [1, 2, 3], "cust_id": [101, 102, 103]})
customers = pd.DataFrame({"customer_id": [101, 102, 103], "name": ["Maria", "James", "Priya"]})
```
(note the key columns have different names: `cust_id` vs. `customer_id`), merge them correctly and print the result.

<details>
<summary>Hint</summary>
`left_on="cust_id", right_on="customer_id"`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({"order_id": [1, 2, 3], "cust_id": [101, 102, 103]})
customers = pd.DataFrame({"customer_id": [101, 102, 103], "name": ["Maria", "James", "Priya"]})

result = orders.merge(customers, left_on="cust_id", right_on="customer_id")
print(result)
```
</details>

---

**Exercise 5 (Hard):** Given three related tables:
```python
customers = pd.DataFrame({
    "customer_id": [1, 2, 3],
    "name": ["Maria", "James", "Priya"],
    "tier": ["gold", "silver", "gold"],
})
orders = pd.DataFrame({
    "order_id": [10, 11, 12, 13],
    "customer_id": [1, 2, 1, 3],
})
order_items = pd.DataFrame({
    "order_id": [10, 10, 11, 12, 13],
    "product": ["Mouse", "Cable", "Lamp", "Keyboard", "Mat"],
    "line_total": [24.99, 12.50, 120.50, 59.99, 19.99],
})
```
Chain the merges to combine all three tables into one DataFrame (`order_items` → `orders` → `customers`), then compute total spend per customer `tier` using `.groupby()`. Print the fully merged DataFrame and the final tier totals.

<details>
<summary>Hint</summary>
Chain two `.merge()` calls, each using `how="left"`, then `.groupby("tier")["line_total"].sum()`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

customers = pd.DataFrame({
    "customer_id": [1, 2, 3],
    "name": ["Maria", "James", "Priya"],
    "tier": ["gold", "silver", "gold"],
})
orders = pd.DataFrame({
    "order_id": [10, 11, 12, 13],
    "customer_id": [1, 2, 1, 3],
})
order_items = pd.DataFrame({
    "order_id": [10, 10, 11, 12, 13],
    "product": ["Mouse", "Cable", "Lamp", "Keyboard", "Mat"],
    "line_total": [24.99, 12.50, 120.50, 59.99, 19.99],
})

full = order_items.merge(orders, on="order_id", how="left").merge(customers, on="customer_id", how="left")
print(full)

tier_totals = full.groupby("tier")["line_total"].sum()
print(tier_totals)
# tier
# gold      117.47
# silver    120.50
```
</details>

---

**Previous lesson:** [5.3 — Pivot Tables & crosstab()](lesson-5.3-pivot-tables.md)
**Next lesson:** [5.5 — Concatenating & Appending](lesson-5.5-concatenating.md)
