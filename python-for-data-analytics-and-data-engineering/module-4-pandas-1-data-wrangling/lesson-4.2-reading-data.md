# Lesson 4.2 — Reading Data: CSV, JSON, Excel, SQL

## CONCEPT

pandas provides a dedicated `read_*()` function for nearly every common data format, each returning a DataFrame directly — replacing the manual `csv`/`json` module work from Module 2 with a single function call that handles headers, type inference, and edge cases automatically.

```python
import pandas as pd

df = pd.read_csv("orders.csv")
df = pd.read_json("orders.json")
df = pd.read_excel("orders.xlsx")
df = pd.read_sql("SELECT * FROM orders", connection)   # previewed here, full detail in Module 7
```

## WHY IT MATTERS

This is the very first step of nearly every real analysis: getting data *into* a DataFrame in the first place. Knowing exactly which `read_*()` function to reach for, and which parameters matter (delimiter, encoding, sheet name, date parsing), saves significant time and avoids a class of subtle bugs — a mis-parsed column on load quietly propagates through everything downstream.

## SYNTAX REFERENCE

```python
import pandas as pd

# CSV
pd.read_csv("orders.csv")
pd.read_csv("orders.csv", sep=";")                    # custom delimiter
pd.read_csv("orders.csv", usecols=["id", "total"])      # load only specific columns
pd.read_csv("orders.csv", dtype={"id": str})              # force a specific dtype on load
pd.read_csv("orders.csv", parse_dates=["order_date"])       # parse a column as datetime directly
pd.read_csv("orders.csv", na_values=["N/A", "missing"])       # treat these strings as NaN
pd.read_csv("orders.csv", nrows=100)                             # only read the first 100 rows

# JSON
pd.read_json("orders.json")                    # expects a JSON array of records by default
pd.read_json("orders.json", orient="records")     # explicit about the JSON's shape

# For deeply nested JSON, flatten it with json_normalize
import json
with open("orders.json") as f:
    raw = json.load(f)
pd.json_normalize(raw)                            # flattens nested dicts into flat columns
pd.json_normalize(raw, record_path="items", meta=["order_id"])   # flatten a nested LIST field

# Excel
pd.read_excel("orders.xlsx")                        # reads the first sheet by default
pd.read_excel("orders.xlsx", sheet_name="Orders")      # a specific sheet by name
pd.read_excel("orders.xlsx", sheet_name=None)            # ALL sheets, as a dict of DataFrames

# SQL (requires a database connection — previewed here, full detail in Module 7)
import sqlalchemy
engine = sqlalchemy.create_engine("postgresql://user:pass@host/dbname")
pd.read_sql("SELECT * FROM orders", engine)
pd.read_sql_table("orders", engine)                # reads an entire table directly by name

# Writing DataFrames back out (the reverse operation)
df.to_csv("output.csv", index=False)                 # index=False avoids writing the row index as a column
df.to_json("output.json", orient="records", indent=2)
df.to_excel("output.xlsx", index=False)
```

## WORKED EXAMPLE

Loading the same order data from three different formats, and confirming they produce equivalent DataFrames:

```python
import pandas as pd
import json

# --- Setup: create sample files to read back (simulating pre-existing data) ---
orders_data = [
    {"order_id": 1, "customer": "Maria Chen", "total": 45.99, "order_date": "2024-11-01"},
    {"order_id": 2, "customer": "James Lee", "total": 120.50, "order_date": "2024-11-05"},
    {"order_id": 3, "customer": "Priya Patel", "total": 89.99, "order_date": "2024-11-10"},
]

pd.DataFrame(orders_data).to_csv("orders.csv", index=False)

with open("orders.json", "w") as f:
    json.dump(orders_data, f)

pd.DataFrame(orders_data).to_excel("orders.xlsx", index=False)

# --- Reading each format back in ---
df_csv = pd.read_csv("orders.csv", parse_dates=["order_date"])
df_json = pd.read_json("orders.json")
df_excel = pd.read_excel("orders.xlsx")

print(df_csv.dtypes)
# order_id               int64
# customer              object
# total                float64
# order_date    datetime64[ns]
# dtype: object

print(df_csv)
#    order_id     customer   total order_date
# 0         1   Maria Chen   45.99 2024-11-01
# 1         2    James Lee  120.50 2024-11-05
# 2         3  Priya Patel   89.99 2024-11-10

# Loading a nested JSON structure and flattening it with json_normalize
nested_orders = [
    {
        "order_id": 501,
        "customer": {"name": "Maria Chen", "tier": "gold"},
        "total": 62.48,
    },
    {
        "order_id": 502,
        "customer": {"name": "James Lee", "tier": "silver"},
        "total": 34.00,
    },
]

flattened = pd.json_normalize(nested_orders)
print(flattened)
#    order_id  total customer.name customer.tier
# 0       501  62.48    Maria Chen          gold
# 1       502  34.00     James Lee        silver
# Note how the nested "customer" dict became two flat columns automatically,
# named with dot notation — exactly the manual flattening pattern from
# Lesson 2.2, done here in a single function call
```

## COMMON MISTAKES

**1. Not parsing dates on load, leaving date columns as plain text**
```python
df = pd.read_csv("orders.csv")
df["order_date"].dtype   # object (text) — NOT datetime, even though it looks like a date!
```
Fix: use `parse_dates=["order_date"]` at load time, or convert afterward with `pd.to_datetime()` (Lesson 4.6) — a date column left as text can't be filtered by range or have date arithmetic applied to it.

**2. Forgetting `index=False` when writing to CSV/Excel**
```python
df.to_csv("output.csv")   # writes an EXTRA unnamed column containing the pandas index
```
Fix: `index=False` is almost always what you want unless you specifically gave the DataFrame a meaningful custom index you intend to preserve.

**3. Assuming `read_json()` handles any JSON shape automatically**
Deeply nested JSON (dicts inside dicts, lists inside records) often doesn't flatten cleanly with plain `read_json()`. Fix: for anything beyond a flat array of simple records, load with the `json` module first (Lesson 2.6), then use `pd.json_normalize()`, which gives you explicit control over how nested structures get flattened.

**4. Not specifying `sheet_name` and being surprised which sheet loaded**
```python
pd.read_excel("report.xlsx")   # silently loads only the FIRST sheet — might not be the one you wanted
```
Fix: always check `sheet_name=None` first to see all available sheet names (returns a dict keyed by sheet name) if you're not certain of the file's structure, then load the specific one you need.

**5. Silent encoding issues with CSV files from non-standard sources**
```python
pd.read_csv("export.csv")   # UnicodeDecodeError, or garbled special characters
```
Fix: try `pd.read_csv("export.csv", encoding="latin-1")` (or `"cp1252"`) if the default UTF-8 encoding fails — this is a common issue with exports from older systems or non-English locales.

**Data Engineer Tip:** In a real pipeline, the "reading" step is where you make your first and most important data quality decisions: which columns to load (`usecols` — smaller memory footprint), what counts as missing (`na_values`), and what types to enforce upfront (`dtype`). Catching a type or format problem *at load time* with an explicit parameter is far cheaper than discovering it three transformation steps later — this is the same "validate early" principle from Lesson 1.1, now applied at the dataset level instead of a single value.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create a small DataFrame with 3 rows of your choosing, write it to `test.csv` with `to_csv(index=False)`, then read it back in and print it.

<details>
<summary>Hint</summary>
`.to_csv("test.csv", index=False)`, then `pd.read_csv("test.csv")`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"id": [1, 2, 3], "name": ["A", "B", "C"]})
df.to_csv("test.csv", index=False)

loaded = pd.read_csv("test.csv")
print(loaded)
```
</details>

---

**Exercise 2 (Easy):** Read a CSV containing a date column (create `orders.csv` with an `order_date` column of your own values first), using `parse_dates` to ensure it loads as a proper datetime. Print `.dtypes` to confirm.

<details>
<summary>Hint</summary>
`pd.read_csv("orders.csv", parse_dates=["order_date"])`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

pd.DataFrame({
    "id": [1, 2],
    "order_date": ["2024-01-15", "2024-02-20"],
}).to_csv("orders.csv", index=False)

df = pd.read_csv("orders.csv", parse_dates=["order_date"])
print(df.dtypes)
# id                    int64
# order_date    datetime64[ns]
```
</details>

---

**Exercise 3 (Medium):** Given a JSON file containing a list of 3 product records (create it with the `json` module first), read it into a DataFrame with `pd.read_json()` and print the result's shape and columns.

<details>
<summary>Hint</summary>
Write with `json.dump()`, read with `pd.read_json()`.
</details>

<details>
<summary>Solution</summary>

```python
import json
import pandas as pd

products = [
    {"id": 1, "name": "Mouse", "price": 24.99},
    {"id": 2, "name": "Cable", "price": 12.50},
    {"id": 3, "name": "Lamp", "price": 34.00},
]

with open("products.json", "w") as f:
    json.dump(products, f)

df = pd.read_json("products.json")
print(df.shape)      # (3, 3)
print(df.columns.tolist())   # ['id', 'name', 'price']
```
</details>

---

**Exercise 4 (Medium):** Given nested JSON data (a list of 2 orders, each with a nested `customer` dict containing `name` and `city`), use `pd.json_normalize()` to flatten it into a DataFrame, and print the resulting column names.

<details>
<summary>Hint</summary>
Build the nested list of dicts in Python directly, then `pd.json_normalize(data)`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

data = [
    {"order_id": 1, "customer": {"name": "Maria Chen", "city": "Toronto"}},
    {"order_id": 2, "customer": {"name": "James Lee", "city": "Vancouver"}},
]

df = pd.json_normalize(data)
print(df.columns.tolist())
# ['order_id', 'customer.name', 'customer.city']
print(df)
```
</details>

---

**Exercise 5 (Hard):** Create an Excel file with **two sheets** — `"Orders"` (columns `order_id`, `total`) and `"Customers"` (columns `customer_id`, `name`) — using `pd.ExcelWriter`. Then read the file back in two ways: (1) load only the `"Orders"` sheet directly by name, and (2) load *all* sheets at once using `sheet_name=None`, and print the keys of the resulting dictionary to confirm both sheet names are present.

<details>
<summary>Hint</summary>
Use `with pd.ExcelWriter("report.xlsx") as writer: df1.to_excel(writer, sheet_name="Orders", index=False); df2.to_excel(writer, sheet_name="Customers", index=False)`. For reading all sheets, `pd.read_excel("report.xlsx", sheet_name=None)` returns a dict of DataFrames keyed by sheet name.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({"order_id": [1, 2], "total": [45.99, 89.99]})
customers = pd.DataFrame({"customer_id": [1, 2], "name": ["Maria Chen", "James Lee"]})

with pd.ExcelWriter("report.xlsx") as writer:
    orders.to_excel(writer, sheet_name="Orders", index=False)
    customers.to_excel(writer, sheet_name="Customers", index=False)

# Load just one sheet
orders_only = pd.read_excel("report.xlsx", sheet_name="Orders")
print(orders_only)

# Load all sheets
all_sheets = pd.read_excel("report.xlsx", sheet_name=None)
print(list(all_sheets.keys()))   # ['Orders', 'Customers']
print(all_sheets["Customers"])
```
</details>

---

**Previous lesson:** [4.1 — Series & DataFrame Fundamentals](lesson-4.1-series-and-dataframes.md)
**Next lesson:** [4.3 — Inspecting Data](lesson-4.3-inspecting-data.md)
