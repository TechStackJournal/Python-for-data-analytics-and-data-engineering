# Lesson 4.6 — Data Type Conversion & Cleaning

## CONCEPT

Columns loaded from raw sources frequently have the wrong dtype — a price stored as text, a date stored as a string, a category with inconsistent casing. pandas provides dedicated conversion functions (`pd.to_numeric()`, `pd.to_datetime()`) plus the `.str` accessor for applying string cleaning across an entire column at once — the DataFrame-scale version of everything from Lesson 1.1 and Lesson 1.3.

```python
import pandas as pd

df["price"] = pd.to_numeric(df["price"])
df["order_date"] = pd.to_datetime(df["order_date"])
df["category"] = df["category"].str.strip().str.title()
```

## WHY IT MATTERS

This is, in practice, where most of the *time* in a real data analytics task goes — not the exciting analysis, but getting every column into the correct, consistent, trustworthy type first. A price column silently left as text can't be summed; a date column left as text can't be filtered by range; inconsistent category casing silently fragments what should be one group into several. Every technique in this lesson is a direct, vectorized scale-up of skills from Module 1 (type conversion) and Module 2 (string cleaning, date parsing).

## SYNTAX REFERENCE

```python
import pandas as pd

# Numeric conversion
pd.to_numeric(df["price"])                        # convert to numeric, errors by default if it can't
pd.to_numeric(df["price"], errors="coerce")          # invalid values become NaN instead of raising an error
df["price"].astype(float)                              # works if every value is ALREADY numeric-looking text
df["quantity"].astype(int)                                # convert to integer

# Date conversion
pd.to_datetime(df["order_date"])                    # auto-detects common date formats
pd.to_datetime(df["order_date"], format="%Y-%m-%d")    # explicit format — faster and safer than guessing
pd.to_datetime(df["order_date"], errors="coerce")        # unparseable dates become NaT ("Not a Time")

# The .str accessor — vectorized string methods, applied to an entire column
df["name"].str.strip()
df["name"].str.lower()
df["name"].str.upper()
df["name"].str.title()
df["name"].str.replace("$", "", regex=False)          # literal replacement
df["name"].str.replace(r"[^\d.]", "", regex=True)       # regex replacement (Lesson 2.8 syntax, vectorized)
df["name"].str.contains("Mouse")                          # boolean Series — for filtering
df["name"].str.startswith("SKU-")
df["name"].str.split("-")                                  # returns a Series of lists
df["name"].str.extract(r"(\d+)")                             # regex capture group extraction
df["name"].str.len()                                          # length of each string

# Boolean conversion from text
df["in_stock"] = df["in_stock_raw"].str.lower() == "true"

# Checking the result
df.dtypes
df["price"].dtype
```

## WORKED EXAMPLE

Cleaning a messy raw products DataFrame — exactly as it might arrive from a CSV export — into properly-typed columns:

```python
import pandas as pd

raw = pd.DataFrame({
    "name": ["  Wireless Mouse ", "USB-C CABLE", "desk lamp"],
    "price_text": ["$24.99", "USD 12.50", "34.00 dollars"],
    "order_date_text": ["2024-11-01", "2024-11-05", "2024-11-10"],
    "in_stock_text": ["true", "FALSE", "True"],
    "category": ["electronics", "Electronics", "HOME"],
})

cleaned = raw.copy()

# 1. Clean text formatting: strip whitespace, standardize casing
cleaned["name"] = cleaned["name"].str.strip().str.title()
cleaned["category"] = cleaned["category"].str.strip().str.title()

# 2. Extract numeric price from messy currency text using regex
cleaned["price"] = cleaned["price_text"].str.replace(r"[^\d.]", "", regex=True).astype(float)

# 3. Parse dates properly
cleaned["order_date"] = pd.to_datetime(cleaned["order_date_text"], format="%Y-%m-%d")

# 4. Convert text booleans to real booleans
cleaned["in_stock"] = cleaned["in_stock_text"].str.lower() == "true"

# 5. Drop the now-redundant raw text columns
cleaned = cleaned.drop(columns=["price_text", "order_date_text", "in_stock_text"])

print(cleaned)
#              name  category  price order_date  in_stock
# 0  Wireless Mouse  Electronics  24.99 2024-11-01      True
# 1     Usb-C Cable  Electronics  12.50 2024-11-05     False
# 2       Desk Lamp         Home  34.00 2024-11-10      True

print(cleaned.dtypes)
# name                  object
# category              object
# price                float64
# order_date    datetime64[ns]
# in_stock                bool

# Now math and date operations work correctly
print(f"Total inventory value: ${cleaned['price'].sum():.2f}")   # $71.49
print(cleaned[cleaned["order_date"] > "2024-11-03"])                # date range filtering works
```

## COMMON MISTAKES

**1. Using `.astype()` on text that isn't purely numeric yet**
```python
df["price_text"].astype(float)   # ValueError if any value still has "$" or letters in it
```
Fix: clean the text first (strip symbols with `.str.replace()`) before casting, or use `pd.to_numeric(..., errors="coerce")` to convert what it can and turn the rest into `NaN` rather than crashing the whole operation.

**2. Not using `errors="coerce"` and having one bad row crash an entire column conversion**
```python
pd.to_numeric(df["price"])   # ValueError on the FIRST bad value — stops everything
pd.to_numeric(df["price"], errors="coerce")   # bad values become NaN; the rest still converts
```
Fix: `errors="coerce"` is usually the safer default during initial cleaning — you can then use `.isna()` (Lesson 4.5) to find and investigate exactly which rows failed to parse, rather than losing the whole operation to one bad value.

**3. Forgetting `.str` methods operate on the whole Series and need reassignment**
```python
df["name"].str.strip()          # returns a NEW Series — df["name"] itself is unchanged!
df["name"] = df["name"].str.strip()   # correct — reassign
```
Same reassignment pattern as every other pandas method covered so far.

**4. Guessing a date format instead of specifying it explicitly**
```python
pd.to_datetime(df["date"])   # works for common formats, but can misinterpret ambiguous ones
                                # (e.g. "03/04/2024" — March 4th, or April 3rd?)
pd.to_datetime(df["date"], format="%m/%d/%Y")   # explicit — no ambiguity, and noticeably faster on large data
```
Fix: whenever you know the source format, specify it explicitly with `format=` — this mirrors the Lesson 2.4 warning about ambiguous date formats, now at DataFrame scale, where a wrong guess silently affects every row instead of just one value.

**5. Chaining too many `.str` operations without checking intermediate results**
```python
df["price"] = df["price_text"].str.replace(r"[^\d.]", "", regex=True).astype(float)
# if this fails, which step broke — the regex or the cast? Hard to tell from one line
```
Fix: while chaining is idiomatic pandas, when something isn't working, temporarily break the chain into separate lines/columns to inspect each intermediate step — it's much faster to isolate a bug that way than staring at one dense chained expression.

**Data Engineer Tip:** This lesson's cleaning routine — strip, standardize case, parse numeric/date types, convert booleans — is precisely the "transform" stage of an ETL pipeline (Module 9), and it's usually the largest single chunk of code in any real pipeline. Writing this as a sequence of clear, named, reusable steps (as in the worked example) rather than one dense chain makes it dramatically easier to test each piece independently and to explain to a reviewer exactly what cleaning decisions were made and why.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `df["price_text"] = ["19.99", "45.00", "12.50"]`, convert it to a proper float column named `price`, and print the resulting dtype.

<details>
<summary>Hint</summary>
`pd.to_numeric()` or `.astype(float)`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"price_text": ["19.99", "45.00", "12.50"]})
df["price"] = pd.to_numeric(df["price_text"])
print(df["price"].dtype)   # float64
```
</details>

---

**Exercise 2 (Easy):** Given `df["name"] = ["  mouse ", "CABLE", "Lamp "]`, clean it into a consistently title-cased, whitespace-trimmed column, and print the result.

<details>
<summary>Hint</summary>
`.str.strip().str.title()`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"name": ["  mouse ", "CABLE", "Lamp "]})
df["name"] = df["name"].str.strip().str.title()
print(df["name"].tolist())   # ['Mouse', 'Cable', 'Lamp']
```
</details>

---

**Exercise 3 (Medium):** Given `df["date_text"] = ["2024-01-15", "2024-02-20", "2024-03-25"]`, convert it to a proper datetime column, then filter to only rows after `"2024-02-01"`.

<details>
<summary>Hint</summary>
`pd.to_datetime(df["date_text"], format="%Y-%m-%d")`, then a boolean filter comparing against a date string.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"date_text": ["2024-01-15", "2024-02-20", "2024-03-25"]})
df["order_date"] = pd.to_datetime(df["date_text"], format="%Y-%m-%d")

recent = df[df["order_date"] > "2024-02-01"]
print(recent)
```
</details>

---

**Exercise 4 (Medium):** Given `df["price_text"] = ["$24.99", "USD 12.50", "not_available", "$89.00"]` (note one invalid entry), use `pd.to_numeric()` with `errors="coerce"` after stripping non-numeric characters with `.str.replace()`, and confirm the invalid entry became `NaN` rather than crashing the conversion.

<details>
<summary>Hint</summary>
Strip with a regex keeping only digits and the decimal point, then `pd.to_numeric(..., errors="coerce")`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"price_text": ["$24.99", "USD 12.50", "not_available", "$89.00"]})

stripped = df["price_text"].str.replace(r"[^\d.]", "", regex=True)
df["price"] = pd.to_numeric(stripped, errors="coerce")

print(df)
#        price_text  price
# 0          $24.99   24.99
# 1       USD 12.50   12.50
# 2   not_available     NaN
# 3          $89.00   89.00
```
</details>

---

**Exercise 5 (Hard):** Given this raw DataFrame:
```python
raw = pd.DataFrame({
    "sku": ["SKU-1042", "SKU-2093", "SKU-3301"],
    "price_text": ["$45.99", "$120.50", "$89.99"],
    "in_stock_text": ["YES", "no", "Yes"],
    "restock_date_text": ["2024-12-01", "2024-11-15", "2024-12-20"],
})
```
Write a cleaning routine that: (1) extracts the numeric SKU ID (the digits after `"SKU-"`) into a new integer column `sku_id` using `.str.extract()` (Lesson 2.8's regex capture-group syntax, vectorized), (2) converts `price_text` into a proper float column `price`, (3) converts `in_stock_text` into a proper boolean column `in_stock` (treating `"yes"` case-insensitively as `True`), (4) converts `restock_date_text` into a proper datetime column `restock_date`, and (5) drops all three original `_text`/raw columns, keeping only the cleaned ones (plus `sku`). Print the final cleaned DataFrame and its `.dtypes`.

<details>
<summary>Hint</summary>
`.str.extract(r"SKU-(\d+)")` returns a DataFrame-like single column of extracted strings — convert it with `.astype(int)`. For the boolean, `.str.lower() == "yes"`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

raw = pd.DataFrame({
    "sku": ["SKU-1042", "SKU-2093", "SKU-3301"],
    "price_text": ["$45.99", "$120.50", "$89.99"],
    "in_stock_text": ["YES", "no", "Yes"],
    "restock_date_text": ["2024-12-01", "2024-11-15", "2024-12-20"],
})

cleaned = raw.copy()

cleaned["sku_id"] = cleaned["sku"].str.extract(r"SKU-(\d+)").astype(int)
cleaned["price"] = cleaned["price_text"].str.replace(r"[^\d.]", "", regex=True).astype(float)
cleaned["in_stock"] = cleaned["in_stock_text"].str.lower() == "yes"
cleaned["restock_date"] = pd.to_datetime(cleaned["restock_date_text"], format="%Y-%m-%d")

cleaned = cleaned.drop(columns=["price_text", "in_stock_text", "restock_date_text"])

print(cleaned)
#         sku  sku_id  price  in_stock restock_date
# 0  SKU-1042    1042  45.99      True   2024-12-01
# 1  SKU-2093    2093 120.50     False   2024-11-15
# 2  SKU-3301    3301  89.99      True   2024-12-20

print(cleaned.dtypes)
```
</details>

---

**Previous lesson:** [4.5 — Handling Missing Data](lesson-4.5-missing-data.md)
**Next lesson:** [4.7 — Renaming, Reordering & Dropping Columns](lesson-4.7-column-management.md)
