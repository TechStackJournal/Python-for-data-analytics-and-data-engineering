# Lesson 4.9 — Applying Functions: `.apply()`, `.map()`, `.replace()`

## CONCEPT

When a transformation can't be expressed with pandas' built-in vectorized operations (Lesson 4.6's `.str` methods, arithmetic, etc.), pandas provides three tools to apply custom logic: **`.map()`** (element-wise substitution or simple function, on a Series), **`.apply()`** (a function applied to each element of a Series, or each row/column of a DataFrame), and **`.replace()`** (substituting specific values with other values).

```python
import pandas as pd

df["tier"].map({"gold": 3, "silver": 2, "bronze": 1})      # lookup-style substitution
df["price"].apply(lambda x: x * 1.1)                          # custom function per value
df["status"].replace({"shipped": "Shipped", "pending": "Pending"})   # value substitution
```

These are more flexible than vectorized built-ins, but — as covered in Module 3 — also slower, since they fall back to row-by-row Python execution rather than optimized vectorized code.

## WHY IT MATTERS

Not every transformation has a ready-made pandas method — sometimes you need genuinely custom logic (a multi-condition business rule, a call to an external lookup, string logic too specific for `.str` methods). `.apply()`/`.map()`/`.replace()` are the escape hatch for exactly this. But knowing *when* to reach for them versus a faster vectorized alternative is an important skill in its own right — this lesson covers both the tools and the judgment call.

## SYNTAX REFERENCE

```python
import pandas as pd

# .map() — Series only. Best for lookup-table style substitution or a simple function.
df["tier"].map({"gold": 3, "silver": 2, "bronze": 1})   # dict-based lookup
df["price"].map(lambda x: round(x, 2))                    # simple function per element

# .apply() — works on a Series (per-element) OR a DataFrame (per-row or per-column)
df["price"].apply(lambda x: x * 1.1)                       # Series: one value in, one value out

def classify(row):
    if row["total"] > 100:
        return "large"
    return "small"

df.apply(classify, axis=1)          # DataFrame: axis=1 = apply row-by-row (whole row available)
df.apply(lambda col: col.max(), axis=0)   # DataFrame: axis=0 = apply column-by-column (default)

# .replace() — direct value substitution, simpler than .map() when you're not
# transforming every value, just swapping specific ones
df["status"].replace({"shipped": "Shipped", "pending": "Pending"})
df["status"].replace("N/A", None)                # replace a single value
df.replace({"status": {"shipped": "Shipped"}})     # scoped to one column, on the whole DataFrame

# Named functions instead of lambdas (often more readable for anything non-trivial)
def apply_discount(price):
    return round(price * 0.9, 2)

df["price"].apply(apply_discount)

# Vectorized alternative — almost always faster, prefer this when possible (see Common Mistakes)
df["price"] * 1.1                              # instead of df["price"].apply(lambda x: x * 1.1)
np.where(df["total"] > 100, "large", "small")    # instead of df.apply(classify, axis=1)
```

## WORKED EXAMPLE

Applying a mix of `.map()`, `.apply()`, and `.replace()` to clean and enrich an orders DataFrame, then comparing against a faster vectorized equivalent:

```python
import pandas as pd
import numpy as np

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5],
    "total": [45.99, 220.00, 12.50, 89.99, 340.00],
    "customer_tier": ["silver", "gold", "bronze", "silver", "gold"],
    "status": ["shipped", "pending", "shipped", "cancelled", "shipped"],
})

# .map() — lookup-table substitution: convert tier labels into numeric weights
tier_weights = {"bronze": 1, "silver": 2, "gold": 3}
orders["tier_weight"] = orders["customer_tier"].map(tier_weights)
print(orders[["customer_tier", "tier_weight"]])
#   customer_tier  tier_weight
# 0        silver            2
# 1          gold            3
# 2        bronze            1
# 3        silver            2
# 4          gold            3

# .replace() — swap specific status values for display-friendly labels
status_labels = {"shipped": "Shipped", "pending": "Pending", "cancelled": "Cancelled"}
orders["status_display"] = orders["status"].replace(status_labels)
print(orders[["status", "status_display"]])

# .apply() with a lambda — simple per-value transformation
orders["total_with_tax"] = orders["total"].apply(lambda x: round(x * 1.08, 2))

# .apply() with axis=1 — a rule that needs MULTIPLE columns from the same row
def shipping_tier(row):
    if row["customer_tier"] == "gold":
        return "free"
    elif row["total"] > 75:
        return "free"
    else:
        return "standard"

orders["shipping"] = orders.apply(shipping_tier, axis=1)
print(orders[["customer_tier", "total", "shipping"]])
#   customer_tier   total shipping
# 0        silver   45.99 standard
# 1          gold  220.00     free
# 2        bronze   12.50 standard
# 3        silver   89.99     free
# 4          gold  340.00     free

# The SAME shipping_tier logic, written as a faster vectorized alternative using np.where
orders["shipping_vectorized"] = np.where(
    (orders["customer_tier"] == "gold") | (orders["total"] > 75),
    "free",
    "standard",
)
print((orders["shipping"] == orders["shipping_vectorized"]).all())   # True — same result, faster
```

## COMMON MISTAKES

**1. Reaching for `.apply()` when a vectorized operation already exists**
```python
df["price"].apply(lambda x: x * 1.1)     # works, but slower
df["price"] * 1.1                          # same result, vectorized, faster
```
Fix: before writing `.apply()`, ask "could this be plain arithmetic, a `.str` method (Lesson 4.6), or `np.where()`?" — reach for `.apply()` only once you've confirmed there's genuinely no vectorized equivalent. This mirrors the Module 3 lesson about loops vs. vectorized NumPy operations directly.

**2. Forgetting `axis=1` when a DataFrame-level `.apply()` needs to see the whole row**
```python
df.apply(shipping_tier)   # default axis=0 — passes each COLUMN, not each row! Wrong result or error.
df.apply(shipping_tier, axis=1)   # correct — passes each ROW as a Series
```
Fix: whenever your function needs values from *multiple columns at once* (like `shipping_tier` above, which reads both `customer_tier` and `total`), you need `axis=1`.

**3. Using `.map()` with a dict that doesn't cover every value**
```python
tier_weights = {"gold": 3, "silver": 2}   # missing "bronze"!
df["tier"].map(tier_weights)               # bronze rows become NaN — no error, silently wrong
```
Fix: double-check your mapping dictionary covers every distinct value actually present (`df["tier"].unique()` is a good way to confirm), or handle the unmapped case explicitly.

**4. Confusing `.map()` (Series only) with `.apply()` (works on both Series and DataFrame)**
```python
df.map({"a": 1})   # AttributeError in most pandas versions — .map() isn't a DataFrame method
```
Fix: `.map()` is specifically for a single column (Series); for whole-DataFrame element-wise replacement, use `.replace()` or `.applymap()` (a DataFrame-level equivalent, less commonly needed).

**5. Overcomplicating `.replace()` when `.map()` or a vectorized condition would be clearer**
`.replace()` is best for simple, direct value swaps. If you're building a large mapping dictionary to reclassify many values based on some pattern, a `.map()` with a function, or a vectorized `np.select()` for multiple conditions, is usually more scalable and maintainable than a giant `.replace()` dictionary.

**Data Engineer Tip:** The "vectorized alternative" comparison in the worked example isn't just a style preference — on a DataFrame with millions of rows, `.apply(axis=1)` can be dramatically slower than the `np.where()`/`np.select()` equivalent, because `axis=1` forces pandas to reconstruct a full row (as a Series) for every single row, one at a time. A common, genuinely important production pattern is: prototype with `.apply()` because it's easier to write and read, then — if that step turns out to be a real bottleneck on production-sized data — rewrite it as a vectorized equivalent once the logic is proven correct.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `df = pd.DataFrame({"status": ["A", "B", "C"]})`, use `.map()` with a dictionary to convert these into `{"A": "Active", "B": "Blocked", "C": "Closed"}`. Print the result.

<details>
<summary>Hint</summary>
`df["status"].map({"A": "Active", "B": "Blocked", "C": "Closed"})`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"status": ["A", "B", "C"]})
df["status_full"] = df["status"].map({"A": "Active", "B": "Blocked", "C": "Closed"})
print(df)
```
</details>

---

**Exercise 2 (Easy):** Given `df = pd.DataFrame({"price": [10, 20, 30]})`, use `.apply()` with a lambda to add a flat $2 fee to each price, and print the result. Then rewrite it as a vectorized operation instead and confirm the results match.

<details>
<summary>Hint</summary>
`.apply(lambda x: x + 2)` vs. simply `df["price"] + 2`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"price": [10, 20, 30]})
applied = df["price"].apply(lambda x: x + 2)
vectorized = df["price"] + 2

print(applied.tolist())       # [12, 22, 32]
print(vectorized.tolist())     # [12, 22, 32]
print((applied == vectorized).all())   # True
```
</details>

---

**Exercise 3 (Medium):** Given `df = pd.DataFrame({"status": ["shipped", "pending", "shipped", "N/A"]})`, use `.replace()` to change `"N/A"` specifically to `None`, leaving every other value unchanged. Print the result and confirm the `None` values are recognized as missing with `.isna()`.

<details>
<summary>Hint</summary>
`df["status"].replace("N/A", None)`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"status": ["shipped", "pending", "shipped", "N/A"]})
df["status_clean"] = df["status"].replace("N/A", None)
print(df)
print(df["status_clean"].isna())
# 0    False
# 1    False
# 2    False
# 3     True
```
</details>

---

**Exercise 4 (Medium):** Given
```python
orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4],
    "total": [45.99, 120.50, 15.00, 200.00],
})
```
write a function `size_label(total)` that returns `"small"` if under 50, `"medium"` if 50–150, and `"large"` otherwise. Apply it to the `total` column using `.apply()` to create a new `size` column. Print the result.

<details>
<summary>Hint</summary>
Define the function separately (not as a lambda, since it has multiple branches), then `df["total"].apply(size_label)`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4],
    "total": [45.99, 120.50, 15.00, 200.00],
})

def size_label(total):
    if total < 50:
        return "small"
    elif total <= 150:
        return "medium"
    else:
        return "large"

orders["size"] = orders["total"].apply(size_label)
print(orders)
#    order_id   total    size
# 0         1   45.99   small
# 1         2  120.50  medium
# 2         3   15.00   small
# 3         4  200.00   large
```
</details>

---

**Exercise 5 (Hard):** Given
```python
orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5],
    "total": [45.99, 220.00, 89.99, 15.00, 340.00],
    "customer_tier": ["silver", "gold", "bronze", "silver", "gold"],
    "is_first_order": [True, False, True, False, True],
})
```
Write a row-based function `loyalty_points(row)` that computes points earned using this rule: base points = `total` rounded down to the nearest whole number (hint: `int(row["total"])`), doubled if `customer_tier == "gold"`, and with an additional flat +50 bonus if `is_first_order` is `True`. Apply it with `.apply(axis=1)` to create a `points` column. Then rewrite the same logic as a fully vectorized version using `np.where()` (you'll need to combine a couple of `np.where()` calls or use `np.select()`), and confirm both approaches produce identical results.

<details>
<summary>Hint</summary>
For the `.apply()` version, build the points step by step inside the function using ordinary if-statements — this is exactly the multi-condition logic from Lesson 1.4. For the vectorized version, compute `base = orders["total"].astype(int)`, then `doubled = np.where(orders["customer_tier"] == "gold", base * 2, base)`, then add the bonus with another `np.where()` on `is_first_order`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import numpy as np

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5],
    "total": [45.99, 220.00, 89.99, 15.00, 340.00],
    "customer_tier": ["silver", "gold", "bronze", "silver", "gold"],
    "is_first_order": [True, False, True, False, True],
})

def loyalty_points(row):
    points = int(row["total"])
    if row["customer_tier"] == "gold":
        points *= 2
    if row["is_first_order"]:
        points += 50
    return points

orders["points"] = orders.apply(loyalty_points, axis=1)

# Vectorized equivalent
base = orders["total"].astype(int)
doubled = np.where(orders["customer_tier"] == "gold", base * 2, base)
final_points = np.where(orders["is_first_order"], doubled + 50, doubled)
orders["points_vectorized"] = final_points

print(orders[["order_id", "total", "customer_tier", "is_first_order", "points", "points_vectorized"]])
print("Match:", (orders["points"] == orders["points_vectorized"]).all())   # True
```
</details>

---

## Module 4 Complete

You've now covered the core pandas data-wrangling toolkit: Series and DataFrames, reading data from multiple formats, inspecting, filtering, cleaning missing data and types, managing columns, deduplicating, and applying custom functions. Module 5 builds directly on this foundation to answer real analytical questions — grouping, aggregating, merging datasets, and working with time series.

**Previous lesson:** [4.8 — Detecting & Handling Duplicates](lesson-4.8-duplicates.md)
**Next module:** [Module 5 — Pandas II: Analysis & Aggregation](../module-5-pandas-2-analysis-and-aggregation/README.md)
