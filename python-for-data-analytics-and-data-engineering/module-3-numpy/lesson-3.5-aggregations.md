# Lesson 3.5 — Aggregations

## CONCEPT

An **aggregation** collapses an array of many values into one summary value (or a smaller set of values) — sum, mean, minimum, maximum, standard deviation, percentiles. NumPy provides fast, vectorized aggregation functions instead of manually accumulating totals in a loop (the pattern from Module 1).

```python
import numpy as np

prices = np.array([24.99, 12.50, 89.00, 34.00])
prices.sum()      # 160.49
prices.mean()      # 40.1225
prices.max()        # 89.0
prices.min()         # 12.5
```

For 2D arrays, aggregations can run across the **whole array**, or along a specific **axis** (rows or columns independently) — a distinction that matters constantly once you're working with tabular data.

## WHY IT MATTERS

"What's the average order value?" "What's the total revenue?" "What's the highest and lowest price in this category?" — these are the exact questions data analytics exists to answer, and NumPy aggregations are the computational foundation for all of them. The `axis` parameter specifically previews one of the most important — and most confusing for beginners — concepts in pandas `groupby()` operations (Module 5): summarizing *within* groups (rows) versus *across* groups (columns).

## SYNTAX REFERENCE

```python
import numpy as np

arr = np.array([24.99, 12.50, 89.00, 34.00, 199.99])

# Basic aggregations
arr.sum()          # total
arr.mean()          # average
arr.min()            # minimum
arr.max()             # maximum
arr.std()              # standard deviation
arr.var()               # variance

# Equivalent function form (same results, different calling style)
np.sum(arr)
np.mean(arr)

# Percentiles and median
np.median(arr)
np.percentile(arr, 50)    # same as median
np.percentile(arr, 90)     # the 90th percentile value

# Position of the min/max (not the value itself — the INDEX)
arr.argmin()        # index of the smallest value
arr.argmax()          # index of the largest value

# 2D aggregations with axis
grid = np.array([[1, 2, 3], [4, 5, 6]])
grid.sum()             # 21 — sums EVERY element, whole array
grid.sum(axis=0)        # [5 7 9] — sums DOWN each column (collapses rows)
grid.sum(axis=1)         # [6 15] — sums ACROSS each row (collapses columns)

# axis=0 means "collapse rows, keep columns" (one result per column)
# axis=1 means "collapse columns, keep rows" (one result per row)

# Cumulative aggregations (running totals)
arr.cumsum()          # running total at each position
arr.cummax()           # running maximum at each position
```

## WORKED EXAMPLE

Summarizing a batch of order data using various aggregations, including per-region breakdowns:

```python
import numpy as np

order_totals = np.array([45.99, 120.50, 89.99, 15.00, 200.00, 65.50, 8.25, 310.00])

# Basic summary statistics
print(f"Total revenue: ${order_totals.sum():.2f}")     # $855.23
print(f"Average order: ${order_totals.mean():.2f}")     # $106.90
print(f"Smallest order: ${order_totals.min():.2f}")      # $8.25
print(f"Largest order: ${order_totals.max():.2f}")        # $310.00
print(f"Std deviation: ${order_totals.std():.2f}")         # $99.63

# Median is often more useful than mean for skewed data (e.g. a few huge orders)
print(f"Median order: ${np.median(order_totals):.2f}")   # $77.75
# Notice median ($77.75) is well below the mean ($106.90) here — the large
# $310 order is pulling the average upward, a classic case for preferring
# the median with skewed real-world order data

# 90th percentile — useful for identifying "typical high spenders"
print(f"90th percentile: ${np.percentile(order_totals, 90):.2f}")   # $244.00

# Find WHICH order was the largest
largest_index = order_totals.argmax()
print(f"Largest order was order #{largest_index}: ${order_totals[largest_index]:.2f}")
# Largest order was order #7: $310.00

# A 2D example: monthly revenue by region (rows = months, columns = regions)
monthly_by_region = np.array([
    [1200, 800, 1500],   # January: [East, West, Central]
    [1350, 900, 1400],   # February
    [1100, 950, 1600],   # March
])

# Total per region across all months — collapse ROWS, keep columns (axis=0)
region_totals = monthly_by_region.sum(axis=0)
print(f"Region totals (East, West, Central): {region_totals}")
# [3650 2650 4500]

# Total per month across all regions — collapse COLUMNS, keep rows (axis=1)
monthly_totals = monthly_by_region.sum(axis=1)
print(f"Monthly totals (Jan, Feb, Mar): {monthly_totals}")
# [3500 3650 3650]

# Which region had the highest average monthly revenue?
region_averages = monthly_by_region.mean(axis=0)
best_region_index = region_averages.argmax()
region_names = ["East", "West", "Central"]
print(f"Top region: {region_names[best_region_index]} (avg ${region_averages[best_region_index]:.2f})")
# Top region: Central (avg $1500.00)
```

## COMMON MISTAKES

**1. Confusing `axis=0` and `axis=1`**
This trips up nearly everyone at first. A reliable mental shortcut: **the axis you specify is the one that "disappears" (gets collapsed)** — `axis=0` collapses rows (you get one result per column), `axis=1` collapses columns (you get one result per row). Fix: when unsure, check the *shape* of the result — for a `(3, 4)` array, `axis=0` produces shape `(4,)` and `axis=1` produces shape `(3,)`.

**2. Using mean when the data is heavily skewed by outliers**
```python
np.array([50, 55, 60, 52, 5000]).mean()   # 1043.4 — badly distorted by one outlier
np.array([50, 55, 60, 52, 5000]).median()  # skips straight to the 55 midpoint value, unaffected
```
Fix: for real-world monetary or count data with occasional extreme values, check both mean and median — a large gap between them is itself a useful signal about the shape of your data (as seen in the worked example).

**3. Forgetting `argmin()`/`argmax()` return the INDEX, not the value**
```python
order_totals.argmax()   # 7 — this is a POSITION, not a dollar amount
order_totals.max()       # 310.0 — this is the actual value
```
Fix: use `.max()`/`.min()` for the value itself, `.argmax()`/`.argmin()` specifically when you need to know *where* that value is located (e.g., to look up a corresponding customer name at that same index in a different array).

**4. Calling an aggregation on an empty array**
```python
np.array([]).mean()   # RuntimeWarning: Mean of empty slice — returns nan, not an error
```
Fix: check `arr.size > 0` before aggregating if there's a real chance the array could be empty (e.g., after a boolean mask filtered out everything).

**5. Not specifying `axis` and being surprised the result is a single number instead of per-row/column**
```python
grid.sum()   # one single number — sums EVERYTHING, ignoring row/column structure entirely
```
Fix: if you wanted per-row or per-column totals and got one flat number back, you forgot the `axis` parameter — this is one of the most common real debugging moments when starting out with 2D aggregations.

**Data Engineer Tip:** The `axis=0` / `axis=1` distinction here is the direct ancestor of pandas' `df.sum(axis=0)` (per-column totals — the default) versus `df.sum(axis=1)` (per-row totals), and it's exactly the same mental model behind `groupby()` aggregations in Module 5: "collapse along one dimension, keep the other." If axis confuses you now, it's worth sitting with the worked example's region/month breakdown until the "the axis you name disappears" rule feels automatic — it pays off across the rest of the course.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `scores = np.array([85, 92, 78, 95, 88])`, print the sum, mean, min, and max.

<details>
<summary>Hint</summary>
`.sum()`, `.mean()`, `.min()`, `.max()`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

scores = np.array([85, 92, 78, 95, 88])
print(scores.sum(), scores.mean(), scores.min(), scores.max())
# 438 87.6 78 95
```
</details>

---

**Exercise 2 (Easy):** Given `order_totals = np.array([45.99, 120.50, 89.99, 15.00, 200.00])`, find the index of the largest order using `.argmax()`, then use that index to print the actual largest value.

<details>
<summary>Hint</summary>
`idx = order_totals.argmax()`, then `order_totals[idx]`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

order_totals = np.array([45.99, 120.50, 89.99, 15.00, 200.00])
idx = order_totals.argmax()
print(idx, order_totals[idx])   # 4 200.0
```
</details>

---

**Exercise 3 (Medium):** Given `salaries = np.array([45000, 48000, 52000, 51000, 250000])`, compute both the mean and median, and print a sentence explaining which better represents a "typical" salary in this data and why.

<details>
<summary>Hint</summary>
Compare `.mean()` and `np.median()` — the large outlier (250000) will distort the mean much more than the median.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

salaries = np.array([45000, 48000, 52000, 51000, 250000])
mean_salary = salaries.mean()
median_salary = np.median(salaries)

print(f"Mean: ${mean_salary:.2f}, Median: ${median_salary:.2f}")
print("The median better represents a typical salary here, since the mean "
      "is pulled far upward by the single $250,000 outlier.")
```
</details>

---

**Exercise 4 (Medium):** Given a 2D array `attendance = np.array([[30, 28, 32], [25, 27, 29], [35, 33, 31]])` (rows = 3 weeks, columns = 3 classes), compute the total attendance per class (across all weeks) and the total attendance per week (across all classes). Print both.

<details>
<summary>Hint</summary>
Per-class (collapse rows): `axis=0`. Per-week (collapse columns): `axis=1`.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

attendance = np.array([[30, 28, 32], [25, 27, 29], [35, 33, 31]])

per_class = attendance.sum(axis=0)
per_week = attendance.sum(axis=1)

print("Per class:", per_class)   # [90 88 92]
print("Per week:", per_week)      # [90 81 99]
```
</details>

---

**Exercise 5 (Hard):** Given a 2D array of daily revenue for 3 stores over 5 days: `revenue = np.array([[1200, 1350, 1100, 1400, 1600], [900, 950, 1000, 890, 920], [2000, 1800, 2100, 1950, 2200]])`:
1. Compute each store's average daily revenue.
2. Identify which store (by index) had the highest average, and print its name assuming `store_names = ["Downtown", "Suburb", "Flagship"]`.
3. Compute the total revenue across ALL stores for each individual day.
4. Compute the single busiest day (highest total across all stores) and print which day number (1-indexed) it was.

<details>
<summary>Hint</summary>
Store averages: `axis=1` (collapse days, keep stores). Store name lookup: use `.argmax()` on the per-store averages as an index into `store_names`. Daily totals: `axis=0` (collapse stores, keep days). Busiest day: `.argmax()` on the daily totals, remembering to add 1 for 1-indexed day numbers.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

revenue = np.array([
    [1200, 1350, 1100, 1400, 1600],
    [900, 950, 1000, 890, 920],
    [2000, 1800, 2100, 1950, 2200],
])
store_names = ["Downtown", "Suburb", "Flagship"]

store_averages = revenue.mean(axis=1)
print("Store averages:", store_averages)   # [1330. 932. 2010.]

top_store_index = store_averages.argmax()
print(f"Top store: {store_names[top_store_index]}")   # Flagship

daily_totals = revenue.sum(axis=0)
print("Daily totals:", daily_totals)   # [4100 4100 4200 4240 4720]

busiest_day_index = daily_totals.argmax()
print(f"Busiest day: Day {busiest_day_index + 1}")   # Day 5
```
</details>

---

**Previous lesson:** [3.4 — Vectorized Operations & Broadcasting](lesson-3.4-vectorization-and-broadcasting.md)
**Next lesson:** [3.6 — Handling Missing Data (NaN)](lesson-3.6-nan-handling.md)
