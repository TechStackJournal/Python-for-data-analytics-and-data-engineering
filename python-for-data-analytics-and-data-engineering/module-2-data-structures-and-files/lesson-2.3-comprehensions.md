# Lesson 2.3 — Comprehensions

## CONCEPT

A **comprehension** is a compact, one-line way to build a list, dictionary, or set by transforming and/or filtering another iterable — replacing the "create empty collection, loop, append" pattern from Module 1 with a single expression.

```python
# Loop version
squares = []
for n in range(5):
    squares.append(n ** 2)

# Comprehension version — same result
squares = [n ** 2 for n in range(5)]
```

There are three flavors, matching the three collection types from this module: **list comprehensions**, **dict comprehensions**, and **set comprehensions**.

## WHY IT MATTERS

Comprehensions are everywhere in real Python code — cleaning a column of raw values, filtering a list of records, building a lookup dictionary from a list of rows. They're not just shorter; once you're fluent reading them, they're *faster to understand* than the equivalent loop, because the intent ("build a new list by transforming this one") is stated upfront instead of buried inside a loop body. They're also a direct mental bridge to vectorized pandas operations in Module 4 — both express "do this to every row" without an explicit loop.

## SYNTAX REFERENCE

```python
# List comprehension
[expression for item in iterable]
[expression for item in iterable if condition]

squares = [n ** 2 for n in range(5)]                    # [0, 1, 4, 9, 16]
evens = [n for n in range(10) if n % 2 == 0]              # [0, 2, 4, 6, 8]
upper_names = [name.upper() for name in ["a", "b"]]       # ['A', 'B']

# Dict comprehension
{key_expr: value_expr for item in iterable}

squared_lookup = {n: n ** 2 for n in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

prices_by_name = {p["name"]: p["price"] for p in products}

# Set comprehension
{expression for item in iterable}

unique_lengths = {len(name) for name in ["cat", "dog", "fish"]}   # {3, 4}

# Conditional expression INSIDE a comprehension (if/else, not filtering)
labels = ["high" if price > 50 else "low" for price in [20, 80, 45, 99]]
# ['low', 'high', 'low', 'high']

# Nested comprehension (use sparingly — readability matters)
flattened = [item for sublist in nested_list for item in sublist]
```

## WORKED EXAMPLE

Transforming a raw list of product dictionaries using comprehensions instead of manual loops:

```python
products = [
    {"name": "Mouse", "price": 24.99, "category": "Electronics", "in_stock": True},
    {"name": "Lamp", "price": 34.00, "category": "Home", "in_stock": False},
    {"name": "Keyboard", "price": 59.99, "category": "Electronics", "in_stock": True},
    {"name": "Yoga Mat", "price": 19.99, "category": "Outdoor", "in_stock": True},
]

# List comprehension: names of in-stock products only
in_stock_names = [p["name"] for p in products if p["in_stock"]]
print(in_stock_names)   # ['Mouse', 'Keyboard', 'Yoga Mat']

# List comprehension: prices with a 10% markup applied
marked_up_prices = [round(p["price"] * 1.10, 2) for p in products]
print(marked_up_prices)   # [27.49, 37.4, 65.99, 21.99]

# Dict comprehension: quick name -> price lookup table
price_lookup = {p["name"]: p["price"] for p in products}
print(price_lookup["Keyboard"])   # 59.99

# Set comprehension: distinct categories present
categories = {p["category"] for p in products}
print(categories)   # {'Electronics', 'Home', 'Outdoor'}

# Conditional expression inside a comprehension: label each product by price tier
price_tiers = ["premium" if p["price"] > 30 else "budget" for p in products]
print(price_tiers)   # ['budget', 'premium', 'premium', 'budget']

# Combining filter + transform: names of Electronics products, uppercased
electronics_upper = [p["name"].upper() for p in products if p["category"] == "Electronics"]
print(electronics_upper)   # ['MOUSE', 'KEYBOARD']
```

## COMMON MISTAKES

**1. Overusing comprehensions until they become unreadable**
```python
result = [x*2 for x in [y+1 for y in range(10) if y % 2 == 0] if x > 5]
```
If a comprehension needs a comment to explain what it does, it's usually clearer as a regular loop, or broken into two steps. Fix: readability beats compactness — don't nest comprehensions more than one level deep in real code.

**2. Confusing the filtering `if` with the conditional `if/else` expression**
```python
[x for x in nums if x > 0]              # FILTERS — only keeps matching items
[x if x > 0 else 0 for x in nums]       # TRANSFORMS — keeps all items, substitutes some
```
These look similar but do fundamentally different things — mixing them up is a common source of subtle bugs.

**3. Forgetting a dict comprehension needs both a key and a value expression**
```python
{p["name"] for p in products}          # this is a SET comprehension (only one expression)
{p["name"]: p["price"] for p in products}   # this is a DICT comprehension (key: value)
```
The `:` is what distinguishes them — easy to typo when switching between the two.

**4. Using a comprehension for side effects instead of building a collection**
```python
[print(name) for name in names]   # works, but creates a wasted list of Nones
```
Fix: if you're not using the resulting list, use a regular `for` loop instead — comprehensions are for *building* a new collection, not just running code repeatedly.

**Data Engineer Tip:** A list comprehension filtering and transforming a list of dicts, like `[p["name"] for p in products if p["in_stock"]]`, is doing exactly what `df[df["in_stock"]]["name"]` does in pandas — just row-by-row instead of vectorized. Recognizing this parallel makes pandas filtering syntax feel far less arbitrary once you get there in Module 4.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `numbers = [1, 2, 3, 4, 5, 6, 7, 8]`, use a list comprehension to build a list of only the even numbers.

<details>
<summary>Hint</summary>
`[n for n in numbers if n % 2 == 0]`
</details>

<details>
<summary>Solution</summary>

```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8]
evens = [n for n in numbers if n % 2 == 0]
print(evens)   # [2, 4, 6, 8]
```
</details>

---

**Exercise 2 (Easy):** Given `names = ["maria", "james", "priya"]`, use a list comprehension to build a list of the same names, title-cased.

<details>
<summary>Hint</summary>
`[name.title() for name in names]`
</details>

<details>
<summary>Solution</summary>

```python
names = ["maria", "james", "priya"]
titled = [name.title() for name in names]
print(titled)   # ['Maria', 'James', 'Priya']
```
</details>

---

**Exercise 3 (Medium):** Given `prices = [24.99, 89.00, 12.50, 145.00, 5.75]`, use a list comprehension to label each price as `"expensive"` if over 50, otherwise `"affordable"`.

<details>
<summary>Hint</summary>
Use the ternary form: `["expensive" if p > 50 else "affordable" for p in prices]`
</details>

<details>
<summary>Solution</summary>

```python
prices = [24.99, 89.00, 12.50, 145.00, 5.75]
labels = ["expensive" if p > 50 else "affordable" for p in prices]
print(labels)
# ['affordable', 'expensive', 'affordable', 'expensive', 'affordable']
```
</details>

---

**Exercise 4 (Medium):** Given a list of order dicts:
```python
orders = [
    {"id": 1, "total": 45.00, "status": "shipped"},
    {"id": 2, "total": 120.50, "status": "pending"},
    {"id": 3, "total": 89.99, "status": "shipped"},
]
```
Build a dictionary comprehension mapping each shipped order's `id` to its `total` (skip non-shipped orders).

<details>
<summary>Hint</summary>
`{o["id"]: o["total"] for o in orders if o["status"] == "shipped"}`
</details>

<details>
<summary>Solution</summary>

```python
orders = [
    {"id": 1, "total": 45.00, "status": "shipped"},
    {"id": 2, "total": 120.50, "status": "pending"},
    {"id": 3, "total": 89.99, "status": "shipped"},
]

shipped_totals = {o["id"]: o["total"] for o in orders if o["status"] == "shipped"}
print(shipped_totals)   # {1: 45.0, 3: 89.99}
```
</details>

---

**Exercise 5 (Hard):** Given a list of nested order dicts:
```python
orders = [
    {"id": 1, "items": [{"price": 10, "qty": 2}, {"price": 5, "qty": 1}]},
    {"id": 2, "items": [{"price": 20, "qty": 1}]},
    {"id": 3, "items": [{"price": 8, "qty": 3}, {"price": 12, "qty": 2}]},
]
```
Using comprehensions (you may use more than one, or nest them), build a dictionary mapping each order's `id` to its total value (sum of `price * qty` across all its items). Print the result: `{1: 25, 2: 20, 3: 48}`.

<details>
<summary>Hint</summary>
For each order, compute its total with a nested list comprehension inside `sum()`: `sum(item["price"] * item["qty"] for item in order["items"])`. Then wrap the whole thing in a dict comprehension over `orders`.
</details>

<details>
<summary>Solution</summary>

```python
orders = [
    {"id": 1, "items": [{"price": 10, "qty": 2}, {"price": 5, "qty": 1}]},
    {"id": 2, "items": [{"price": 20, "qty": 1}]},
    {"id": 3, "items": [{"price": 8, "qty": 3}, {"price": 12, "qty": 2}]},
]

order_totals = {
    order["id"]: sum(item["price"] * item["qty"] for item in order["items"])
    for order in orders
}

print(order_totals)   # {1: 25, 2: 20, 3: 48}
```
Note: `sum(... for ... in ...)` without brackets is a **generator expression** — a close cousin of a list comprehension that produces values one at a time instead of building a full list in memory. It's a natural next thing to look into once comprehensions feel comfortable, but isn't required for this course.
</details>

---

**Previous lesson:** [2.2 — Dictionaries & Nested Structures](lesson-2.2-dictionaries.md)
**Next lesson:** [2.4 — Dates & Times](lesson-2.4-datetime.md)
