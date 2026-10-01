# Lesson 5.7 — Window Functions (`.rolling()`, `.expanding()`)

## CONCEPT

A **window function** computes a value using a "window" of nearby rows, rather than the whole dataset (a full aggregation) or a single row alone. pandas provides two main tools: **`.rolling()`** (a fixed-size sliding window — e.g., "the last 7 days") and **`.expanding()`** (a growing window that always starts from the beginning — e.g., "running total to date").

```python
import pandas as pd

df["total"].rolling(window=7).mean()      # 7-period moving average
df["total"].expanding().sum()               # running (cumulative) total
```

If you covered window functions in the SQL course (`OVER (ORDER BY ... ROWS BETWEEN ...)`), this is the exact same concept — pandas syntax, same underlying idea.

## WHY IT MATTERS

"7-day moving average of daily revenue" (to smooth out day-to-day noise), "running total of orders this month," "each customer's cumulative spend over time" — these are extremely common in real reporting, and they can't be computed with a simple `groupby()` (which collapses rows) or a plain aggregation (which produces one number). Rolling and expanding windows are the tool specifically built for "a value that depends on this row plus some nearby/prior rows."

## SYNTAX REFERENCE

```python
import pandas as pd

# Rolling (fixed-size, sliding window) — requires data already sorted by the window order
df["total"].rolling(window=7).mean()       # 7-row moving average
df["total"].rolling(window=7).sum()          # 7-row moving sum
df["total"].rolling(window=3).max()            # 3-row moving max

# min_periods — allow a result even before the full window is available (default requires a full window)
df["total"].rolling(window=7, min_periods=1).mean()   # partial windows at the start are OK

# Expanding (growing window, from the start of the data to the current row)
df["total"].expanding().sum()          # running total
df["total"].expanding().mean()          # running average
df["total"].expanding().max()            # running maximum-so-far

# Simple cumulative functions (a shortcut for common expanding operations)
df["total"].cumsum()          # equivalent to expanding().sum()
df["total"].cummax()           # equivalent to expanding().max()
df["total"].cummin()

# Rolling/expanding on TIME-based windows (requires a DatetimeIndex, from Lesson 5.6)
df.set_index("order_date")["total"].rolling("7D").sum()    # a true 7-CALENDAR-DAY window,
                                                              # not just "the last 7 rows"

# Rolling/expanding WITHIN groups — combine with groupby for per-customer running totals
df.groupby("customer_id")["total"].cumsum()
df.groupby("customer_id")["total"].transform(lambda s: s.rolling(3, min_periods=1).mean())
```

## WORKED EXAMPLE

Building a 7-day moving average of daily revenue, a running total, and a per-customer cumulative spend — three genuinely different window-function use cases:

```python
import pandas as pd

daily_revenue = pd.DataFrame({
    "date": pd.date_range("2024-11-01", periods=10, freq="D"),
    "revenue": [1200, 950, 1400, 1100, 1600, 900, 1750, 1300, 1050, 1900],
})

# Rolling 3-day moving average — smooths out day-to-day noise
daily_revenue["moving_avg_3d"] = daily_revenue["revenue"].rolling(window=3).mean()
print(daily_revenue[["date", "revenue", "moving_avg_3d"]])
#         date  revenue  moving_avg_3d
# 0 2024-11-01     1200            NaN
# 1 2024-11-02      950            NaN
# 2 2024-11-03     1400   1183.333333    <- average of rows 0,1,2
# 3 2024-11-04     1100   1150.000000    <- average of rows 1,2,3
# ...
# Notice the first 2 rows are NaN — there aren't yet 3 rows to average.
# This is expected default behavior, not a bug.

# Allow partial windows at the start with min_periods
daily_revenue["moving_avg_3d_partial"] = daily_revenue["revenue"].rolling(window=3, min_periods=1).mean()
print(daily_revenue[["revenue", "moving_avg_3d_partial"]].head(3))
#    revenue  moving_avg_3d_partial
# 0     1200             1200.000000
# 1      950             1075.000000
# 2     1400             1183.333333

# Expanding — running total of revenue, day over day
daily_revenue["running_total"] = daily_revenue["revenue"].expanding().sum()
print(daily_revenue[["date", "revenue", "running_total"]])
#         date  revenue  running_total
# 0 2024-11-01     1200         1200.0
# 1 2024-11-02      950         2150.0
# 2 2024-11-03     1400         3550.0
# ...

# cumsum() is a simpler shortcut for the same running-total calculation
daily_revenue["running_total_v2"] = daily_revenue["revenue"].cumsum()
print((daily_revenue["running_total"] == daily_revenue["running_total_v2"]).all())   # True

# Per-customer running totals — combining groupby with a window operation
orders = pd.DataFrame({
    "customer_id": [1, 1, 2, 1, 2, 2],
    "order_date": pd.to_datetime(
        ["2024-11-01", "2024-11-05", "2024-11-02", "2024-11-10", "2024-11-06", "2024-11-15"]
    ),
    "total": [45.99, 30.00, 89.99, 15.00, 60.00, 120.00],
})
orders = orders.sort_values(["customer_id", "order_date"])
orders["customer_running_total"] = orders.groupby("customer_id")["total"].cumsum()
print(orders[["customer_id", "order_date", "total", "customer_running_total"]])
#    customer_id order_date   total  customer_running_total
# 0            1 2024-11-01   45.99                    45.99
# 1            1 2024-11-05   30.00                    75.99
# 3            1 2024-11-10   15.00                    90.99
# 2            2 2024-11-02   89.99                    89.99
# 4            2 2024-11-06   60.00                   149.99
# 5            2 2024-11-15  120.00                   269.99
# Each customer's running total resets and accumulates independently — this is
# the pandas equivalent of SQL's PARTITION BY customer_id ORDER BY order_date
```

## COMMON MISTAKES

**1. Forgetting data must be sorted correctly before applying a rolling/expanding calculation**
```python
df["total"].rolling(window=3).mean()   # meaningless if rows aren't in chronological order!
```
Fix: always `.sort_values()` by the relevant date/order column first — a rolling average computed on unsorted data silently produces a number, just not a meaningful one.

**2. Being surprised by leading `NaN` values from `.rolling()`**
```python
df["total"].rolling(window=7).mean()   # first 6 rows are NaN — not enough data yet for a full window
```
This is standard, correct behavior — fix by using `min_periods=1` (or another appropriate minimum) if partial-window results are acceptable for your use case, or simply expect and handle the leading `NaN`s otherwise.

**3. Using `.rolling(window=7)` on ROW COUNT when you actually meant 7 CALENDAR DAYS**
```python
df["total"].rolling(window=7).mean()   # "last 7 ROWS" — if there are gaps in dates (e.g. no
                                          # orders on some days), this ISN'T the same as "last 7 days"!
```
Fix: if your data has gaps and you specifically need a true calendar-time window, use a time-based rolling window instead — `df.set_index("date")["total"].rolling("7D").mean()` — which correctly accounts for actual elapsed calendar time, not just row count.

**4. Applying `.cumsum()` directly on an ungrouped column when you meant per-group running totals**
```python
orders["total"].cumsum()                              # one single running total across ALL customers
orders.groupby("customer_id")["total"].cumsum()          # correct — resets per customer
```
Fix: always ask whether the running calculation should be global or per-group — this is a very easy detail to overlook, and the two produce very different (and differently wrong, if you meant the other one) results.

**5. Confusing `.rolling()` (fixed window size) with `.expanding()` (always grows from the start)**
A rolling window of size 3 always uses exactly the last 3 rows (once enough data exists); an expanding window always includes *every* row from the beginning up to the current one. Fix: pick based on the actual question — "recent trend, smoothing out noise" wants rolling; "running total to date" wants expanding (or `cumsum()`/`cummax()` for the common cases).

**Data Engineer Tip:** `.groupby("customer_id")["total"].cumsum()` in the worked example is the direct pandas equivalent of `SUM(total) OVER (PARTITION BY customer_id ORDER BY order_date)` in SQL — both compute a running total that resets per customer. If your SQL course's window functions module covered `ROWS BETWEEN` or a moving average, `.rolling()` is the same concept again. As with the module-level tip: for large datasets already living in a database, computing this kind of running/moving metric in SQL (closer to the data) is often more efficient than pulling everything into pandas first — but pandas' version is frequently faster to prototype and iterate on during exploratory analysis.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `values = pd.Series([10, 20, 30, 40, 50])`, compute a rolling 2-period sum and print it, noting the first value is `NaN`.

<details>
<summary>Hint</summary>
`values.rolling(window=2).sum()`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

values = pd.Series([10, 20, 30, 40, 50])
rolling_sum = values.rolling(window=2).sum()
print(rolling_sum)
# 0     NaN
# 1    30.0
# 2    50.0
# 3    70.0
# 4    90.0
```
</details>

---

**Exercise 2 (Easy):** Using the same Series, compute the running (cumulative) total using `.cumsum()`, and print it.

<details>
<summary>Hint</summary>
`values.cumsum()`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

values = pd.Series([10, 20, 30, 40, 50])
running_total = values.cumsum()
print(running_total)
# 0     10
# 1     30
# 2     60
# 3    100
# 4    150
```
</details>

---

**Exercise 3 (Medium):** Given
```python
sales = pd.DataFrame({
    "day": range(1, 8),
    "revenue": [100, 150, 120, 200, 180, 90, 220],
})
```
compute a 3-day rolling average of `revenue`, using `min_periods=1` so the first two days still show a (partial-window) value instead of `NaN`. Print the result.

<details>
<summary>Hint</summary>
`sales["revenue"].rolling(window=3, min_periods=1).mean()`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

sales = pd.DataFrame({
    "day": range(1, 8),
    "revenue": [100, 150, 120, 200, 180, 90, 220],
})

sales["rolling_avg"] = sales["revenue"].rolling(window=3, min_periods=1).mean()
print(sales)
```
</details>

---

**Exercise 4 (Medium):** Given
```python
orders = pd.DataFrame({
    "customer_id": [1, 2, 1, 2, 1],
    "order_date": pd.to_datetime(["2024-01-01", "2024-01-02", "2024-01-10", "2024-01-15", "2024-01-20"]),
    "total": [20.00, 30.00, 15.00, 25.00, 40.00],
})
```
sort by `customer_id` and `order_date`, then compute each customer's running total spend using `.groupby().cumsum()`. Print the result.

<details>
<summary>Hint</summary>
Sort first with `.sort_values(["customer_id", "order_date"])`, then `.groupby("customer_id")["total"].cumsum()`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "customer_id": [1, 2, 1, 2, 1],
    "order_date": pd.to_datetime(["2024-01-01", "2024-01-02", "2024-01-10", "2024-01-15", "2024-01-20"]),
    "total": [20.00, 30.00, 15.00, 25.00, 40.00],
})

orders = orders.sort_values(["customer_id", "order_date"])
orders["running_total"] = orders.groupby("customer_id")["total"].cumsum()
print(orders)
```
</details>

---

**Exercise 5 (Hard):** Given 14 days of daily revenue:
```python
daily = pd.DataFrame({
    "date": pd.date_range("2024-11-01", periods=14, freq="D"),
    "revenue": [1200, 950, 1400, 1100, 1600, 900, 1750,
                1300, 1050, 1900, 1150, 1400, 1600, 2000],
})
```
Compute: (1) a 7-day rolling average (with `min_periods=1`), (2) a 7-day rolling standard deviation (same `min_periods`), and (3) flag any day where the actual `revenue` is more than 1.5 standard deviations above that day's 7-day rolling average (a simple anomaly-detection pattern) — call this column `is_spike`. Print the full DataFrame with all new columns, and print just the rows flagged as spikes.

<details>
<summary>Hint</summary>
Compute the rolling mean and rolling std as separate columns first. Then build `is_spike` as a boolean: `daily["revenue"] > daily["rolling_avg"] + 1.5 * daily["rolling_std"]`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

daily = pd.DataFrame({
    "date": pd.date_range("2024-11-01", periods=14, freq="D"),
    "revenue": [1200, 950, 1400, 1100, 1600, 900, 1750,
                1300, 1050, 1900, 1150, 1400, 1600, 2000],
})

daily["rolling_avg"] = daily["revenue"].rolling(window=7, min_periods=1).mean()
daily["rolling_std"] = daily["revenue"].rolling(window=7, min_periods=1).std()

daily["is_spike"] = daily["revenue"] > (daily["rolling_avg"] + 1.5 * daily["rolling_std"])

print(daily)
print("\nSpike days:")
print(daily[daily["is_spike"]][["date", "revenue", "rolling_avg", "rolling_std"]])
```
Note: the very first row will always have `rolling_std` as `NaN` (a single value has no standard deviation), so `is_spike` for that row evaluates to `False` — worth being aware of when interpreting results near the start of any rolling-window calculation.
</details>

---

**Previous lesson:** [5.6 — Time Series Basics](lesson-5.6-time-series-basics.md)
**Next lesson:** [5.8 — Cohort Analysis Patterns](lesson-5.8-cohort-analysis.md)
