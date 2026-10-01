# Lesson 4.4 — Selecting & Filtering

## CONCEPT

pandas offers two primary ways to select rows and columns: **`.loc[]`** (label-based — select by row/column *names*) and **`.iloc[]`** (position-based — select by numeric *position*, like a list index). Combined with **boolean masking** (Module 3's technique, applied directly to DataFrames), these three tools cover essentially every selection and filtering need.

```python
import pandas as pd

df.loc[0]              # row with INDEX LABEL 0
df.iloc[0]               # row at POSITION 0 (the first row, regardless of its label)
df[df["price"] > 50]      # boolean filtering — rows where the condition is True
```

## WHY IT MATTERS

"Show me orders over $100," "get the first 10 rows," "find this specific customer's record" — these are the exact operations you'll perform dozens of times in any real analysis. This lesson is the direct pandas continuation of Lesson 3.3 (NumPy boolean masking) — the same `&`, `|`, `~` syntax applies here, just now filtering entire rows of a table instead of elements of a flat array.

## SYNTAX REFERENCE

```python
import pandas as pd

df = pd.DataFrame({
    "name": ["Mouse", "Cable", "Lamp", "Keyboard"],
    "price": [24.99, 12.50, 34.00, 59.99],
    "category": ["Electronics", "Electronics", "Home", "Electronics"],
}, index=[101, 102, 103, 104])

# .loc — label-based selection
df.loc[101]                    # row with index label 101
df.loc[101, "price"]            # a single value: row 101, column "price"
df.loc[101:103]                  # rows 101 through 103 INCLUSIVE (unlike Python slicing!)
df.loc[:, "price"]                 # every row, just the "price" column
df.loc[101:103, ["name", "price"]]   # rows 101-103, specific columns

# .iloc — position-based selection (always 0-indexed, upper bound EXCLUSIVE, like Python lists)
df.iloc[0]                    # first row, by position
df.iloc[0, 1]                    # row 0, column 1 (positionally)
df.iloc[0:2]                       # first 2 rows (position 0 and 1 — position 2 excluded)
df.iloc[:, 0]                        # every row, first column (positionally)
df.iloc[-1]                            # last row

# Boolean filtering — the most common day-to-day selection method
df[df["price"] > 30]
df[df["category"] == "Electronics"]

# Combining conditions — same rules as NumPy: use & | ~ with parentheses
df[(df["price"] > 20) & (df["category"] == "Electronics")]
df[(df["price"] < 15) | (df["price"] > 50)]
df[~(df["category"] == "Home")]

# .isin() — membership check against a list of values
df[df["category"].isin(["Electronics", "Outdoor"])]

# .between() — a clean range check
df[df["price"].between(20, 60)]

# Selecting specific columns after filtering
df[df["price"] > 30]["name"]                  # just the names of expensive products
df.loc[df["price"] > 30, "name"]                # equivalent, using .loc — often preferred

# Modifying values based on a condition (using .loc — the safe way, avoids warnings)
df.loc[df["price"] > 50, "category"] = "Premium Electronics"
```

## WORKED EXAMPLE

Selecting and filtering a small orders DataFrame using every technique from this lesson:

```python
import pandas as pd

orders = pd.DataFrame({
    "customer": ["Maria Chen", "James Lee", "Priya Patel", "Sam Osei", "Maria Chen"],
    "total": [45.99, 120.50, 89.99, 15.00, 200.00],
    "status": ["shipped", "pending", "shipped", "cancelled", "shipped"],
    "tier": ["gold", "silver", "gold", "bronze", "gold"],
}, index=[501, 502, 503, 504, 505])

# .loc — label-based access
print(orders.loc[502])
# customer     James Lee
# total            120.5
# status         pending
# tier            silver

print(orders.loc[501:503, ["customer", "total"]])   # note: 503 IS included with .loc
#            customer   total
# 501      Maria Chen   45.99
# 502       James Lee  120.50
# 503     Priya Patel   89.99

# .iloc — position-based access (ignores the custom index labels entirely)
print(orders.iloc[0])          # the FIRST row, regardless of its label (501)
print(orders.iloc[0:2])         # first 2 rows by position — 505 (position 4) NOT included

# Boolean filtering — orders over $80
large_orders = orders[orders["total"] > 80]
print(large_orders)
#            customer   total   status  tier
# 502       James Lee  120.50  pending silver
# 503     Priya Patel   89.99  shipped   gold
# 505      Maria Chen  200.00  shipped   gold

# Combined condition: shipped AND gold tier
shipped_gold = orders[(orders["status"] == "shipped") & (orders["tier"] == "gold")]
print(shipped_gold)
#            customer   total   status  tier
# 501      Maria Chen   45.99  shipped   gold
# 503     Priya Patel   89.99  shipped   gold
# 505      Maria Chen  200.00  shipped   gold

# isin() — orders from gold or silver tier customers
premium_tier = orders[orders["tier"].isin(["gold", "silver"])]
print(len(premium_tier))   # 4

# between() — a mid-range order total
mid_range = orders[orders["total"].between(40, 100)]
print(mid_range["customer"].tolist())   # ['Maria Chen', 'Priya Patel']

# Safely modifying values based on a condition, using .loc
orders.loc[orders["total"] > 150, "tier"] = "platinum"
print(orders)
#            customer   total   status      tier
# 501      Maria Chen   45.99  shipped      gold
# 502       James Lee  120.50  pending    silver
# 503     Priya Patel   89.99  shipped      gold
# 504        Sam Osei   15.00 cancelled    bronze
# 505      Maria Chen  200.00  shipped  platinum
```

## COMMON MISTAKES

**1. Confusing `.loc`'s inclusive slicing with `.iloc`'s exclusive slicing**
```python
df.loc[101:103]    # INCLUDES the row labeled 103
df.iloc[0:2]         # EXCLUDES position 2 — only positions 0 and 1
```
This is a genuinely confusing inconsistency in pandas (`.loc` slicing includes the endpoint, unlike almost everything else in Python) — worth memorizing deliberately, since it's a very common source of off-by-one bugs.

**2. Using `and`/`or` instead of `&`/`|` for combining boolean conditions**
```python
df[(df["price"] > 20) and (df["category"] == "Electronics")]   # ValueError
df[(df["price"] > 20) & (df["category"] == "Electronics")]     # correct
```
Identical trap to Lesson 3.3 — Python's `and`/`or` don't work element-wise on Series, and each condition must be wrapped in parentheses due to operator precedence.

**3. Getting a `SettingWithCopyWarning` when modifying a filtered subset**
```python
subset = df[df["price"] > 50]
subset["category"] = "Premium"   # SettingWithCopyWarning — unclear if this modifies df or not
```
Fix: use `.loc[]` with the condition directly on the original DataFrame for modifications, as shown in the worked example (`df.loc[df["price"] > 50, "category"] = "Premium"`), rather than filtering into an intermediate variable and modifying that.

**4. Forgetting `.iloc` ignores custom index labels entirely**
```python
df.iloc[501]   # IndexError, if the DataFrame only has 5 rows — .iloc wants a POSITION (0-4), not a label
```
Fix: use `.loc[501]` for label-based access, `.iloc[0]` for "the first row regardless of what it's labeled" — mixing these up is one of the most common early pandas mistakes.

**5. Chaining indexing operations unnecessarily**
```python
df[df["price"] > 30]["name"]      # works, but triggers the same ambiguity as Mistake #3 if you then try to assign to it
df.loc[df["price"] > 30, "name"]    # preferred — a single, clear operation
```
Fix: prefer a single `.loc[]` call combining the row filter and column selection over chaining two separate bracket operations, especially the moment you intend to *modify* the result rather than just read it.

**Data Engineer Tip:** `df.loc[condition, "column"] = value` is the standard, safe pattern for conditional updates in a pipeline — e.g., "flag any order over $500 as requiring manual review" or "recode a legacy status value." It's the direct pandas equivalent of an `UPDATE ... SET ... WHERE ...` SQL statement, and getting comfortable with this exact pattern here pays off directly when you're writing transformation logic for real pipeline stages later.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given a DataFrame `products` with a default index, use `.iloc[]` to select the first 3 rows.

<details>
<summary>Hint</summary>
`.iloc[0:3]`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

products = pd.DataFrame({
    "name": ["Mouse", "Cable", "Lamp", "Keyboard", "Monitor"],
    "price": [24.99, 12.50, 34.00, 59.99, 199.00],
})

first_three = products.iloc[0:3]
print(first_three)
```
</details>

---

**Exercise 2 (Easy):** Using the same DataFrame, filter to only rows where `price > 30`, and print the result.

<details>
<summary>Hint</summary>
`products[products["price"] > 30]`
</details>

<details>
<summary>Solution</summary>

```python
expensive = products[products["price"] > 30]
print(expensive)
```
</details>

---

**Exercise 3 (Medium):** Given a DataFrame `orders` with columns `total` and `status`, select just the `total` values for orders where `status == "shipped"`, using `.loc[]`.

<details>
<summary>Hint</summary>
`orders.loc[orders["status"] == "shipped", "total"]`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "total": [45.99, 120.50, 89.99, 15.00],
    "status": ["shipped", "pending", "shipped", "cancelled"],
})

shipped_totals = orders.loc[orders["status"] == "shipped", "total"]
print(shipped_totals)
```
</details>

---

**Exercise 4 (Medium):** Given `orders` with columns `total` and `tier` (values `"gold"`, `"silver"`, `"bronze"`), select rows where `tier` is either `"gold"` or `"silver"` AND `total > 50`, using `.isin()` combined with a comparison.

<details>
<summary>Hint</summary>
`orders[orders["tier"].isin(["gold", "silver"]) & (orders["total"] > 50)]`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "total": [45.99, 120.50, 89.99, 15.00, 200.00],
    "tier": ["gold", "silver", "bronze", "gold", "silver"],
})

result = orders[orders["tier"].isin(["gold", "silver"]) & (orders["total"] > 50)]
print(result)
```
</details>

---

**Exercise 5 (Hard):** Given this orders DataFrame:
```python
orders = pd.DataFrame({
    "customer": ["Maria Chen", "James Lee", "Priya Patel", "Sam Osei", "James Lee"],
    "total": [45.99, 320.00, 89.99, 15.00, 275.00],
    "status": ["shipped", "shipped", "pending", "cancelled", "shipped"],
})
```
Using `.loc[]`, apply this business rule directly to the DataFrame (modifying it in place, no new column): add a new column `priority` set to `"high"` for every row where `status == "shipped"` AND `total > 100`, and `"normal"` for every other row. (Hint: set the whole column to `"normal"` first, then overwrite the qualifying rows to `"high"` using a `.loc[]` condition.)

<details>
<summary>Hint</summary>
`orders["priority"] = "normal"` first, then `orders.loc[(orders["status"] == "shipped") & (orders["total"] > 100), "priority"] = "high"`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "customer": ["Maria Chen", "James Lee", "Priya Patel", "Sam Osei", "James Lee"],
    "total": [45.99, 320.00, 89.99, 15.00, 275.00],
    "status": ["shipped", "shipped", "pending", "cancelled", "shipped"],
})

orders["priority"] = "normal"
orders.loc[(orders["status"] == "shipped") & (orders["total"] > 100), "priority"] = "high"

print(orders)
#       customer   total     status priority
# 0   Maria Chen   45.99    shipped   normal
# 1    James Lee  320.00    shipped     high
# 2  Priya Patel   89.99    pending   normal
# 3     Sam Osei   15.00  cancelled   normal
# 4    James Lee  275.00    shipped     high
```
</details>

---

**Previous lesson:** [4.3 — Inspecting Data](lesson-4.3-inspecting-data.md)
**Next lesson:** [4.5 — Handling Missing Data](lesson-4.5-missing-data.md)
