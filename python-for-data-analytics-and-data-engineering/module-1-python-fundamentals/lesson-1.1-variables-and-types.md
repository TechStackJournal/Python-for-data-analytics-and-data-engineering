# Lesson 1.1 — Variables, Data Types & Type Conversion

## CONCEPT

A **variable** is a name that points to a value stored in memory. Think of it as a labeled box: you put a value in the box, and from then on you can refer to that value by its label instead of retyping it.

```python
customer_name = "Maria Chen"
```

Here, `customer_name` is the label, and `"Maria Chen"` is the value stored. Python figures out the **data type** of a value automatically — this is called **dynamic typing**. The core built-in types you'll use constantly are:

| Type | Example | Meaning |
|---|---|---|
| `int` | `42` | whole numbers |
| `float` | `19.99` | decimal numbers |
| `str` | `"Maria Chen"` | text |
| `bool` | `True` / `False` | yes/no, on/off |
| `NoneType` | `None` | explicitly "no value" |

**Type conversion** (also called casting) is changing a value from one type to another — e.g., turning the text `"42"` into the number `42`.

## WHY IT MATTERS

Raw data almost never arrives in the type you need. A CSV export will hand you `"19.99"` as *text*, not as a number, because everything read from a file starts life as a string. If you try to do math on that text directly, Python will refuse — and understanding *why* it refuses, and how to fix it, is one of the most common day-to-day tasks in data work. Getting comfortable with types now prevents a huge share of bugs later, especially once you're cleaning real data in pandas (Module 4).

## SYNTAX REFERENCE

```python
# Assignment
variable_name = value

# Checking a type
type(value)              # returns the type, e.g. <class 'int'>

# Type conversion (casting)
int("42")                # "42" -> 42
float("19.99")           # "19.99" -> 19.99
str(42)                  # 42 -> "42"
bool(0)                  # 0 -> False
bool(1)                  # 1 -> True

# Multiple assignment
a, b, c = 1, 2, 3

# None — representing "no value yet"
discount_code = None
```

## WORKED EXAMPLE

Let's represent one row from the `ecommerce.products` table as plain Python variables:

```python
# Raw data as it might arrive from a CSV export — note everything is text
product_id_raw = "1042"
product_name = "Wireless Mouse"
unit_price_raw = "24.99"
in_stock_raw = "true"

# Convert to the correct types
product_id = int(product_id_raw)
unit_price = float(unit_price_raw)
in_stock = in_stock_raw.lower() == "true"   # a common pattern for text -> bool

print(type(product_id), product_id)     # <class 'int'> 1042
print(type(unit_price), unit_price)     # <class 'float'> 24.99
print(type(in_stock), in_stock)         # <class 'bool'> True

# Now math works correctly
quantity = 3
line_total = unit_price * quantity
print(f"Line total: ${line_total}")     # Line total: $74.97
```

Notice `in_stock_raw.lower() == "true"` — you can't just do `bool("false")`, because **any non-empty string is truthy in Python**, including the string `"false"`. This is exactly the kind of trap covered next.

## COMMON MISTAKES

**1. Trying to do math on a string that looks like a number**
```python
price = "24.99"
total = price * 2        # doesn't error! "24.9924.99" — string repetition
```
Fix: convert first — `total = float(price) * 2`.

**2. Assuming `bool("false")` is `False`**
```python
bool("false")   # True! Any non-empty string is truthy.
bool("")        # False — only an empty string is falsy.
```
Fix: compare explicitly, e.g. `text.lower() == "true"`.

**3. `int()` failing on decimal-looking strings**
```python
int("19.99")    # ValueError: invalid literal for int() with base 10: '19.99'
```
Fix: go through `float` first — `int(float("19.99"))` — understanding this truncates, it doesn't round.

**4. Confusing `=` (assignment) with `==` (comparison)**
```python
if price = 0:     # SyntaxError — this is assignment, not comparison
```
Fix: use `==` when comparing values.

**Data Engineer Tip:** When a pipeline reads from a CSV, JSON, or an API response, every field is a string or a loosely-typed value until you explicitly cast it. Silent type mismatches (e.g., a price column that's secretly a string) are one of the most common causes of pipeline bugs — always validate and cast types as early as possible in your ETL flow, not downstream where the error is harder to trace.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create variables for a customer's `first_name`, `last_name`, and `age` (as an integer), then print a sentence combining all three, e.g. `"Maria Chen is 29 years old."`

<details>
<summary>Hint</summary>
Use an f-string: `f"{first_name} {last_name} is {age} years old."`
</details>

<details>
<summary>Solution</summary>

```python
first_name = "Maria"
last_name = "Chen"
age = 29
print(f"{first_name} {last_name} is {age} years old.")
```
</details>

---

**Exercise 2 (Easy):** You're given `order_total_raw = "156.50"` as a string. Convert it to a float, then print its type and value.

<details>
<summary>Hint</summary>
Use `float()` and `type()`.
</details>

<details>
<summary>Solution</summary>

```python
order_total_raw = "156.50"
order_total = float(order_total_raw)
print(type(order_total), order_total)
```
</details>

---

**Exercise 3 (Medium):** Given `quantity_raw = "5"` and `unit_price_raw = "12.50"`, both strings, calculate and print the total cost as a float, formatted to two decimal places (hint: look up f-string number formatting, `f"{value:.2f}"`).

<details>
<summary>Hint</summary>
Convert both variables to numeric types first, then multiply. Use `f"{total:.2f}"` to format.
</details>

<details>
<summary>Solution</summary>

```python
quantity_raw = "5"
unit_price_raw = "12.50"

quantity = int(quantity_raw)
unit_price = float(unit_price_raw)
total = quantity * unit_price

print(f"Total: ${total:.2f}")
```
</details>

---

**Exercise 4 (Medium):** Given `in_stock_raw = "False"` (a string), write code that correctly converts it to a real boolean `False`, and prove `bool(in_stock_raw)` alone gives the wrong answer by printing both.

<details>
<summary>Hint</summary>
Remember: any non-empty string is truthy. Compare the lowercased string against `"true"` instead.
</details>

<details>
<summary>Solution</summary>

```python
in_stock_raw = "False"

wrong = bool(in_stock_raw)                    # True (wrong!)
correct = in_stock_raw.lower() == "true"       # False (correct)

print("Naive bool():", wrong)
print("Correct check:", correct)
```
</details>

---

**Exercise 5 (Hard):** You receive one row of raw data as strings:
```python
row = {"id": "7", "price": "9.5", "discount_pct": "10", "active": "yes"}
```
Write code that produces a cleaned dictionary with correctly typed values: `id` as int, `price` as float, `discount_pct` as float, and `active` as a bool (treat `"yes"` as `True`, anything else as `False`). Then compute and print the final discounted price.

<details>
<summary>Hint</summary>
Build a new dictionary, casting each field individually. Discounted price = `price * (1 - discount_pct / 100)`.
</details>

<details>
<summary>Solution</summary>

```python
row = {"id": "7", "price": "9.5", "discount_pct": "10", "active": "yes"}

cleaned = {
    "id": int(row["id"]),
    "price": float(row["price"]),
    "discount_pct": float(row["discount_pct"]),
    "active": row["active"].lower() == "yes",
}

discounted_price = cleaned["price"] * (1 - cleaned["discount_pct"] / 100)
print(cleaned)
print(f"Discounted price: ${discounted_price:.2f}")
```
</details>

---

**Next lesson:** [1.2 — Operators & Expressions](lesson-1.2-operators-and-expressions.md)
