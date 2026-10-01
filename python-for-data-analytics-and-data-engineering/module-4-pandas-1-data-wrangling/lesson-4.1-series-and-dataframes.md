# Lesson 4.1 — Series & DataFrame Fundamentals

## CONCEPT

pandas has two core data structures:

- **`Series`** — a single labeled, one-dimensional array. Think of it as one column of a spreadsheet, with an index (row label) attached to each value.
- **`DataFrame`** — a two-dimensional table made up of multiple `Series` sharing a common index. Think of it as the entire spreadsheet, or the result of a SQL query.

```python
import pandas as pd

prices = pd.Series([24.99, 12.50, 89.00], name="price")
print(prices)
# 0    24.99
# 1    12.50
# 2    89.00
# Name: price, dtype: float64

df = pd.DataFrame({
    "name": ["Mouse", "Cable", "Lamp"],
    "price": [24.99, 12.50, 34.00],
})
```

Every DataFrame column is a `Series`, and every DataFrame has an **index** — the labels along the rows (by default, `0, 1, 2, ...`, but it can be set to anything, like an `order_id`).

## WHY IT MATTERS

Nearly everything in data analytics work happens inside a DataFrame — it's the direct pandas equivalent of a SQL result set or an Excel sheet, and it's what every other library (matplotlib, scikit-learn) expects as input. Understanding that a DataFrame is really "a dictionary of NumPy arrays, all sharing row labels" demystifies a huge amount of pandas behavior, and directly connects back to the dictionary-of-lists and NumPy array concepts from Modules 2–3.

## SYNTAX REFERENCE

```python
import pandas as pd

# Creating a Series
pd.Series([1, 2, 3])                        # default index: 0, 1, 2
pd.Series([1, 2, 3], index=["a", "b", "c"])   # custom index
pd.Series({"a": 1, "b": 2, "c": 3})            # from a dict — keys become the index

# Creating a DataFrame
pd.DataFrame({
    "name": ["Mouse", "Cable"],
    "price": [24.99, 12.50],
})

pd.DataFrame([
    {"name": "Mouse", "price": 24.99},   # list of dicts — one dict per row
    {"name": "Cable", "price": 12.50},
])

# Core attributes
df.shape          # (rows, columns) — e.g. (3, 2)
df.columns          # Index(['name', 'price'], dtype='object')
df.index              # RangeIndex(start=0, stop=3, step=1)
df.dtypes               # dtype of each column
df.values                 # the underlying data as a raw NumPy array

# Accessing a single column (returns a Series)
df["price"]
df.price              # also works, but avoid if the column name has spaces or matches a method name

# Accessing multiple columns (returns a DataFrame)
df[["name", "price"]]

# Setting a custom index
df.set_index("name")            # returns a new DataFrame with "name" as the index
df.set_index("name", inplace=True)   # modifies df directly
df.reset_index()                  # moves the index back to a regular column

# Basic info
len(df)              # number of rows
df.columns.tolist()    # column names as a plain list
```

## WORKED EXAMPLE

Building a small products DataFrame from scratch and exploring its structure — the exact shape of the `ecommerce.products` table:

```python
import pandas as pd

products = pd.DataFrame({
    "product_id": [1042, 1043, 1044, 1045],
    "name": ["Wireless Mouse", "USB-C Cable", "Desk Lamp", "Keyboard"],
    "category": ["Electronics", "Electronics", "Home", "Electronics"],
    "price": [24.99, 12.50, 34.00, 59.99],
    "in_stock": [True, True, False, True],
})

print(products)
#    product_id            name     category  price  in_stock
# 0        1042  Wireless Mouse  Electronics  24.99      True
# 1        1043     USB-C Cable  Electronics  12.50      True
# 2        1044       Desk Lamp         Home  34.00     False
# 3        1045        Keyboard  Electronics  59.99      True

# Shape and structure
print(products.shape)      # (4, 5) — 4 rows, 5 columns
print(products.dtypes)
# product_id      int64
# name           object
# category       object
# price         float64
# in_stock         bool
# dtype: object

# Access a single column — this is a Series
price_series = products["price"]
print(type(price_series))    # <class 'pandas.core.series.Series'>
print(price_series.mean())    # 32.87 — Series have the same aggregation methods as NumPy arrays

# Access multiple columns — this is a DataFrame
subset = products[["name", "price"]]
print(type(subset))           # <class 'pandas.core.frame.DataFrame'>

# Set product_id as the index — makes lookups by ID more natural
products_indexed = products.set_index("product_id")
print(products_indexed.loc[1043])
# name        USB-C Cable
# category    Electronics
# price             12.5
# in_stock          True
# Name: 1043, dtype: object

# The underlying NumPy array (dtype becomes 'object' when columns are mixed types)
print(products.values[0])   # [1042 'Wireless Mouse' 'Electronics' 24.99 True]
```

## COMMON MISTAKES

**1. Confusing a single-bracket selection (`Series`) with double-bracket (`DataFrame`)**
```python
products["price"]         # Series — 1D
products[["price"]]        # DataFrame — 2D, with just one column
```
This distinction matters a lot in later lessons (Lesson 4.4 especially) — many operations behave differently depending on whether you're working with a Series or a single-column DataFrame.

**2. Using `df.column_name` when the column name isn't a valid Python identifier**
```python
df.order total    # SyntaxError — spaces aren't allowed in attribute access
df["order total"]  # works fine — bracket access handles any string
```
Fix: prefer bracket notation (`df["column"]`) consistently — it always works, while dot notation only works for simple, space-free names that don't collide with existing DataFrame methods (e.g. `df.count` would return the *method*, not a column literally named "count").

**3. Forgetting most DataFrame operations return a NEW object rather than modifying in place**
```python
df.set_index("name")          # returns a new DataFrame; df itself is UNCHANGED
print(df.index)                # still the original default index!
```
Fix: either reassign the result (`df = df.set_index("name")`) or pass `inplace=True` — though many pandas style guides now discourage `inplace=True` in favor of reassignment, since it's more explicit and chains more predictably.

**4. Assuming the index is always `0, 1, 2, ...`**
Once you've filtered, sorted, or set a custom index, row labels can become non-sequential or non-numeric. Relying on positional assumptions (like "row 3 is always the fourth row") breaks the moment the DataFrame has been reshaped — use `.iloc` (Lesson 4.4) when you specifically need positional access regardless of the index's current labels.

**5. Building a DataFrame from mismatched-length lists**
```python
pd.DataFrame({"a": [1, 2, 3], "b": [1, 2]})   # ValueError — All arrays must be of the same length
```
Fix: every column's list/array must have the same length — this mirrors the "same shape" requirement from NumPy broadcasting in Module 3.

**Data Engineer Tip:** `df.dtypes` is one of the very first things worth checking after loading any new dataset — a numeric column that shows up as `object` (pandas' catch-all type, usually meaning "text" or "mixed") is an immediate signal that something didn't parse as expected, often due to a stray non-numeric value hiding somewhere in that column. Making a habit of checking `.dtypes` right after loading data catches a large share of data quality issues before they propagate further into an analysis.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create a `Series` of 5 prices of your choosing, and print its `.mean()` and `.max()`.

<details>
<summary>Hint</summary>
`pd.Series([...])`, then `.mean()` and `.max()` — same methods as NumPy arrays.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

prices = pd.Series([24.99, 12.50, 89.00, 34.00, 15.75])
print(prices.mean())   # 35.248
print(prices.max())     # 89.0
```
</details>

---

**Exercise 2 (Easy):** Create a DataFrame with columns `name` and `age` for 3 people of your choosing, and print its `.shape` and `.dtypes`.

<details>
<summary>Hint</summary>
`pd.DataFrame({"name": [...], "age": [...]})`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

people = pd.DataFrame({
    "name": ["Maria", "James", "Priya"],
    "age": [29, 35, 41],
})
print(people.shape)    # (3, 2)
print(people.dtypes)
```
</details>

---

**Exercise 3 (Medium):** Given the `products` DataFrame from the worked example (recreate it), select just the `name` and `category` columns as a new DataFrame, and confirm the result's `type()` is a DataFrame, not a Series.

<details>
<summary>Hint</summary>
Use double brackets: `products[["name", "category"]]`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

products = pd.DataFrame({
    "product_id": [1042, 1043, 1044],
    "name": ["Wireless Mouse", "USB-C Cable", "Desk Lamp"],
    "category": ["Electronics", "Electronics", "Home"],
    "price": [24.99, 12.50, 34.00],
})

subset = products[["name", "category"]]
print(subset)
print(type(subset))   # <class 'pandas.core.frame.DataFrame'>
```
</details>

---

**Exercise 4 (Medium):** Using the same `products` DataFrame, set `product_id` as the index, then look up and print the full row for `product_id = 1043` using `.loc[]`.

<details>
<summary>Hint</summary>
`products.set_index("product_id")`, then `.loc[1043]`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

products = pd.DataFrame({
    "product_id": [1042, 1043, 1044],
    "name": ["Wireless Mouse", "USB-C Cable", "Desk Lamp"],
    "category": ["Electronics", "Electronics", "Home"],
    "price": [24.99, 12.50, 34.00],
})

products_indexed = products.set_index("product_id")
print(products_indexed.loc[1043])
```
</details>

---

**Exercise 5 (Hard):** Build a DataFrame representing 5 orders with columns `order_id`, `customer_name`, and `total`. Then: (1) extract the `total` column as a Series and compute its sum, mean, and standard deviation, (2) create a *new* Series (not added to the DataFrame yet) representing a 10% tax on each total, using vectorized Series arithmetic (no loop), and (3) add this new Series as a column called `tax` to the original DataFrame, then print the full updated DataFrame.

<details>
<summary>Hint</summary>
Series arithmetic works exactly like NumPy array arithmetic (Module 3) — `df["total"] * 0.10` produces a new Series directly. Assigning it to `df["tax"] = ...` adds it as a new column.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5],
    "customer_name": ["Maria Chen", "James Lee", "Priya Patel", "Maria Chen", "Sam Osei"],
    "total": [45.99, 120.50, 89.99, 15.00, 200.00],
})

totals = orders["total"]
print("Sum:", totals.sum())         # 471.48
print("Mean:", totals.mean())        # 94.296
print("Std:", totals.std())           # ~73.4

tax = totals * 0.10
orders["tax"] = tax

print(orders)
#    order_id customer_name   total     tax
# 0         1    Maria Chen   45.99   4.599
# 1         2     James Lee  120.50  12.050
# 2         3   Priya Patel   89.99   8.999
# 3         4    Maria Chen   15.00   1.500
# 4         5      Sam Osei  200.00  20.000
```
</details>

---

**Next lesson:** [4.2 — Reading Data: CSV, JSON, Excel, SQL](lesson-4.2-reading-data.md)
