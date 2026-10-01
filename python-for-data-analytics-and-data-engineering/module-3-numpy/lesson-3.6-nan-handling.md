# Lesson 3.6 — Handling Missing Data (`NaN`)

## CONCEPT

**`NaN`** ("Not a Number") is NumPy's representation of a missing or undefined numeric value — `np.nan`. It behaves unlike any other value in Python: it's not equal to anything, not even itself.

```python
import numpy as np

np.nan == np.nan     # False! NaN is never equal to anything, including itself
np.isnan(np.nan)      # True — the correct way to actually check for NaN
```

`NaN` appears constantly in real data: a missing survey response, a sensor reading that failed, a customer who hasn't made a purchase yet. NumPy provides `nan`-aware versions of most aggregation functions that skip missing values instead of letting them silently contaminate a calculation.

## WHY IT MATTERS

Real datasets are never perfectly complete. A regular `.mean()` call on an array containing even a single `NaN` returns `NaN` for the *entire* result — silently ruining a calculation unless you know to watch for it. Understanding `NaN` behavior here is essential preparation for Module 4, where missing data handling (`.isna()`, `.dropna()`, `.fillna()`) is one of the most common and important parts of any real data cleaning workflow.

## SYNTAX REFERENCE

```python
import numpy as np

# Creating an array with missing values
arr = np.array([24.99, np.nan, 89.00, np.nan, 34.00])

# Checking for NaN — NEVER use == for this
np.isnan(arr)             # array([False,  True, False,  True, False])
np.isnan(arr).sum()         # 2 — count of missing values (True counts as 1)

# Regular aggregations get "contaminated" by any NaN present
arr.sum()           # nan — the whole result becomes NaN!
arr.mean()           # nan

# NaN-aware ("nan-safe") aggregation functions skip NaN automatically
np.nansum(arr)          # 147.99 — ignores the NaN values
np.nanmean(arr)          # 49.33
np.nanmax(arr)            # 89.0
np.nanmin(arr)             # 24.99
np.nanstd(arr)              # standard deviation, ignoring NaN

# Filtering OUT the missing values entirely
valid_values = arr[~np.isnan(arr)]     # keep only where NOT NaN
print(valid_values)                      # [24.99 89.   34.  ]

# Replacing NaN with a specific value
filled = np.where(np.isnan(arr), 0, arr)      # replace NaN with 0, keep everything else
print(filled)                                    # [24.99  0.   89.    0.   34.  ]

# Replacing NaN with the mean of the valid values (a common imputation technique)
mean_value = np.nanmean(arr)
filled_with_mean = np.where(np.isnan(arr), mean_value, arr)
print(filled_with_mean)   # [24.99 49.33... 89.  49.33... 34.  ]
```

## WORKED EXAMPLE

Handling missing price data in a batch of products — a common real-world scenario when some records have incomplete information:

```python
import numpy as np

# Simulating raw product prices with some missing entries (e.g. not yet priced)
raw_prices = np.array([24.99, np.nan, 89.00, 34.00, np.nan, 199.99, 15.75])

# 1. Detect how much data is missing
missing_count = np.isnan(raw_prices).sum()
total_count = raw_prices.size
missing_pct = (missing_count / total_count) * 100
print(f"Missing: {missing_count}/{total_count} ({missing_pct:.1f}%)")
# Missing: 2/7 (28.6%)

# 2. WRONG way — a regular mean is silently ruined
naive_mean = raw_prices.mean()
print(f"Naive mean (WRONG): {naive_mean}")
# Naive mean (WRONG): nan

# 3. CORRECT way — use the nan-safe aggregation
correct_mean = np.nanmean(raw_prices)
print(f"Correct mean: ${correct_mean:.2f}")
# Correct mean: $72.75

# 4. Decide how to handle the missing values, depending on the business context

# Option A: drop rows with missing prices entirely (safest if unsure)
valid_prices = raw_prices[~np.isnan(raw_prices)]
print(f"After dropping missing: {valid_prices}")
# [ 24.99  89.    34.   199.99  15.75]

# Option B: fill missing prices with the average of the known prices (imputation)
imputed_prices = np.where(np.isnan(raw_prices), correct_mean, raw_prices)
print(f"After imputing with mean: {np.round(imputed_prices, 2)}")
# [ 24.99  72.75  89.    34.    72.75 199.99  15.75]

# 5. Compare totals under each approach — this matters for downstream decisions!
print(f"Total if dropped: ${valid_prices.sum():.2f}")       # $363.73
print(f"Total if imputed: ${imputed_prices.sum():.2f}")      # $509.23
# Notice these give MEANINGFULLY different answers — the choice of how to
# handle missing data is a real analytical decision, not just a technical step
```

## COMMON MISTAKES

**1. Checking for `NaN` with `==`**
```python
arr[arr == np.nan]   # returns an EMPTY array — always! NaN never equals anything
arr[np.isnan(arr)]   # correct — this actually finds the NaN positions
```
This is the single most common `NaN`-related bug — it looks like it should work, produces no error, and just silently returns nothing.

**2. Using regular `.sum()`/`.mean()` without realizing NaN contaminates the whole result**
```python
np.array([1, 2, np.nan, 4]).sum()   # nan — not 7!
```
Fix: default to the `nan`-prefixed versions (`np.nansum`, `np.nanmean`, etc.) whenever there's a real chance of missing values in your data — or explicitly filter them out first.

**3. Silently dropping data without understanding the impact**
As shown in the worked example, dropping vs. imputing missing values can produce meaningfully different totals and averages. Fix: always check *how much* data is missing (`np.isnan(arr).sum()`) and *why* (if knowable) before deciding how to handle it — silently choosing one approach without checking can distort your analysis in ways that are hard to catch later.

**4. Forgetting `NaN` is a float-only concept**
```python
np.array([1, 2, np.nan], dtype=int)   # ValueError — cannot convert NaN to integer
```
Fix: an array containing `NaN` must have a float dtype — this is one reason a column of "whole number" data (like counts) sometimes unexpectedly shows up as `float64` in pandas: the presence of even one missing value forces the whole column to float, since integers have no way to represent "missing."

**5. Assuming `np.isnan()` works on non-numeric arrays**
```python
np.isnan(np.array(["a", "b", None]))   # TypeError — isnan() only works on numeric dtypes
```
Fix: for string or object arrays, missing values are usually represented as `None` rather than `np.nan` — check with `value is None` or `arr == None` (with `==`, unusually, this specific check does work correctly for `None`, unlike `NaN`) instead.

**Data Engineer Tip:** Data quality checks in a real pipeline almost always start with a missing-value audit: "what percentage of this column is null, and is that expected?" A sudden spike in missing values compared to yesterday's load is one of the most common early-warning signs of an upstream data quality issue — a source system change, a broken integration, a failed extract — and it's usually the very first check worth automating in any pipeline (a preview of the data quality checks covered in Module 9).

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `arr = np.array([1.0, 2.0, np.nan, 4.0])`, use `np.isnan()` to count how many missing values it contains.

<details>
<summary>Hint</summary>
`np.isnan(arr).sum()`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

arr = np.array([1.0, 2.0, np.nan, 4.0])
missing_count = np.isnan(arr).sum()
print(missing_count)   # 1
```
</details>

---

**Exercise 2 (Easy):** Given the same array, compute its mean correctly (ignoring the `NaN`), and print the result.

<details>
<summary>Hint</summary>
`np.nanmean(arr)`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

arr = np.array([1.0, 2.0, np.nan, 4.0])
correct_mean = np.nanmean(arr)
print(correct_mean)   # 2.3333...
```
</details>

---

**Exercise 3 (Medium):** Given `ratings = np.array([4.5, np.nan, 3.0, 5.0, np.nan, 2.5])`, build a new array containing only the valid (non-missing) ratings, and print it along with its length.

<details>
<summary>Hint</summary>
`ratings[~np.isnan(ratings)]`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

ratings = np.array([4.5, np.nan, 3.0, 5.0, np.nan, 2.5])
valid_ratings = ratings[~np.isnan(ratings)]
print(valid_ratings, len(valid_ratings))
# [4.5 3.  5.  2.5] 4
```
</details>

---

**Exercise 4 (Medium):** Given `prices = np.array([19.99, np.nan, 45.00, np.nan, 12.50])`, replace every missing value with `0.0` (rather than dropping or imputing with the mean), and print the resulting array and its (now safe to compute normally) sum.

<details>
<summary>Hint</summary>
`np.where(np.isnan(prices), 0, prices)`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

prices = np.array([19.99, np.nan, 45.00, np.nan, 12.50])
filled = np.where(np.isnan(prices), 0, prices)
print(filled)          # [19.99  0.   45.    0.   12.5 ]
print(filled.sum())      # 77.49
```
</details>

---

**Exercise 5 (Hard):** Given a batch of customer satisfaction scores with missing values: `scores = np.array([8.5, np.nan, 6.0, 9.0, np.nan, 7.5, np.nan, 4.0, 8.0])`:
1. Report the percentage of missing data.
2. Compute the mean using both the "drop missing" approach (filter first, then mean) and the `np.nanmean()` approach, and confirm they produce the same result (they should — nanmean effectively does this internally).
3. Build a version of the array where missing values are imputed with the *median* of the valid scores instead of the mean (a common choice when the data might be skewed).
4. Print all results, each clearly labeled.

<details>
<summary>Hint</summary>
For step 2, compare `scores[~np.isnan(scores)].mean()` against `np.nanmean(scores)` — they should match. For step 3, use `np.nanmedian()` to get the median while ignoring NaN, then `np.where()` to fill.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

scores = np.array([8.5, np.nan, 6.0, 9.0, np.nan, 7.5, np.nan, 4.0, 8.0])

missing_pct = (np.isnan(scores).sum() / scores.size) * 100
print(f"Missing: {missing_pct:.1f}%")   # 33.3%

drop_then_mean = scores[~np.isnan(scores)].mean()
nanmean_result = np.nanmean(scores)
print(f"Drop-then-mean: {drop_then_mean:.4f}")     # 7.1667
print(f"np.nanmean:      {nanmean_result:.4f}")     # 7.1667
print("Match:", round(drop_then_mean, 4) == round(nanmean_result, 4))   # True

median_value = np.nanmedian(scores)
imputed_with_median = np.where(np.isnan(scores), median_value, scores)
print(f"Median used for imputation: {median_value}")     # 7.75
print(f"Imputed array: {imputed_with_median}")
# [8.5  7.75 6.   9.   7.75 7.5  7.75 4.   8.  ]
```
</details>

---

**Previous lesson:** [3.5 — Aggregations](lesson-3.5-aggregations.md)
**Next lesson:** [3.7 — Random Sampling](lesson-3.7-random-sampling.md)
