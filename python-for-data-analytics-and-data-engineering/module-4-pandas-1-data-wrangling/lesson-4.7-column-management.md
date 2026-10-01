# Lesson 4.7 — Renaming, Reordering & Dropping Columns

## CONCEPT

Beyond cleaning individual values (Lesson 4.6), datasets often need structural cleanup at the column level: inconsistent or unclear column names, columns in a confusing order, or columns you simply don't need for this analysis. pandas provides direct, explicit methods for all three.

```python
import pandas as pd

df.rename(columns={"cust_nm": "customer_name"})
df[["customer_name", "total", "status"]]      # reordering by explicit selection
df.drop(columns=["internal_notes"])
```

## WHY IT MATTERS

Raw data — especially from databases or legacy systems — frequently has cryptic column names (`cust_nm`, `amt`, `dt`), inconsistent naming conventions (`CustomerName` next to `order_total` next to `Status`), and extra columns irrelevant to your analysis. Cleaning up column names and structure early makes every subsequent line of code more readable, both for you and for anyone reviewing your work — a `df["customer_name"]` is simply easier to trust and reason about than `df["cust_nm"]`.

## SYNTAX REFERENCE

```python
import pandas as pd

# Renaming specific columns
df.rename(columns={"cust_nm": "customer_name", "amt": "total"})

# Renaming ALL columns at once (must match the exact number of columns)
df.columns = ["customer_name", "total", "status"]

# Standardizing all column names in bulk (a very common pattern)
df.columns = df.columns.str.lower().str.replace(" ", "_")
# "Customer Name" -> "customer_name", "Order Total" -> "order_total"

# Reordering columns — just select them in the order you want
df[["customer_name", "total", "status"]]

# Moving one column to the front
cols = ["order_id"] + [c for c in df.columns if c != "order_id"]
df = df[cols]

# Dropping columns
df.drop(columns=["internal_notes"])                # drop one column
df.drop(columns=["internal_notes", "legacy_id"])      # drop multiple
df.drop("internal_notes", axis=1)                       # older syntax, same result

# Dropping rows (axis=0, the default) vs columns (axis=1) — same .drop() method
df.drop(index=[0, 1])          # drop specific rows by label
df.drop(columns=["col_a"])       # drop specific columns

# Adding a new column (covered fully in Lesson 4.9, previewed here)
df["new_column"] = df["price"] * 1.1

# Reassignment is required — rename/drop return a NEW DataFrame by default
df = df.rename(columns={"amt": "total"})
```

## WORKED EXAMPLE

Cleaning up a raw dataset's column structure — a common "first pass" step right after loading:

```python
import pandas as pd

raw = pd.DataFrame({
    "OrderID": [1, 2, 3],
    "cust_nm": ["Maria Chen", "James Lee", "Priya Patel"],
    "amt": [45.99, 120.50, 89.99],
    "Status ": ["shipped", "pending", "shipped"],       # note the trailing space in the name!
    "internal_notes": ["", "follow up needed", ""],
    "legacy_flag": [0, 0, 1],
})

print(raw.columns.tolist())
# ['OrderID', 'cust_nm', 'amt', 'Status ', 'internal_notes', 'legacy_flag']

# Step 1: standardize all column names — lowercase, strip whitespace, snake_case
cleaned = raw.copy()
cleaned.columns = (
    cleaned.columns
    .str.strip()
    .str.lower()
    .str.replace(" ", "_")
)
print(cleaned.columns.tolist())
# ['orderid', 'cust_nm', 'amt', 'status', 'internal_notes', 'legacy_flag']

# Step 2: rename specific columns to clearer, more descriptive names
cleaned = cleaned.rename(columns={
    "orderid": "order_id",
    "cust_nm": "customer_name",
    "amt": "total",
})
print(cleaned.columns.tolist())
# ['order_id', 'customer_name', 'total', 'status', 'internal_notes', 'legacy_flag']

# Step 3: drop columns not needed for this analysis
cleaned = cleaned.drop(columns=["internal_notes", "legacy_flag"])
print(cleaned.columns.tolist())
# ['order_id', 'customer_name', 'total', 'status']

# Step 4: reorder — put status right after order_id for readability
cleaned = cleaned[["order_id", "status", "customer_name", "total"]]

print(cleaned)
#    order_id   status customer_name   total
# 0         1  shipped    Maria Chen   45.99
# 1         2  pending     James Lee  120.50
# 2         3  shipped   Priya Patel   89.99
```

## COMMON MISTAKES

**1. Forgetting `rename()` needs `columns=` to target column names specifically**
```python
df.rename({"amt": "total"})            # does nothing by default — ambiguous, targets the INDEX
df.rename(columns={"amt": "total"})      # correct — explicitly targets columns
```
Fix: always be explicit with `columns=` (or `index=` if you specifically mean to rename row labels) — the unscoped call defaults to renaming index labels, not columns, which silently does nothing useful if that's not what you intended.

**2. Assigning `df.columns = [...]` with the wrong number of names**
```python
df.columns = ["a", "b"]   # ValueError, if df actually has 5 columns
```
Fix: this approach requires you to supply a name for literally every column in order — `rename()` with a dictionary is safer when you only want to change a few specific names, since it doesn't require you to re-specify every column.

**3. Trying to drop a column that doesn't exist and getting a `KeyError`**
```python
df.drop(columns=["typo_column_name"])   # KeyError: "['typo_column_name'] not found in axis"
```
Fix: double-check exact column names with `df.columns.tolist()` first — a trailing space or inconsistent casing (as in the worked example's `"Status "`) is a very common, hard-to-spot cause of this error.

**4. Confusing `axis=0` (rows) and `axis=1` (columns) in `.drop()`**
```python
df.drop("total", axis=0)     # tries to drop a ROW labeled "total" — probably not what you meant
df.drop("total", axis=1)       # drops the COLUMN named "total"
df.drop(columns=["total"])       # equivalent to axis=1, and much clearer to read — prefer this form
```
Fix: prefer the explicit `columns=`/`index=` keyword arguments over relying on `axis=0`/`axis=1` — it removes any ambiguity about which dimension you're targeting, directly connecting to the same axis confusion flagged back in Lesson 3.5.

**5. Reordering columns by typing out every name manually, and missing one**
```python
df = df[["order_id", "status", "total"]]   # accidentally dropped customer_name — it's just GONE now
```
Fix: when you only want to move one or two columns without risking silently dropping others, build the new order programmatically (as shown in the Syntax Reference's "moving one column to the front" example) rather than retyping the full column list by hand.

**Data Engineer Tip:** Standardizing column names immediately after loading (lowercase, snake_case, no special characters) is a near-universal convention in data engineering — it matches typical SQL/warehouse naming conventions, avoids needing to quote column names in downstream SQL, and makes joining data from multiple sources far less error-prone when everyone's using the same naming style. Making this an automatic first step, as in the worked example, saves a surprising amount of friction later in a pipeline.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given a DataFrame with columns `["CustID", "Nm", "Amt"]`, rename them to `["customer_id", "name", "amount"]` using a dictionary with `.rename()`.

<details>
<summary>Hint</summary>
`df.rename(columns={"CustID": "customer_id", "Nm": "name", "Amt": "amount"})`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"CustID": [1, 2], "Nm": ["A", "B"], "Amt": [10, 20]})
df = df.rename(columns={"CustID": "customer_id", "Nm": "name", "Amt": "amount"})
print(df.columns.tolist())
```
</details>

---

**Exercise 2 (Easy):** Given a DataFrame with a column `internal_id` you don't need, drop it and print the remaining columns.

<details>
<summary>Hint</summary>
`df.drop(columns=["internal_id"])`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"id": [1, 2], "internal_id": [101, 102], "name": ["A", "B"]})
df = df.drop(columns=["internal_id"])
print(df.columns.tolist())   # ['id', 'name']
```
</details>

---

**Exercise 3 (Medium):** Given a DataFrame with messy column names `["Order ID", "Customer Name", "Order Total "]` (note spaces, inconsistent casing, and a trailing space), standardize all of them to lowercase snake_case in one bulk operation.

<details>
<summary>Hint</summary>
`df.columns = df.columns.str.strip().str.lower().str.replace(" ", "_")`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"Order ID": [1], "Customer Name": ["A"], "Order Total ": [10]})
df.columns = df.columns.str.strip().str.lower().str.replace(" ", "_")
print(df.columns.tolist())   # ['order_id', 'customer_name', 'order_total']
```
</details>

---

**Exercise 4 (Medium):** Given a DataFrame with columns in the order `["total", "customer", "order_id", "status"]`, reorder them to `["order_id", "customer", "status", "total"]` by explicit column selection.

<details>
<summary>Hint</summary>
`df[["order_id", "customer", "status", "total"]]`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({
    "total": [45.99, 120.50],
    "customer": ["Maria Chen", "James Lee"],
    "order_id": [1, 2],
    "status": ["shipped", "pending"],
})

df = df[["order_id", "customer", "status", "total"]]
print(df.columns.tolist())
```
</details>

---

**Exercise 5 (Hard):** Given this raw DataFrame:
```python
raw = pd.DataFrame({
    "OrderNum": [1, 2, 3],
    "cust": ["Maria Chen", "James Lee", "Priya Patel"],
    "Total Amt": [45.99, 120.50, 89.99],
    "notes_internal": ["", "urgent", ""],
    "src_system_id": ["A1", "A2", "A3"],
    "Status": ["shipped", "pending", "shipped"],
})
```
Perform a full column cleanup: (1) standardize all column names to snake_case/lowercase, (2) rename `ordernum` to `order_id` and `cust` to `customer_name` specifically, (3) drop `notes_internal` and `src_system_id` since they're not needed for this analysis, (4) reorder the final columns to `["order_id", "customer_name", "status", "total_amt"]`. Print the final `.columns.tolist()` and the full cleaned DataFrame.

<details>
<summary>Hint</summary>
Do the bulk standardization first (this will turn "Total Amt" into "total_amt" and "Status" into "status" automatically), then the targeted rename, then the drop, then the reorder — in that order, since each step depends on the column names produced by the previous one.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

raw = pd.DataFrame({
    "OrderNum": [1, 2, 3],
    "cust": ["Maria Chen", "James Lee", "Priya Patel"],
    "Total Amt": [45.99, 120.50, 89.99],
    "notes_internal": ["", "urgent", ""],
    "src_system_id": ["A1", "A2", "A3"],
    "Status": ["shipped", "pending", "shipped"],
})

cleaned = raw.copy()

# Step 1: bulk standardize
cleaned.columns = cleaned.columns.str.strip().str.lower().str.replace(" ", "_")
print(cleaned.columns.tolist())
# ['ordernum', 'cust', 'total_amt', 'notes_internal', 'src_system_id', 'status']

# Step 2: targeted renames
cleaned = cleaned.rename(columns={"ordernum": "order_id", "cust": "customer_name"})

# Step 3: drop unneeded columns
cleaned = cleaned.drop(columns=["notes_internal", "src_system_id"])

# Step 4: reorder
cleaned = cleaned[["order_id", "customer_name", "status", "total_amt"]]

print(cleaned.columns.tolist())
# ['order_id', 'customer_name', 'status', 'total_amt']
print(cleaned)
#    order_id customer_name   status  total_amt
# 0         1    Maria Chen  shipped      45.99
# 1         2     James Lee  pending     120.50
# 2         3   Priya Patel  shipped      89.99
```
</details>

---

**Previous lesson:** [4.6 — Data Type Conversion & Cleaning](lesson-4.6-type-conversion-and-cleaning.md)
**Next lesson:** [4.8 — Detecting & Handling Duplicates](lesson-4.8-duplicates.md)
