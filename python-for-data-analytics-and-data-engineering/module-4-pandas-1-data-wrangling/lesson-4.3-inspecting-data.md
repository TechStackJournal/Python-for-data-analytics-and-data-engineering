# Lesson 4.3 — Inspecting Data

## CONCEPT

Before cleaning or analyzing any dataset, you need to understand its shape, structure, and quality: how many rows, what columns exist, what types they are, whether anything is missing, and what the values actually look like. pandas provides a small set of methods that should become an automatic first step every time you load new data.

```python
import pandas as pd

df = pd.read_csv("orders.csv")
df.head()        # first 5 rows — a quick visual sanity check
df.info()          # column names, types, non-null counts, memory usage
df.describe()        # statistical summary of numeric columns
```

## WHY IT MATTERS

Skipping this step is one of the most common mistakes beginners make — jumping straight into analysis on a dataset you haven't actually looked at. A five-second `.info()` call catches an enormous share of problems before they cause confusing downstream errors: wrong types, unexpected nulls, a column that didn't parse the way you expected. This is the pandas equivalent of the "always check `.dtypes` after loading" habit from Lesson 4.2, expanded into a full first-look routine.

## SYNTAX REFERENCE

```python
import pandas as pd

df = pd.read_csv("orders.csv")

# Quick previews
df.head()             # first 5 rows (default)
df.head(10)             # first 10 rows
df.tail()                 # last 5 rows
df.sample(5)                # 5 RANDOM rows — useful for spotting issues head() might miss

# Structural overview
df.shape             # (rows, columns)
df.columns             # column names
df.dtypes                # dtype per column
df.info()                  # combined summary: dtypes, non-null counts, memory usage

# Statistical summary
df.describe()              # count, mean, std, min, 25/50/75%, max — numeric columns by default
df.describe(include="all")   # includes non-numeric columns too (counts, unique, top, freq)
df.describe(include="object")  # ONLY the text/object columns

# Missing data overview
df.isna().sum()             # count of missing values PER COLUMN
df.isna().sum().sum()         # total missing values across the whole DataFrame
df.isna().mean() * 100          # percentage missing per column

# Unique values
df["category"].unique()        # array of distinct values in a column
df["category"].nunique()         # COUNT of distinct values
df["category"].value_counts()      # frequency of each distinct value, sorted descending

# Checking for duplicates (full detail in Lesson 4.8)
df.duplicated().sum()          # count of fully duplicate rows
```

## WORKED EXAMPLE

A full first-look inspection routine on a freshly-loaded orders dataset:

```python
import pandas as pd
import numpy as np

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5, 6],
    "customer": ["Maria Chen", "James Lee", "Priya Patel", "Maria Chen", None, "Sam Osei"],
    "total": [45.99, 120.50, np.nan, 15.00, 200.00, 89.99],
    "status": ["shipped", "pending", "shipped", "cancelled", "shipped", "shipped"],
    "order_date": pd.to_datetime(["2024-11-01", "2024-11-05", "2024-11-10",
                                    "2024-11-12", "2024-11-15", "2024-11-18"]),
})

# 1. Quick visual check
print(orders.head(3))
#    order_id     customer   total     status order_date
# 0         1   Maria Chen   45.99    shipped 2024-11-01
# 1         2    James Lee  120.50    pending 2024-11-05
# 2         3  Priya Patel     NaN    shipped 2024-11-10

# 2. Full structural summary
orders.info()
# <class 'pandas.core.frame.DataFrame'>
# RangeIndex: 6 entries, 0 to 5
# Data columns (total 5 columns):
#  #   Column      Non-Null Count  Dtype
# ---  ------      --------------  -----
#  0   order_id    6 non-null      int64
#  1   customer    5 non-null      object      <- 1 missing!
#  2   total       5 non-null      float64     <- 1 missing!
#  3   status      6 non-null      object
#  4   order_date  6 non-null      datetime64[ns]

# 3. Statistical summary of numeric columns
print(orders.describe())
#        order_id       total
# count  6.000000    5.000000
# mean   3.500000   94.296000
# std    1.870829   71.906...
# min    1.000000   15.000000
# 25%    2.250000   45.990000
# 50%    3.500000   89.990000
# 75%    4.750000  140.375000
# max    6.000000  200.000000

# 4. Missing data check — pinpoint exactly what's missing
print(orders.isna().sum())
# order_id      0
# customer      1
# total         1
# status        0
# order_date    0

# 5. Explore a categorical column's distinct values and frequency
print(orders["status"].value_counts())
# shipped      4
# pending      1
# cancelled    1
```
Five function calls, and you already know: 6 rows, 1 missing customer name, 1 missing total, order values ranging $15–$200 with a median around $90, and the status breakdown. This is the routine to run on every new dataset before doing anything else with it.

## COMMON MISTAKES

**1. Skipping inspection and jumping straight to analysis**
Running `.mean()` on a column without first checking `.info()` means you might not notice it's secretly an `object` (text) column, or that 30% of it is missing — silently producing wrong or `NaN` results. Fix: make `.info()` and `.head()` an automatic reflex immediately after loading any new dataset, every time.

**2. Only using `.head()` and never checking a random sample**
```python
df.head()   # the first 5 rows might all look clean, by coincidence
```
Real-world data quality issues are sometimes concentrated later in a file (e.g., a batch of bad records appended at the end). Fix: occasionally use `.sample(10)` for a random spot-check, especially on larger datasets.

**3. Assuming `.describe()` covers every column by default**
```python
df.describe()   # by default, ONLY numeric columns are summarized — text columns are silently excluded
```
Fix: use `.describe(include="all")` if you want a summary of every column, or `.describe(include="object")` specifically for text columns (which shows count, unique values, the most frequent value, and its frequency instead of mean/std).

**4. Misreading `.info()`'s "non-null count" as the total row count**
```python
# 5   total       5 non-null      float64
```
This means 5 out of however-many total rows are non-null — always cross-check against the DataFrame's actual total row count (shown at the top: "RangeIndex: 6 entries") to know how many are actually missing.

**5. Confusing `.unique()` (the values) with `.nunique()` (the count)**
```python
df["category"].unique()     # array(['Electronics', 'Home', 'Outdoor'], dtype=object)
df["category"].nunique()      # 3 — just the count
```
Both are useful for different purposes — `.unique()` to see *what* the distinct values actually are (useful for spotting inconsistent casing or typos), `.nunique()` when you just need the count.

**Data Engineer Tip:** This exact inspection routine — shape, dtypes, missing counts, value distributions — is the manual version of an automated data quality check in a production pipeline. Tools like Great Expectations or Pandera (previewed briefly in Module 9) essentially codify these same checks (`.isna().sum()`, `.dtypes`, `.duplicated().sum()`) into automated assertions that run on every pipeline execution, failing loudly if something looks different from what's expected — the habit you're building manually here is exactly what that automation formalizes later.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given any DataFrame of your choosing with at least 4 columns, print its `.shape`, `.columns`, and `.dtypes`.

<details>
<summary>Hint</summary>
Straightforward attribute access on any DataFrame.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({
    "id": [1, 2, 3],
    "name": ["A", "B", "C"],
    "price": [9.99, 19.99, 29.99],
    "in_stock": [True, False, True],
})

print(df.shape)
print(df.columns)
print(df.dtypes)
```
</details>

---

**Exercise 2 (Easy):** Given a DataFrame with a `category` column containing some repeated values, print the distinct values and how many times each appears.

<details>
<summary>Hint</summary>
`.unique()` for distinct values, `.value_counts()` for frequency.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({
    "category": ["Electronics", "Home", "Electronics", "Outdoor", "Home", "Electronics"],
})

print(df["category"].unique())
print(df["category"].value_counts())
```
</details>

---

**Exercise 3 (Medium):** Given a DataFrame with some missing values (build one with `np.nan` in at least 2 columns), print how many missing values exist per column, and what percentage of each column is missing.

<details>
<summary>Hint</summary>
`.isna().sum()` and `.isna().mean() * 100`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    "id": [1, 2, 3, 4],
    "price": [9.99, np.nan, 29.99, np.nan],
    "name": ["A", "B", None, "D"],
})

print(df.isna().sum())
print(df.isna().mean() * 100)
```
</details>

---

**Exercise 4 (Medium):** Given a DataFrame with both numeric and text columns, use `.describe(include="all")` to get a summary of every column, and print it. Identify (in a comment) which summary statistics only apply to numeric columns and which only apply to text columns.

<details>
<summary>Hint</summary>
`.describe(include="all")` — numeric columns get mean/std/percentiles; text columns get unique/top/freq instead; both share `count`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({
    "id": [1, 2, 3, 4],
    "price": [9.99, 19.99, 9.99, 29.99],
    "category": ["Electronics", "Home", "Electronics", "Outdoor"],
})

print(df.describe(include="all"))

# mean, std, min, 25%/50%/75%, max apply only to numeric columns (id, price)
# unique, top, freq apply only to text/object columns (category)
# count applies to every column
```
</details>

---

**Exercise 5 (Hard):** Given this DataFrame simulating a freshly-loaded, imperfect dataset:
```python
import numpy as np

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5, 6, 7],
    "customer": ["Maria Chen", "James Lee", None, "Priya Patel", "James Lee", "Sam Osei", None],
    "total": [45.99, np.nan, 89.99, 15.00, 120.50, np.nan, 65.00],
    "status": ["shipped", "shipped", "pending", "cancelled", "shipped", "shipped", "pending"],
})
```
Write a short "data quality report" that prints: (1) total row count, (2) missing value count and percentage for every column, (3) the number of distinct customers (ignoring missing values), and (4) the full value_counts breakdown of `status`. Format the output with clear labels.

<details>
<summary>Hint</summary>
Combine `len(df)`, `.isna().sum()`, `.isna().mean() * 100`, `["customer"].nunique()` (this automatically excludes NaN by default), and `["status"].value_counts()`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import numpy as np

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5, 6, 7],
    "customer": ["Maria Chen", "James Lee", None, "Priya Patel", "James Lee", "Sam Osei", None],
    "total": [45.99, np.nan, 89.99, 15.00, 120.50, np.nan, 65.00],
    "status": ["shipped", "shipped", "pending", "cancelled", "shipped", "shipped", "pending"],
})

print("=== DATA QUALITY REPORT ===")
print(f"Total rows: {len(orders)}")

print("\nMissing values per column:")
missing_counts = orders.isna().sum()
missing_pcts = orders.isna().mean() * 100
for col in orders.columns:
    print(f"  {col}: {missing_counts[col]} missing ({missing_pcts[col]:.1f}%)")

print(f"\nDistinct customers: {orders['customer'].nunique()}")

print("\nStatus breakdown:")
print(orders["status"].value_counts())
```
</details>

---

**Previous lesson:** [4.2 — Reading Data: CSV, JSON, Excel, SQL](lesson-4.2-reading-data.md)
**Next lesson:** [4.4 — Selecting & Filtering](lesson-4.4-selecting-and-filtering.md)
