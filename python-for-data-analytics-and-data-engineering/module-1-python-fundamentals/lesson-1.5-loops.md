# Lesson 1.5 — Loops & Loop Control

## CONCEPT

A **loop** repeats a block of code multiple times. Python has two kinds:

- **`for` loop**: iterates over a known sequence (a list, a string, a range of numbers) — "do this once for each item."
- **`while` loop**: repeats as long as a condition stays `True` — "keep doing this until something changes."

```python
for item in [1, 2, 3]:
    print(item)

count = 0
while count < 3:
    print(count)
    count += 1
```

**Loop control** keywords let you interrupt normal flow:
- `break` — exit the loop immediately
- `continue` — skip the rest of this iteration, move to the next one

## WHY IT MATTERS

Before you learn pandas, `for` loops are how you'll process every row of a dataset — checking each order, validating each customer record, summing each transaction. Even though pandas replaces most row-by-row loops with faster vectorized operations, understanding loops is essential for: (1) understanding *why* vectorization is faster, (2) handling the cases pandas can't vectorize cleanly (like calling an external API per row), and (3) general-purpose scripting outside of dataframes entirely — file processing, log parsing, automation.

## SYNTAX REFERENCE

```python
# for loop over a list
for product in products:
    print(product)

# for loop with range()
for i in range(5):          # 0, 1, 2, 3, 4
    print(i)

for i in range(1, 6):       # 1, 2, 3, 4, 5
    print(i)

for i in range(0, 10, 2):   # 0, 2, 4, 6, 8 (step of 2)
    print(i)

# for loop with index using enumerate()
for index, product in enumerate(products):
    print(index, product)

# for loop over a dictionary
for key, value in order.items():
    print(key, value)

# while loop
count = 0
while count < 5:
    print(count)
    count += 1

# break — exit early
for item in items:
    if item == "stop_here":
        break

# continue — skip this iteration
for item in items:
    if item is None:
        continue
    print(item)

# Loop with an accumulator pattern (very common)
total = 0
for price in prices:
    total += price
```

## WORKED EXAMPLE

Processing a list of raw order totals — summing valid ones, skipping bad data, and stopping early if a cancellation flag appears:

```python
raw_orders = [45.99, 120.00, "CANCELLED", 89.50, None, 200.00, 15.25]

total_revenue = 0.0
valid_order_count = 0
skipped_count = 0

for order in raw_orders:
    if order == "CANCELLED":
        print("Hit a cancellation marker — stopping processing.")
        break                       # exit the loop entirely

    if order is None:
        skipped_count += 1
        continue                    # skip this one, move to the next

    total_revenue += order
    valid_order_count += 1

print(f"Valid orders processed: {valid_order_count}")
print(f"Skipped (null) orders: {skipped_count}")
print(f"Total revenue so far: ${total_revenue:.2f}")

# Output:
# Valid orders processed: 2
# Skipped (null) orders: 0
# Total revenue so far: $165.99
```

Note that processing **stopped** at `"CANCELLED"` because of `break` — the `None` and remaining values after it were never reached. This is why `break` should be used deliberately, not casually.

A second example: using `enumerate()` to number rows while scanning for a data quality issue.

```python
prices = [24.99, 15.50, -5.00, 89.00, 12.00]

for index, price in enumerate(prices):
    if price < 0:
        print(f"Data quality issue at row {index}: negative price {price}")
```

## COMMON MISTAKES

**1. Off-by-one errors with `range()`**
```python
for i in range(5):
    print(i)   # 0,1,2,3,4 — NOT including 5
```
`range(n)` stops *before* `n`. This surprises almost every beginner at least once.

**2. Infinite `while` loops**
```python
count = 0
while count < 5:
    print(count)
    # forgot count += 1 — this loop never ends!
```
Fix: always make sure the loop's condition can eventually become `False`. Before running an unfamiliar `while` loop, double-check the update step exists.

**3. Modifying a list while iterating over it**
```python
items = [1, 2, 3, 4]
for item in items:
    if item % 2 == 0:
        items.remove(item)   # unpredictable — skips elements!
```
Fix: iterate over a copy (`for item in items[:]`) or build a new list instead of mutating the one you're looping over.

**4. Confusing `break` and `continue`**
`break` exits the *entire* loop. `continue` skips only the *current* iteration and keeps going. Mixing these up silently produces wrong totals — a very common source of subtle bugs.

**5. Off-by-reference: reusing an accumulator variable that wasn't reset**
```python
total = 100   # leftover value from earlier code
for price in prices:
    total += price   # starts from 100, not 0 — wrong total!
```
Fix: always explicitly initialize accumulator variables right before the loop that uses them.

**Data Engineer Tip:** In pandas, a `for` loop over rows (`.iterrows()`) is almost always the *slowest* way to process a DataFrame — vectorized operations (Module 4–5) can be 10–100x faster on large datasets. Learn loops thoroughly here because they're the right tool for smaller, non-tabular tasks (looping over files, API calls, config values) — but the moment you're processing thousands of rows of structured data, this is the pattern you'll be replacing.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Loop through `numbers = [4, 8, 15, 16, 23, 42]` and print each number doubled.

<details>
<summary>Hint</summary>
`for n in numbers: print(n * 2)`
</details>

<details>
<summary>Solution</summary>

```python
numbers = [4, 8, 15, 16, 23, 42]
for n in numbers:
    print(n * 2)
```
</details>

---

**Exercise 2 (Easy):** Use a `for` loop with `range()` to print the numbers 1 through 10.

<details>
<summary>Hint</summary>
`range(1, 11)` — remember range stops *before* the second argument.
</details>

<details>
<summary>Solution</summary>

```python
for i in range(1, 11):
    print(i)
```
</details>

---

**Exercise 3 (Medium):** Given `order_totals = [45.00, 120.50, 89.99, 200.00, 15.75]`, calculate and print the sum of all order totals using a loop (don't use the built-in `sum()` function — practice the accumulator pattern).

<details>
<summary>Hint</summary>
Initialize `total = 0` before the loop, then `total += value` inside it.
</details>

<details>
<summary>Solution</summary>

```python
order_totals = [45.00, 120.50, 89.99, 200.00, 15.75]
total = 0
for amount in order_totals:
    total += amount
print(f"Total: ${total:.2f}")
```
</details>

---

**Exercise 4 (Medium):** Given `statuses = ["shipped", "pending", "cancelled", "shipped", "delivered"]`, count how many orders are `"cancelled"` using a loop, and print the count.

<details>
<summary>Hint</summary>
Use an accumulator counter that increments only when `status == "cancelled"`.
</details>

<details>
<summary>Solution</summary>

```python
statuses = ["shipped", "pending", "cancelled", "shipped", "delivered"]
cancelled_count = 0
for status in statuses:
    if status == "cancelled":
        cancelled_count += 1
print(f"Cancelled orders: {cancelled_count}")
```
</details>

---

**Exercise 5 (Hard):** Given a list of raw order amounts that includes some invalid entries: `raw_amounts = [50, "N/A", 120, -10, None, 89.99, "CANCELLED"]`, write a loop that:
- Stops entirely (using `break`) if it encounters `"CANCELLED"`
- Skips (using `continue`) any value that's `None`, a negative number, or the string `"N/A"`
- Otherwise adds the value to a running total
- After the loop, print the total and how many valid values were summed

<details>
<summary>Hint</summary>
Check for `"CANCELLED"` first (break), then check for the invalid cases together (continue) — note you'll need to check the type or value carefully since the list is mixed types.
</details>

<details>
<summary>Solution</summary>

```python
raw_amounts = [50, "N/A", 120, -10, None, 89.99, "CANCELLED"]

total = 0
valid_count = 0

for amount in raw_amounts:
    if amount == "CANCELLED":
        break

    if amount is None or amount == "N/A":
        continue

    if isinstance(amount, (int, float)) and amount < 0:
        continue

    total += amount
    valid_count += 1

print(f"Total: {total}, valid values summed: {valid_count}")
# Total: 170, valid values summed: 2
```
</details>

---

**Previous lesson:** [1.4 — Conditional Logic](lesson-1.4-conditionals.md)
**Next lesson:** [1.6 — Functions](lesson-1.6-functions.md)
