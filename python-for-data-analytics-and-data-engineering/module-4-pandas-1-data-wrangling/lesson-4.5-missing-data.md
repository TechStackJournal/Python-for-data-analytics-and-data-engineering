# Lesson 4.5 — Handling Missing Data

## CONCEPT

pandas builds on NumPy's `NaN` (Lesson 3.6) as its standard representation of missing data, and adds a rich set of DataFrame-aware methods for detecting, removing, and filling it. The three core decisions you'll make repeatedly: **detect** (how much is missing, and where), **drop** (remove incomplete rows/columns), or **impute** (fill in a reasonable substitute value).

```python
import pandas as pd

df.isna()             # DataFrame of True/False, same shape as df
df.dropna()             # remove rows with ANY missing value
df.fillna(0)               # replace every missing value with 0
```

## WHY IT MATTERS

Missing data is the rule, not the exception, in real datasets — incomplete forms, failed sensor readings, optional fields, merge mismatches. How you handle it is a genuine analytical decision, not just a technical cleanup step: dropping rows loses information (and can bias your results if the missingness isn't random), while imputation introduces assumptions of its own. Module 5's aggregations and Module 9's data quality checks both depend on deliberate, documented choices made here — "I dropped 3% of rows with missing totals" is a decision worth recording, not silently making.

## SYNTAX REFERENCE

```python
import pandas as pd

# Detecting missing data
df.isna()                    # DataFrame of booleans, same shape as df
df.isna().sum()                # missing count per column
df.isna().sum().sum()            # total missing values in the whole DataFrame
df.isna().mean() * 100             # percentage missing per column
df["price"].isna()                   # missing mask for a single column
df[df["price"].isna()]                 # the actual ROWS where price is missing

# Dropping missing data
df.dropna()                      # drop any row with AT LEAST ONE missing value
df.dropna(subset=["price"])        # drop rows where SPECIFICALLY "price" is missing
df.dropna(how="all")                 # drop rows only if EVERY value in that row is missing
df.dropna(axis=1)                      # drop COLUMNS with any missing value, instead of rows
df.dropna(thresh=3)                      # keep rows with at least 3 non-missing values

# Filling (imputing) missing data
df.fillna(0)                          # replace every missing value with 0
df["price"].fillna(0)                   # fill just one column
df["price"].fillna(df["price"].mean())    # impute with the column's mean
df["price"].fillna(df["price"].median())    # impute with the median (often safer for skewed data)
df.fillna({"price": 0, "category": "Unknown"})   # different fill value per column
df["status"].fillna(method="ffill")           # forward-fill: carry the previous valid value forward
df["status"].fillna(method="bfill")             # backward-fill: use the NEXT valid value

# Reassigning the result (fillna/dropna return a NEW DataFrame by default)
df["price"] = df["price"].fillna(df["price"].mean())
```

## WORKED EXAMPLE

Auditing and handling missing data in an orders DataFrame, comparing drop vs. impute approaches:

```python
import pandas as pd
import numpy as np

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5, 6],
    "customer": ["Maria Chen", "James Lee", None, "Priya Patel", "Sam Osei", None],
    "total": [45.99, np.nan, 89.99, 15.00, np.nan, 65.00],
    "status": ["shipped", "shipped", "pending", "cancelled", "shipped", "pending"],
})

# 1. Detect: how much is missing, and where?
print(orders.isna().sum())
# order_id    0
# customer    2
# total       2
# status      0

missing_totals = orders[orders["total"].isna()]
print(missing_totals)
#    order_id customer  total  status
# 1         2 James Lee    NaN shipped
# 4         5 Sam Osei      NaN shipped

# 2. Option A: drop rows missing a total (safest if you can't reliably estimate it)
dropped = orders.dropna(subset=["total"])
print(f"Rows after dropping: {len(dropped)}")   # 4 (started with 6)
print(f"Total revenue (dropped): ${dropped['total'].sum():.2f}")   # $215.99

# 3. Option B: impute missing totals with the column median
median_total = orders["total"].median()
imputed = orders.copy()
imputed["total"] = imputed["total"].fillna(median_total)
print(imputed[["order_id", "total"]])
#    order_id  total
# 0         1  45.99
# 1         2  57.50   <- filled with median
# 2         3  89.99
# 3         4  15.00
# 4         5  57.50   <- filled with median
# 5         6  65.00
print(f"Total revenue (imputed): ${imputed['total'].sum():.2f}")   # $330.98

# Notice the two approaches give meaningfully different total revenue figures
# ($215.99 vs $330.98) — this is exactly why the choice matters and should be
# a deliberate, documented decision, not a default you don't think about

# 4. Handle the missing customer names separately — here, a categorical fill
# makes more sense than dropping, since we still want the order's total counted
orders["customer"] = orders["customer"].fillna("Unknown Customer")
print(orders["customer"].tolist())
# ['Maria Chen', 'James Lee', 'Unknown Customer', 'Priya Patel', 'Sam Osei', 'Unknown Customer']
```

## COMMON MISTAKES

**1. Dropping rows without checking how much data you're actually losing**
```python
df.dropna()   # if 40% of rows have SOME missing value, you just lost almost half your data!
```
Fix: always check `df.isna().sum()` or `len(df.dropna())` *before* committing to a drop, so you know the scale of what you're discarding.

**2. Using `.dropna()` (which defaults to "any row with ANY missing value") when you meant to target one specific column**
```python
df.dropna()                    # drops a row even if an UNRELATED column has a missing value
df.dropna(subset=["price"])      # only drops rows missing SPECIFICALLY the price column
```
Fix: use `subset=` whenever you only care about missingness in particular columns — the unscoped default is more aggressive than most people expect.

**3. Forgetting `fillna()`/`dropna()` return a new object rather than modifying in place**
```python
df.fillna(0)              # df itself is UNCHANGED — the result wasn't saved anywhere!
df = df.fillna(0)           # correct — reassign the result
```
Same pattern as Lesson 4.1's `set_index()` mistake — most pandas methods return new objects by default.

**4. Imputing with the mean on heavily skewed data without checking first**
```python
df["total"].fillna(df["total"].mean())   # a few huge outlier orders will pull the mean upward,
                                           # making every imputed value too high
```
Fix: as in Lesson 3.5, check whether mean and median diverge meaningfully before choosing — median is often the safer default for monetary or count data prone to outliers.

**5. Filling missing values with a placeholder that looks like real data**
```python
df["total"].fillna(0)   # a $0 order is very different from a MISSING total — this could
                          # silently corrupt a sum() calculation into looking like a real free order
```
Fix: think about whether `0` (or any specific value) is a *plausible real value* for that column — if so, filling with it makes missing data indistinguishable from genuine zeros. Consider a clearly-flagged placeholder (`"Unknown"` for text, or leaving numeric fields as `NaN` and using `nanmean`-style aggregations from Lesson 3.6 instead) when zero would be misleading.

**Data Engineer Tip:** In a production pipeline, missing-data decisions belong in a written **data cleaning decisions log** — a short record of what was dropped, what was imputed, with what value, and why. This isn't just good practice — it's what lets a colleague (or your future self, or a stakeholder questioning a number) understand exactly how the final numbers were derived. This habit is what separates a "quick script" from a defensible, reproducible analysis, and it's especially valuable for a portfolio project where a reviewer can see you made a deliberate, reasoned choice rather than an unexamined default.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given a DataFrame with a `price` column containing 2 `NaN` values (out of 5 rows), print the count of missing values and the percentage missing.

<details>
<summary>Hint</summary>
`.isna().sum()` and `.isna().mean() * 100`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({"price": [9.99, np.nan, 29.99, np.nan, 15.00]})
print(df["price"].isna().sum())        # 2
print(df["price"].isna().mean() * 100)   # 40.0
```
</details>

---

**Exercise 2 (Easy):** Given the same DataFrame, fill the missing prices with `0` and print the result.

<details>
<summary>Hint</summary>
`df["price"] = df["price"].fillna(0)`
</details>

<details>
<summary>Solution</summary>

```python
df["price"] = df["price"].fillna(0)
print(df)
```
</details>

---

**Exercise 3 (Medium):** Given a DataFrame with missing values in two different columns (`price` and `category`), drop only the rows where `price` specifically is missing (ignore missingness in `category`), and print how many rows remain.

<details>
<summary>Hint</summary>
`df.dropna(subset=["price"])`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({
    "price": [9.99, np.nan, 29.99, 15.00],
    "category": ["Electronics", "Home", None, "Outdoor"],
})

result = df.dropna(subset=["price"])
print(len(result))   # 3
print(result)
```
</details>

---

**Exercise 4 (Medium):** Given a `total` column with some missing values, compute both the mean and median of the non-missing values, then create two separate imputed versions of the column — one filled with the mean, one filled with the median — and print both, along with a comment on which values differ and by how much.

<details>
<summary>Hint</summary>
`.mean()` and `.median()` automatically skip NaN by default in pandas (unlike raw NumPy, no `nan`-prefix needed here) — then use two separate `.fillna()` calls.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import numpy as np

df = pd.DataFrame({"total": [45.99, np.nan, 89.99, 15.00, np.nan, 500.00]})

mean_val = df["total"].mean()
median_val = df["total"].median()

filled_mean = df["total"].fillna(mean_val)
filled_median = df["total"].fillna(median_val)

print("Filled with mean:", filled_mean.tolist())
print("Filled with median:", filled_median.tolist())
print(f"Mean: {mean_val:.2f}, Median: {median_val:.2f}")
# The $500 outlier pulls the mean noticeably higher than the median
```
</details>

---

**Exercise 5 (Hard):** Given this DataFrame:
```python
orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5, 6, 7, 8],
    "customer": ["Maria Chen", None, "James Lee", "Priya Patel", None, "Sam Osei", "Maria Chen", None],
    "total": [45.99, 120.50, np.nan, 15.00, 200.00, np.nan, 89.99, 65.00],
})
```
Write code that: (1) reports the missing count and percentage for both `customer` and `total`, (2) creates a cleaned copy where missing `total` values are filled with the median of the non-missing totals, and missing `customer` values are filled with `"Guest"`, (3) confirms the cleaned copy has zero missing values remaining anywhere, and (4) prints the total revenue before (ignoring missing, using pandas' default NaN-skipping `.sum()`) and after cleaning, noting they should be IDENTICAL for `total`, since median imputation of only the total column doesn't change what `.sum()` would already compute while skipping NaN — but explain in a comment why this wouldn't be true if you'd imputed with something other than the sum-preserving choice.

<details>
<summary>Hint</summary>
For part 4: pandas' `.sum()` already ignores NaN by default, so the sum BEFORE cleaning (ignoring the 2 missing totals) equals the sum AFTER cleaning only if you filled with a value that doesn't add anything extra — but median fill DOES add real numbers to what were previously ignored positions, so the sums will actually differ. Compute both and observe the difference is exactly `2 * median_total` (since 2 rows were filled).
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import numpy as np

orders = pd.DataFrame({
    "order_id": [1, 2, 3, 4, 5, 6, 7, 8],
    "customer": ["Maria Chen", None, "James Lee", "Priya Patel", None, "Sam Osei", "Maria Chen", None],
    "total": [45.99, 120.50, np.nan, 15.00, 200.00, np.nan, 89.99, 65.00],
})

# 1. Missing report
for col in ["customer", "total"]:
    count = orders[col].isna().sum()
    pct = orders[col].isna().mean() * 100
    print(f"{col}: {count} missing ({pct:.1f}%)")

# 2. Clean a copy
cleaned = orders.copy()
median_total = cleaned["total"].median()
cleaned["total"] = cleaned["total"].fillna(median_total)
cleaned["customer"] = cleaned["customer"].fillna("Guest")

# 3. Confirm no missing values remain
print("Missing after cleaning:", cleaned.isna().sum().sum())   # 0

# 4. Compare totals before and after
sum_before = orders["total"].sum()      # pandas .sum() already skips NaN by default
sum_after = cleaned["total"].sum()
print(f"Sum before (NaN skipped): {sum_before:.2f}")
print(f"Sum after (imputed):      {sum_after:.2f}")
print(f"Difference: {sum_after - sum_before:.2f} (= 2 * median of {median_total:.2f})")

# The sums DIFFER because median imputation adds real, non-zero values into
# the 2 previously-missing positions — .sum() before cleaning silently
# ignored them entirely, while after cleaning they now contribute their
# imputed value to the total. Only filling with a value of exactly 0 would
# have left the sum unchanged from the "ignore missing" baseline.
```
</details>

---

**Previous lesson:** [4.4 — Selecting & Filtering](lesson-4.4-selecting-and-filtering.md)
**Next lesson:** [4.6 — Data Type Conversion & Cleaning](lesson-4.6-type-conversion-and-cleaning.md)
