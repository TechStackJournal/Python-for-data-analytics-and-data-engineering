# Lesson 5.6 — Time Series Basics

## CONCEPT

pandas has first-class support for time-indexed data — once a column of dates is properly converted to pandas' `datetime64` type (building directly on Module 2's `datetime` lesson), you unlock resampling (grouping by time periods like week/month), date-based filtering, and a rich `.dt` accessor for extracting date components from every row at once.

```python
import pandas as pd

df["order_date"] = pd.to_datetime(df["order_date"])
df.set_index("order_date").resample("M")["total"].sum()   # monthly totals
```

## WHY IT MATTERS

Nearly every business question has a time dimension: "revenue by month," "orders in the last 30 days," "which day of the week has the most sales." Module 2's `datetime` lesson covered doing this one value at a time — pandas' time series tools do it for an entire column at once, and add powerful grouping-by-time-period operations (`resample()`) that have no simple manual equivalent.

## SYNTAX REFERENCE

```python
import pandas as pd

# Converting a column to real datetime (essential first step — see Lesson 4.6 too)
df["order_date"] = pd.to_datetime(df["order_date"])

# The .dt accessor — extract date components from an entire column at once
df["order_date"].dt.year
df["order_date"].dt.month
df["order_date"].dt.day
df["order_date"].dt.day_name()        # 'Monday', 'Tuesday', ...
df["order_date"].dt.dayofweek           # 0=Monday ... 6=Sunday
df["order_date"].dt.quarter               # 1, 2, 3, or 4
df["order_date"].dt.is_month_end            # True/False

# Filtering by date range
df[df["order_date"] >= "2024-11-01"]
df[(df["order_date"] >= "2024-11-01") & (df["order_date"] <= "2024-11-30")]
df[df["order_date"].dt.year == 2024]

# Date arithmetic on a column
df["days_since_order"] = (pd.Timestamp.now() - df["order_date"]).dt.days

# Setting a datetime column as the index — required for .resample()
df = df.set_index("order_date")

# Resampling: group by a time period and aggregate — like groupby(), but time-aware
df.resample("D")["total"].sum()        # daily totals
df.resample("W")["total"].sum()         # weekly totals
df.resample("M")["total"].sum()          # monthly totals
df.resample("Q")["total"].sum()           # quarterly totals
df.resample("M").agg({"total": "sum", "order_id": "count"})   # multiple aggregations

# Common resample frequency codes: 'D' day, 'W' week, 'M' month, 'Q' quarter, 'Y' year
```

## WORKED EXAMPLE

Converting raw order timestamps and building a monthly revenue report, plus a day-of-week analysis:

```python
import pandas as pd

orders = pd.DataFrame({
    "order_id": range(1, 9),
    "order_date": ["2024-10-05", "2024-10-18", "2024-11-02", "2024-11-15",
                   "2024-11-28", "2024-12-01", "2024-12-10", "2024-12-22"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00, 199.99, 22.50, 15.00],
})

# Step 1: convert to real datetime
orders["order_date"] = pd.to_datetime(orders["order_date"])
print(orders.dtypes)
# order_id       int64
# order_date    datetime64[ns]
# total        float64

# Extracting components with .dt — no loop needed
orders["month"] = orders["order_date"].dt.month
orders["day_of_week"] = orders["order_date"].dt.day_name()
print(orders[["order_date", "month", "day_of_week"]])
#   order_date  month day_of_week
# 0 2024-10-05     10    Saturday
# 1 2024-10-18     10      Friday
# ...

# Filtering: orders from November onward
recent = orders[orders["order_date"] >= "2024-11-01"]
print(f"Orders since November: {len(recent)}")   # 6

# Resampling: monthly revenue — requires a DatetimeIndex first
orders_indexed = orders.set_index("order_date")
monthly_revenue = orders_indexed.resample("M")["total"].sum()
print(monthly_revenue)
# order_date
# 2024-10-31    166.49
# 2024-11-30    188.99
# 2024-12-31    237.49
# Freq: M, Name: total, dtype: float64

# Resample with multiple aggregations at once
monthly_summary = orders_indexed.resample("M").agg(
    total_revenue=("total", "sum"),
    order_count=("order_id", "count"),
)
print(monthly_summary)
#             total_revenue  order_count
# order_date
# 2024-10-31         166.49            2
# 2024-11-30         188.99            3
# 2024-12-31         237.49            3

# Day-of-week analysis (using the ORIGINAL, non-indexed DataFrame, since groupby
# doesn't require a DatetimeIndex the way resample() does)
by_day = orders.groupby("day_of_week")["total"].sum().sort_values(ascending=False)
print(by_day.head(3))
```

## COMMON MISTAKES

**1. Forgetting to convert to datetime before using `.dt` or `.resample()`**
```python
df["order_date"].dt.month   # AttributeError if order_date is still a plain string column!
```
Fix: `pd.to_datetime()` is a required first step, exactly as in Lesson 4.6 — the `.dt` accessor and `.resample()` only exist on genuine datetime-typed columns.

**2. Trying to `.resample()` without a DatetimeIndex**
```python
df.resample("M")["total"].sum()   # TypeError — resample needs the DATETIME COLUMN as the INDEX
df.set_index("order_date").resample("M")["total"].sum()   # correct
```
Fix: `.resample()` specifically requires a `DatetimeIndex` — `.groupby()` with a `.dt` accessor is the alternative when you'd rather not restructure the index (see the day-of-week example above, which uses `groupby` instead).

**3. Comparing a datetime column to a string without pandas' implicit conversion working as expected**
```python
df[df["order_date"] >= "2024-11-01"]   # this actually WORKS — pandas auto-converts the string
```
This one is technically *not* a mistake — pandas is smart about comparing a real `datetime64` column to a well-formatted date string. It's included here as a reassurance: you don't need to manually convert the comparison value in most cases, though being explicit (`pd.Timestamp("2024-11-01")`) is safer for unusual date formats.

**4. Choosing the wrong resample frequency and getting an unexpectedly coarse or fine result**
```python
df.resample("Y")["total"].sum()   # yearly — probably too coarse if you only have 3 months of data
```
Fix: match the resample frequency to the actual time span and granularity of your question — 'D' for daily operational detail, 'M' for a monthly business report, 'Q'/'Y' for longer-term trend views.

**5. Not sorting by date before doing anything date-order-dependent**
```python
df.resample("M")["total"].sum()   # resample handles unsorted data fine internally, but...
df["running_total"] = df["total"].cumsum()   # ...a cumulative calculation on UNSORTED dates is meaningless
```
Fix: `.sort_values("order_date")` before any calculation that depends on chronological order (a preview of Lesson 5.7's rolling/cumulative operations).

**Data Engineer Tip:** `.resample("M")["total"].sum()` is the pandas equivalent of SQL's `DATE_TRUNC('month', order_date)` combined with `GROUP BY` — both collapse a date column down to a coarser period before aggregating. As with earlier lessons, this is a genuine candidate for pushing upstream into SQL when the source data lives in a database and the volume is large — `DATE_TRUNC` + `GROUP BY` in the database is typically far more efficient than pulling every raw row into pandas first and resampling client-side.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `df = pd.DataFrame({"date": ["2024-01-15", "2024-02-20"], "value": [100, 200]})`, convert the `date` column to datetime and print its dtype to confirm.

<details>
<summary>Hint</summary>
`pd.to_datetime(df["date"])`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"date": ["2024-01-15", "2024-02-20"], "value": [100, 200]})
df["date"] = pd.to_datetime(df["date"])
print(df.dtypes)
```
</details>

---

**Exercise 2 (Easy):** Using the same DataFrame (with `date` already converted), extract and print the month number and day name for each row using the `.dt` accessor.

<details>
<summary>Hint</summary>
`.dt.month` and `.dt.day_name()`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

df = pd.DataFrame({"date": ["2024-01-15", "2024-02-20"], "value": [100, 200]})
df["date"] = pd.to_datetime(df["date"])

df["month"] = df["date"].dt.month
df["day_name"] = df["date"].dt.day_name()
print(df)
```
</details>

---

**Exercise 3 (Medium):** Given
```python
orders = pd.DataFrame({
    "order_date": ["2024-06-01", "2024-06-15", "2024-07-03", "2024-07-20"],
    "total": [45.99, 89.99, 120.50, 34.00],
})
```
convert `order_date` to datetime, filter to keep only orders from July 2024 onward, and print the filtered result.

<details>
<summary>Hint</summary>
`df[df["order_date"] >= "2024-07-01"]` after conversion.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "order_date": ["2024-06-01", "2024-06-15", "2024-07-03", "2024-07-20"],
    "total": [45.99, 89.99, 120.50, 34.00],
})
orders["order_date"] = pd.to_datetime(orders["order_date"])

july_onward = orders[orders["order_date"] >= "2024-07-01"]
print(july_onward)
```
</details>

---

**Exercise 4 (Medium):** Using the same `orders` DataFrame, set `order_date` as the index and compute weekly total revenue using `.resample("W")`. Print the result.

<details>
<summary>Hint</summary>
`orders.set_index("order_date").resample("W")["total"].sum()`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "order_date": ["2024-06-01", "2024-06-15", "2024-07-03", "2024-07-20"],
    "total": [45.99, 89.99, 120.50, 34.00],
})
orders["order_date"] = pd.to_datetime(orders["order_date"])

weekly = orders.set_index("order_date").resample("W")["total"].sum()
print(weekly)
```
</details>

---

**Exercise 5 (Hard):** Given a larger set of orders:
```python
orders = pd.DataFrame({
    "order_date": ["2024-01-05", "2024-01-20", "2024-02-10", "2024-02-14",
                   "2024-02-28", "2024-03-01", "2024-03-15", "2024-03-30"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00, 199.99, 22.50, 300.00],
})
```
Convert `order_date` to datetime. Then: (1) build a monthly summary with `total_revenue` and `order_count` using named aggregation with `.resample("M")`, (2) find which single month had the highest revenue, and (3) separately (without resampling), use `.dt.day_name()` and a `groupby()` to find which day of the week has the highest total revenue across the whole dataset. Print all results.

<details>
<summary>Hint</summary>
For step 2, use `.idxmax()` on the monthly revenue Series to get the month with the highest value. For step 3, add a `day_of_week` column with `.dt.day_name()`, then `groupby("day_of_week")["total"].sum().idxmax()`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

orders = pd.DataFrame({
    "order_date": ["2024-01-05", "2024-01-20", "2024-02-10", "2024-02-14",
                   "2024-02-28", "2024-03-01", "2024-03-15", "2024-03-30"],
    "total": [45.99, 120.50, 89.99, 34.00, 65.00, 199.99, 22.50, 300.00],
})
orders["order_date"] = pd.to_datetime(orders["order_date"])

monthly = orders.set_index("order_date").resample("M").agg(
    total_revenue=("total", "sum"),
    order_count=("total", "count"),
)
print(monthly)

best_month = monthly["total_revenue"].idxmax()
print(f"Best month: {best_month.strftime('%B %Y')}")

orders["day_of_week"] = orders["order_date"].dt.day_name()
by_day = orders.groupby("day_of_week")["total"].sum()
best_day = by_day.idxmax()
print(f"Best day of week: {best_day} (${by_day[best_day]:.2f})")
```
</details>

---

**Previous lesson:** [5.5 — Concatenating & Appending](lesson-5.5-concatenating.md)
**Next lesson:** [5.7 — Window Functions](lesson-5.7-window-functions.md)
