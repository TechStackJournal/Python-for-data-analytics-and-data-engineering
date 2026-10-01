# Lesson 2.4 — Dates & Times

## CONCEPT

Python's built-in `datetime` module represents dates and times as proper objects (not plain strings), which lets you do date arithmetic, comparisons, and formatting reliably — instead of manually parsing text.

```python
from datetime import datetime, date, timedelta

now = datetime.now()               # current date and time
today = date.today()               # current date only
order_date = datetime(2024, 11, 3) # a specific date
```

The two operations you'll use constantly: **parsing** (turning a text timestamp into a `datetime` object) and **formatting** (turning a `datetime` object back into readable text).

## WHY IT MATTERS

Every dataset with a time dimension — order dates, event timestamps, signup dates — arrives as text (`"2024-11-03"`, `"2024-11-03T14:22:00Z"`) and needs to become a real `datetime` before you can filter by date range, compute "days since signup," or group by month. Getting comfortable with parsing, formatting, and date arithmetic here is directly reused in Module 5 (pandas time series) — pandas' `Timestamp` type is built on the exact same concepts as Python's `datetime`.

## SYNTAX REFERENCE

```python
from datetime import datetime, date, timedelta

# Parsing a string into a datetime (strptime = "string parse time")
datetime.strptime("2024-11-03", "%Y-%m-%d")
datetime.strptime("11/03/2024 14:22", "%m/%d/%Y %H:%M")

# Common format codes
# %Y = 4-digit year    %m = month (01-12)   %d = day (01-31)
# %H = hour (00-23)    %M = minute          %S = second
# %B = full month name (e.g. "November")

# Formatting a datetime back into a string (strftime = "string format time")
order_date = datetime(2024, 11, 3, 14, 22)
order_date.strftime("%Y-%m-%d")            # "2024-11-03"
order_date.strftime("%B %d, %Y")           # "November 03, 2024"

# Accessing components
order_date.year        # 2024
order_date.month       # 11
order_date.day         # 3
order_date.weekday()   # 0 = Monday ... 6 = Sunday

# Date arithmetic with timedelta
one_week_later = order_date + timedelta(days=7)
one_hour_earlier = order_date - timedelta(hours=1)

# Difference between two dates
delta = datetime(2024, 12, 1) - datetime(2024, 11, 3)
delta.days              # 28

# Comparisons work naturally
datetime(2024, 11, 3) < datetime(2024, 12, 1)   # True

# Today's date / current timestamp
date.today()
datetime.now()
```

## WORKED EXAMPLE

Parsing raw order timestamps, computing "days since order," and formatting for a report:

```python
from datetime import datetime, timedelta

raw_orders = [
    {"id": 1, "order_date": "2024-10-15"},
    {"id": 2, "order_date": "2024-11-20"},
    {"id": 3, "order_date": "2024-11-28"},
]

# Assume "today" for this example
today = datetime(2024, 12, 1)

for order in raw_orders:
    # Parse the raw string into a real datetime
    parsed_date = datetime.strptime(order["order_date"], "%Y-%m-%d")
    order["parsed_date"] = parsed_date

    # Date arithmetic: how many days ago was this order?
    days_ago = (today - parsed_date).days
    order["days_ago"] = days_ago

    # Formatting for a human-readable report
    order["display_date"] = parsed_date.strftime("%B %d, %Y")

for order in raw_orders:
    print(f"Order {order['id']}: {order['display_date']} ({order['days_ago']} days ago)")

# Order 1: October 15, 2024 (47 days ago)
# Order 2: November 20, 2024 (11 days ago)
# Order 3: November 28, 2024 (3 days ago)

# Filtering: orders placed in the last 14 days
recent_orders = [o for o in raw_orders if o["days_ago"] <= 14]
print(f"Recent orders: {[o['id'] for o in recent_orders]}")   # [2, 3]

# A common business rule: flag orders as "stale" if pending over 7 days
delivery_deadline = raw_orders[0]["parsed_date"] + timedelta(days=7)
print(f"Order 1 delivery deadline: {delivery_deadline.strftime('%Y-%m-%d')}")
# Order 1 delivery deadline: 2024-10-22
```

## COMMON MISTAKES

**1. Guessing the format string instead of matching it exactly**
```python
datetime.strptime("11/03/2024", "%Y-%m-%d")   # ValueError — format doesn't match the string
```
Fix: the format string must describe the *exact* shape of the input, including separators. `"11/03/2024"` needs `"%m/%d/%Y"`, not `"%Y-%m-%d"`.

**2. Ambiguous date formats (US vs. international)**
```python
"03/11/2024"   # Is this March 11th, or November 3rd? Depends on the source system!
```
Fix: always confirm the actual convention of your data source rather than assuming — a wrong guess here silently produces incorrect dates that are hard to catch later.

**3. Comparing a `datetime` to a `date` directly**
```python
from datetime import datetime, date
datetime.now() < date.today()   # TypeError — can't compare datetime and date directly
```
Fix: be consistent about which type you're working with; convert one to match the other if needed (`datetime.combine(date.today(), datetime.min.time())`).

**4. Forgetting `.strftime()` returns a string, not a date**
```python
formatted = order_date.strftime("%Y-%m-%d")
formatted + timedelta(days=1)   # TypeError — formatted is a string now, not a datetime
```
Fix: do all date arithmetic *before* formatting for display — once you've called `.strftime()`, you have plain text again, not a date object.

**5. Ignoring time zones when timestamps come from different sources**
Raw timestamps from different systems (a database in UTC, a user's local browser event) can silently represent different absolute moments even if they "look" similar. This course treats timezone-naive dates for simplicity, but be aware that production pipelines dealing with multiple data sources often need explicit timezone handling (`datetime`'s `tzinfo`, or the `zoneinfo` module).

**Data Engineer Tip:** "Parse this date string, then compute days-since or filter by range" is one of the single most common transformation steps in any pipeline — it shows up in cohort analysis, SLA monitoring, churn detection, and reporting. Getting comfortable with `strptime`/`strftime` format codes now means you won't need to look them up every time you hit a `to_datetime()` call in pandas (Module 5).

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Parse the string `"2024-06-15"` into a `datetime` object and print its `year`, `month`, and `day` separately.

<details>
<summary>Hint</summary>
`datetime.strptime("2024-06-15", "%Y-%m-%d")`
</details>

<details>
<summary>Solution</summary>

```python
from datetime import datetime

d = datetime.strptime("2024-06-15", "%Y-%m-%d")
print(d.year, d.month, d.day)   # 2024 6 15
```
</details>

---

**Exercise 2 (Easy):** Given `order_date = datetime(2024, 3, 10)`, format it as `"March 10, 2024"`.

<details>
<summary>Hint</summary>
`.strftime("%B %d, %Y")`
</details>

<details>
<summary>Solution</summary>

```python
from datetime import datetime

order_date = datetime(2024, 3, 10)
print(order_date.strftime("%B %d, %Y"))   # "March 10, 2024"
```
</details>

---

**Exercise 3 (Medium):** Given `order_date = datetime(2024, 11, 20)`, calculate and print the date exactly 30 days later, formatted as `"YYYY-MM-DD"`.

<details>
<summary>Hint</summary>
`order_date + timedelta(days=30)`, then `.strftime("%Y-%m-%d")`
</details>

<details>
<summary>Solution</summary>

```python
from datetime import datetime, timedelta

order_date = datetime(2024, 11, 20)
later = order_date + timedelta(days=30)
print(later.strftime("%Y-%m-%d"))   # "2024-12-20"
```
</details>

---

**Exercise 4 (Medium):** Given `signup_date = datetime(2024, 1, 15)` and `today = datetime(2024, 12, 1)`, calculate how many days the customer has been signed up, and print `"Customer since: X days"`.

<details>
<summary>Hint</summary>
Subtract the two `datetime` objects — the result has a `.days` attribute.
</details>

<details>
<summary>Solution</summary>

```python
from datetime import datetime

signup_date = datetime(2024, 1, 15)
today = datetime(2024, 12, 1)

days_signed_up = (today - signup_date).days
print(f"Customer since: {days_signed_up} days")   # 321 days
```
</details>

---

**Exercise 5 (Hard):** Given a list of raw order timestamps as strings, in ISO format with time included: `raw_timestamps = ["2024-11-01T09:15:00", "2024-11-15T18:42:00", "2024-11-29T23:10:00"]`, and `today = datetime(2024, 12, 1)`:
- Parse each into a `datetime`
- Compute `days_ago` for each
- Classify each as `"this_week"` if `days_ago <= 7`, `"this_month"` if `days_ago <= 30`, otherwise `"older"`
- Print a list of dicts with `timestamp`, `days_ago`, and `classification` for each

<details>
<summary>Hint</summary>
Parse with format `"%Y-%m-%dT%H:%M:%S"`. Use an `if/elif/else` chain (Lesson 1.4) for the classification, checking the most restrictive condition first.
</details>

<details>
<summary>Solution</summary>

```python
from datetime import datetime

raw_timestamps = [
    "2024-11-01T09:15:00",
    "2024-11-15T18:42:00",
    "2024-11-29T23:10:00",
]
today = datetime(2024, 12, 1)

results = []
for ts in raw_timestamps:
    parsed = datetime.strptime(ts, "%Y-%m-%dT%H:%M:%S")
    days_ago = (today - parsed).days

    if days_ago <= 7:
        classification = "this_week"
    elif days_ago <= 30:
        classification = "this_month"
    else:
        classification = "older"

    results.append({
        "timestamp": ts,
        "days_ago": days_ago,
        "classification": classification,
    })

for r in results:
    print(r)

# {'timestamp': '2024-11-01T09:15:00', 'days_ago': 30, 'classification': 'this_month'}
# {'timestamp': '2024-11-15T18:42:00', 'days_ago': 16, 'classification': 'this_month'}
# {'timestamp': '2024-11-29T23:10:00', 'days_ago': 2, 'classification': 'this_week'}
```
</details>

---

**Previous lesson:** [2.3 — Comprehensions](lesson-2.3-comprehensions.md)
**Next lesson:** [2.5 — Reading & Writing CSV](lesson-2.5-csv.md)
