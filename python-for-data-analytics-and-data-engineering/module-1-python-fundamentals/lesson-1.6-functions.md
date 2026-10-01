# Lesson 1.6 — Functions

## CONCEPT

A **function** is a named, reusable block of code that performs a specific task. You define it once with `def`, and then **call** it as many times as you need, optionally passing in different inputs (**parameters**) and getting back a result (**return value**).

```python
def greet(name):
    return f"Hello, {name}!"

message = greet("Maria")   # calling the function
print(message)             # "Hello, Maria!"
```

Functions are the primary tool for avoiding repeated code and for organizing a program into clear, testable pieces.

## WHY IT MATTERS

Every transformation step in a data pipeline — cleaning a column, validating a row, calculating a derived metric — should live in its own function. This isn't just style: it means you can test each step in isolation, reuse it across scripts, and reason about a 500-line pipeline as a handful of well-named functions instead of one long wall of code. Module 8 (Production-Grade Python) builds heavily on the habits you form here.

## SYNTAX REFERENCE

```python
# Basic function
def calculate_total(price, quantity):
    return price * quantity

# Default parameter values
def calculate_total(price, quantity=1):
    return price * quantity

calculate_total(9.99)          # quantity defaults to 1
calculate_total(9.99, 3)       # quantity = 3

# Keyword arguments (order doesn't matter when named)
calculate_total(quantity=3, price=9.99)

# Multiple return values (returns a tuple)
def get_min_max(numbers):
    return min(numbers), max(numbers)

low, high = get_min_max([4, 8, 15, 16, 23, 42])

# *args — accept any number of positional arguments
def total_all(*amounts):
    return sum(amounts)

total_all(10, 20, 30)   # 60

# **kwargs — accept any number of named arguments
def build_order(**fields):
    return fields

build_order(id=1, total=59.99, status="shipped")
# {'id': 1, 'total': 59.99, 'status': 'shipped'}

# A function with no explicit return implicitly returns None
def log_message(msg):
    print(msg)

result = log_message("hi")   # result is None

# Docstrings — document what a function does
def apply_discount(price, pct):
    """Return price reduced by pct percent (e.g. pct=10 means 10% off)."""
    return price * (1 - pct / 100)
```

## WORKED EXAMPLE

Building a small toolkit of reusable functions for processing order data:

```python
def clean_price(raw_price):
    """Convert a raw price string like '$24.99' into a float."""
    if isinstance(raw_price, str):
        raw_price = raw_price.replace("$", "").strip()
    return float(raw_price)


def calculate_line_total(unit_price, quantity, discount_pct=0):
    """Calculate the total for a line item, with an optional discount."""
    subtotal = unit_price * quantity
    discount_amount = subtotal * (discount_pct / 100)
    return subtotal - discount_amount


def classify_order_size(item_count):
    """Return a size label based on item count."""
    if item_count >= 10:
        return "bulk"
    elif item_count >= 3:
        return "standard"
    else:
        return "small"


def summarize_order(raw_price, quantity, discount_pct=0):
    """Combine the helpers above into one order summary."""
    unit_price = clean_price(raw_price)
    total = calculate_line_total(unit_price, quantity, discount_pct)
    size = classify_order_size(quantity)
    return {
        "unit_price": unit_price,
        "quantity": quantity,
        "total": round(total, 2),
        "size": size,
    }


# Using the functions together
summary = summarize_order("$24.99", quantity=5, discount_pct=10)
print(summary)
# {'unit_price': 24.99, 'quantity': 5, 'total': 112.46, 'size': 'standard'}

# Reusing clean_price() independently, on a different field
tax_raw = "$7.50"
tax = clean_price(tax_raw)
print(tax)   # 7.5
```

Notice how `summarize_order()` calls the three smaller functions instead of repeating their logic inline — each function does one clear thing, and can be tested and reused on its own.

## COMMON MISTAKES

**1. Forgetting to `return` a value**
```python
def calculate_total(price, quantity):
    total = price * quantity   # computed, but never returned!

result = calculate_total(10, 2)
print(result)   # None
```
Fix: always `return` the value you intend to use outside the function.

**2. Confusing `print()` inside a function with `return`**
```python
def get_total(price, quantity):
    print(price * quantity)   # displays it, but doesn't hand it back

x = get_total(10, 2)   # x is None — print() is not the same as return
```
`print()` shows something to the user; `return` hands a value back to the calling code so it can be used further.

**3. Mutable default arguments (a classic Python gotcha)**
```python
def add_item(item, cart=[]):    # DANGER: default list is shared across calls!
    cart.append(item)
    return cart

add_item("apple")    # ['apple']
add_item("banana")   # ['apple', 'banana'] — unexpected! Same list reused.
```
Fix: use `None` as the default, and create a new list inside the function:
```python
def add_item(item, cart=None):
    if cart is None:
        cart = []
    cart.append(item)
    return cart
```

**4. Shadowing built-in names**
```python
def sum(numbers):   # overwrites Python's built-in sum() in this scope
    ...
```
Fix: avoid naming your own functions/variables the same as built-ins like `sum`, `list`, `type`, `id`, `min`, `max`.

**5. Too many responsibilities in one function**
A function that cleans data, calculates a total, *and* prints a report is hard to test and reuse. Prefer small, single-purpose functions (like the worked example above) over one large one.

**Data Engineer Tip:** In Module 8, you'll learn to write `pytest` unit tests for functions like these. A function that takes clear inputs and returns a clear output (no hidden side effects like printing or modifying global state) is called a "pure function" — and pure functions are dramatically easier to test, debug, and reuse across pipeline stages. Start building that habit now, even before you're formally testing anything.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Write a function `square(n)` that returns `n` squared. Call it with `5` and print the result.

<details>
<summary>Hint</summary>
`return n * n` or `return n ** 2`
</details>

<details>
<summary>Solution</summary>

```python
def square(n):
    return n ** 2

print(square(5))   # 25
```
</details>

---

**Exercise 2 (Easy):** Write a function `calculate_tax(amount, tax_rate=0.08)` that returns the tax amount (not the total). Call it once using the default rate, and once passing `0.05` explicitly.

<details>
<summary>Hint</summary>
`return amount * tax_rate`
</details>

<details>
<summary>Solution</summary>

```python
def calculate_tax(amount, tax_rate=0.08):
    return amount * tax_rate

print(calculate_tax(100))         # 8.0
print(calculate_tax(100, 0.05))   # 5.0
```
</details>

---

**Exercise 3 (Medium):** Write a function `is_valid_email(email)` that returns `True` if the string contains both `"@"` and `"."`, otherwise `False`. Test it against `"maria@example.com"` and `"not-an-email"`.

<details>
<summary>Hint</summary>
Combine two `in` checks with `and`.
</details>

<details>
<summary>Solution</summary>

```python
def is_valid_email(email):
    return "@" in email and "." in email

print(is_valid_email("maria@example.com"))   # True
print(is_valid_email("not-an-email"))        # False
```
</details>

---

**Exercise 4 (Medium):** Write a function `order_summary(items)` that takes a list of `(price, quantity)` tuples and returns the total cost across all of them. Test it with `[(9.99, 2), (24.99, 1), (5.00, 3)]`.

<details>
<summary>Hint</summary>
Loop through the list, unpacking each tuple into `price, quantity`, and accumulate a total.
</details>

<details>
<summary>Solution</summary>

```python
def order_summary(items):
    total = 0
    for price, quantity in items:
        total += price * quantity
    return total

result = order_summary([(9.99, 2), (24.99, 1), (5.00, 3)])
print(f"${result:.2f}")   # $59.97
```
</details>

---

**Exercise 5 (Hard):** Write a function `apply_tiered_discount(order_total, tier)` where `tier` is one of `"gold"`, `"silver"`, `"bronze"`, or anything else (treated as no discount). It should return a dictionary with `original_total`, `discount_pct`, and `final_total` (rounded to 2 decimals). Discount rates: gold=20, silver=10, bronze=5, other=0. Then write a second function `summarize_orders(orders)` that takes a list of `(order_total, tier)` tuples, calls `apply_tiered_discount` on each, and returns the grand total of all `final_total` values. Test with `[(100, "gold"), (50, "silver"), (30, "none")]`.

<details>
<summary>Hint</summary>
Build `apply_tiered_discount` first and test it alone before writing `summarize_orders`, which should call it in a loop and accumulate `final_total`.
</details>

<details>
<summary>Solution</summary>

```python
def apply_tiered_discount(order_total, tier):
    rates = {"gold": 20, "silver": 10, "bronze": 5}
    discount_pct = rates.get(tier, 0)
    final_total = order_total * (1 - discount_pct / 100)
    return {
        "original_total": order_total,
        "discount_pct": discount_pct,
        "final_total": round(final_total, 2),
    }


def summarize_orders(orders):
    grand_total = 0
    for order_total, tier in orders:
        result = apply_tiered_discount(order_total, tier)
        grand_total += result["final_total"]
    return round(grand_total, 2)


orders = [(100, "gold"), (50, "silver"), (30, "none")]
print(summarize_orders(orders))   # 80 + 45 + 30 = 155.0
```
</details>

---

**Previous lesson:** [1.5 — Loops & Loop Control](lesson-1.5-loops.md)
**Next lesson:** [1.7 — Scope & Namespaces](lesson-1.7-scope.md)
