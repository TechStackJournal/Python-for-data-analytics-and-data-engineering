# Lesson 2.1 — Lists, Tuples & Sets

## CONCEPT

Python has three core built-in collection types for grouping multiple values together:

- **List** — an ordered, mutable (changeable) collection. Use when order matters and the contents may change.
  ```python
  products = ["Mouse", "Keyboard", "Monitor"]
  ```
- **Tuple** — an ordered, immutable (unchangeable) collection. Use for fixed groupings that shouldn't be modified after creation.
  ```python
  coordinates = (40.7128, -74.0060)
  ```
- **Set** — an unordered collection of unique values, with no duplicates. Use when you only care about membership or uniqueness, not order.
  ```python
  unique_categories = {"Electronics", "Home", "Outdoor"}
  ```

## WHY IT MATTERS

Almost every dataset you touch is, underneath, a list of records. A `pandas` DataFrame is essentially a list of rows with column labels attached. Tuples show up constantly for fixed pairs (like `(price, quantity)` you used in Module 1). Sets are the fastest way to answer "does this value already exist?" or "what are the distinct categories in this data?" — a task you'll do constantly during data exploration.

## SYNTAX REFERENCE

```python
# Lists — ordered, mutable
products = ["Mouse", "Keyboard", "Monitor"]
products.append("Webcam")           # add to the end
products.insert(0, "Cable")         # insert at a specific position
products.remove("Keyboard")         # remove by value
products.pop()                      # remove and return the last item
products[0]                         # index access
products[-1]                        # last item
products[1:3]                       # slicing
len(products)                       # length
"Mouse" in products                 # membership check
products.sort()                     # sort in place
sorted(products)                    # return a new sorted list, original unchanged

# Tuples — ordered, immutable
point = (10, 20)
point[0]              # access works like a list
# point[0] = 5         # TypeError — tuples can't be modified
x, y = point           # unpacking

# Sets — unordered, unique values only
categories = {"Electronics", "Home", "Electronics"}   # duplicate dropped automatically
categories.add("Outdoor")
categories.remove("Home")
"Electronics" in categories          # very fast membership check

# Set operations
a = {1, 2, 3}
b = {2, 3, 4}
a | b        # union: {1, 2, 3, 4}
a & b        # intersection: {2, 3}
a - b        # difference: {1}

# Converting between types
list({1, 2, 3})       # set -> list
set([1, 2, 2, 3])      # list -> set (drops duplicates)
tuple([1, 2, 3])       # list -> tuple
```

## WORKED EXAMPLE

Representing a batch of order line items and finding unique categories purchased:

```python
# Each order line as a tuple: (product_name, category, price, quantity)
order_lines = [
    ("Wireless Mouse", "Electronics", 24.99, 2),
    ("Yoga Mat", "Outdoor", 19.99, 1),
    ("USB-C Cable", "Electronics", 12.50, 3),
    ("Desk Lamp", "Home", 34.00, 1),
]

# Lists: build a running list of line totals
line_totals = []
for name, category, price, qty in order_lines:
    line_totals.append(round(price * qty, 2))

print(line_totals)          # [49.98, 19.99, 37.5, 34.0]
print(sum(line_totals))     # 141.47

# Sets: find the distinct categories in this order
categories_purchased = set()
for name, category, price, qty in order_lines:
    categories_purchased.add(category)

print(categories_purchased)   # {'Electronics', 'Outdoor', 'Home'} (order not guaranteed)

# Tuples: unpacking a single record cleanly
first_item = order_lines[0]
name, category, price, qty = first_item
print(f"{name}: ${price} x {qty}")   # Wireless Mouse: $24.99 x 2

# Set operations: compare this order's categories against a promo list
promo_categories = {"Electronics", "Outdoor", "Books"}
matching_categories = categories_purchased & promo_categories
print(f"Categories eligible for promo: {matching_categories}")  # {'Electronics', 'Outdoor'}
```

## COMMON MISTAKES

**1. Trying to modify a tuple**
```python
point = (10, 20)
point[0] = 99   # TypeError: 'tuple' object does not support item assignment
```
Fix: if you need to change values, use a list instead — reach for a tuple specifically *because* you want it fixed.

**2. Expecting sets to preserve order**
```python
s = {"c", "a", "b"}
print(s)   # order is NOT guaranteed to match insertion order
```
Fix: if order matters, use a list (or, from Python 3.7+, a dict — covered in Lesson 2.2 — which does preserve insertion order, unlike sets).

**3. Trying to put unhashable types (like a list) inside a set**
```python
s = {[1, 2], [3, 4]}   # TypeError: unhashable type: 'list'
```
Fix: use tuples instead of lists as set elements if you need to group multiple values: `s = {(1, 2), (3, 4)}`.

**4. Using `==` to compare a list and a tuple with the same values**
```python
[1, 2, 3] == (1, 2, 3)   # False — different types are never equal, even with the same contents
```

**5. Forgetting duplicates silently disappear in a set**
```python
categories = {"Electronics", "electronics"}   # NOT deduplicated — case-sensitive!
len(categories)   # 2, not 1
```
Fix: normalize case (`.lower()`) *before* adding to a set if you want true duplicates removed regardless of casing.

**Data Engineer Tip:** Sets are the standard tool for a very common ETL check: "which IDs are in the source system but missing from the target?" — `source_ids - target_ids` instantly gives you exactly that, using the same `-` difference operator shown above. This is a real technique used for reconciliation checks between two data sources.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `prices = [24.99, 12.50, 89.00, 5.75]`, append `15.00` to the list, then print the full list and its length.

<details>
<summary>Hint</summary>
`.append()`, then `len()`.
</details>

<details>
<summary>Solution</summary>

```python
prices = [24.99, 12.50, 89.00, 5.75]
prices.append(15.00)
print(prices, len(prices))
```
</details>

---

**Exercise 2 (Easy):** Given `coordinates = (40.7128, -74.0060)`, unpack it into `latitude` and `longitude` variables and print them.

<details>
<summary>Hint</summary>
`latitude, longitude = coordinates`
</details>

<details>
<summary>Solution</summary>

```python
coordinates = (40.7128, -74.0060)
latitude, longitude = coordinates
print(latitude, longitude)
```
</details>

---

**Exercise 3 (Medium):** Given `raw_categories = ["Electronics", "Home", "electronics", "Outdoor", "HOME"]`, produce a set of truly unique categories, normalized to title case (so `"electronics"` and `"Electronics"` count as the same). Print the resulting set and its size.

<details>
<summary>Hint</summary>
Loop through the list, apply `.title()` to each value, and add each to a set.
</details>

<details>
<summary>Solution</summary>

```python
raw_categories = ["Electronics", "Home", "electronics", "Outdoor", "HOME"]

unique_categories = set()
for cat in raw_categories:
    unique_categories.add(cat.title())

print(unique_categories, len(unique_categories))
# {'Electronics', 'Home', 'Outdoor'}  3
```
</details>

---

**Exercise 4 (Medium):** Given two sets of customer IDs — `source_ids = {101, 102, 103, 104}` (from the source system) and `target_ids = {102, 103, 105}` (already loaded into the warehouse) — find which IDs are missing from the target (need to be loaded) and which are in the target but no longer in the source (possibly deleted upstream). Print both.

<details>
<summary>Hint</summary>
Missing from target: `source_ids - target_ids`. Extra in target: `target_ids - source_ids`.
</details>

<details>
<summary>Solution</summary>

```python
source_ids = {101, 102, 103, 104}
target_ids = {102, 103, 105}

missing_from_target = source_ids - target_ids
extra_in_target = target_ids - source_ids

print("Need to load:", missing_from_target)     # {101, 104}
print("No longer in source:", extra_in_target)  # {105}
```
</details>

---

**Exercise 5 (Hard):** Given a list of order tuples `orders = [(1, "Electronics", 49.99), (2, "Home", 34.00), (3, "Electronics", 12.50), (4, "Outdoor", 19.99), (5, "Home", 89.00)]` (each: `id, category, total`), do the following without using pandas:
- Build a list of just the totals for `"Electronics"` orders
- Compute the sum and average of those totals (rounded to 2 decimals)
- Build a set of all distinct categories present
- Print all three results

<details>
<summary>Hint</summary>
Loop once, filtering by category with `if` inside the loop, appending matching totals to a list, and adding every category to a set regardless of the filter.
</details>

<details>
<summary>Solution</summary>

```python
orders = [
    (1, "Electronics", 49.99),
    (2, "Home", 34.00),
    (3, "Electronics", 12.50),
    (4, "Outdoor", 19.99),
    (5, "Home", 89.00),
]

electronics_totals = []
all_categories = set()

for order_id, category, total in orders:
    all_categories.add(category)
    if category == "Electronics":
        electronics_totals.append(total)

total_sum = round(sum(electronics_totals), 2)
average = round(total_sum / len(electronics_totals), 2)

print("Electronics totals:", electronics_totals)   # [49.99, 12.50]
print("Sum:", total_sum, "Average:", average)        # Sum: 62.49 Average: 31.25
print("All categories:", all_categories)             # {'Electronics', 'Home', 'Outdoor'}
```
</details>

---

**Next lesson:** [2.2 — Dictionaries & Nested Structures](lesson-2.2-dictionaries.md)
