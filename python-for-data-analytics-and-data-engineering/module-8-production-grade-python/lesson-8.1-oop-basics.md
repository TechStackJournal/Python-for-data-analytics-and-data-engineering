# Lesson 8.1 — From Functions to Classes: Intro to OOP

## CONCEPT

A **class** is a blueprint for creating objects that bundle together **data** (attributes) and **behavior** (methods) that belong together. Where Module 1's functions took data in and returned data out, a class lets you *group* related data and the functions that operate on it into a single, self-contained unit.

```python
class OrderProcessor:
    def __init__(self, tax_rate):
        self.tax_rate = tax_rate           # data the object carries around

    def calculate_total(self, price, quantity):    # behavior — a method
        subtotal = price * quantity
        return round(subtotal * (1 + self.tax_rate), 2)

processor = OrderProcessor(tax_rate=0.08)   # create an "instance" of the class
processor.calculate_total(24.99, 3)           # 80.97
```

`__init__` is a special method (a **constructor**) that runs automatically when you create a new instance, setting up its initial data. `self` refers to "this specific instance" — every method needs it as its first parameter.

## WHY IT MATTERS

Not everything needs to be a class — plenty of good pipeline code is just well-organized functions (Module 1, Lesson 1.6), and you shouldn't reach for a class just because it feels more "professional." But once a piece of logic has **state that needs to persist across multiple calls** (a database connection, a running configuration, accumulated statistics) or **several related operations that share the same data**, a class organizes that far more cleanly than a pile of functions passing the same arguments around repeatedly. Recognizing *when* a class is the right tool — not just knowing the syntax — is the actual skill this lesson builds.

## SYNTAX REFERENCE

```python
class ClassName:
    def __init__(self, param1, param2):
        self.attribute1 = param1        # instance attributes — unique per object
        self.attribute2 = param2

    def some_method(self):
        return self.attribute1 * 2       # methods access attributes via self

# Creating an instance
obj = ClassName(10, 20)
obj.attribute1              # 10 — accessing an attribute
obj.some_method()             # calling a method

# A class-level attribute (shared across ALL instances, not just one)
class Config:
    DEFAULT_TIMEOUT = 30     # same for every instance unless overridden

    def __init__(self, timeout=None):
        self.timeout = timeout or Config.DEFAULT_TIMEOUT

# String representation — controls what print(obj) shows
class Product:
    def __init__(self, name, price):
        self.name = name
        self.price = price

    def __repr__(self):
        return f"Product(name={self.name!r}, price={self.price})"

print(Product("Mouse", 24.99))   # Product(name='Mouse', price=24.99)

# Inheritance — a class building on another class (used sparingly in this course)
class BaseCleaner:
    def clean(self, value):
        raise NotImplementedError    # subclasses must implement this

class PriceCleaner(BaseCleaner):
    def clean(self, value):
        return float(str(value).replace("$", "").strip())
```

## WORKED EXAMPLE

Refactoring a set of related functions (the style from Module 1, Lesson 1.6) into a class, once they start sharing state — here, a running count of processed/failed rows that several functions all need to update:

```python
# --- Before: functions passing shared state around manually ---
def clean_order(raw_order, stats):
    try:
        cleaned = {
            "id": int(raw_order["id"]),
            "total": float(raw_order["total"]),
        }
        stats["processed"] += 1
        return cleaned
    except (ValueError, KeyError):
        stats["failed"] += 1
        return None

stats = {"processed": 0, "failed": 0}
raw_orders = [{"id": "1", "total": "45.99"}, {"id": "2", "total": "bad"}]
for raw in raw_orders:
    clean_order(raw, stats)   # stats has to be threaded through every call — easy to forget

print(stats)   # {'processed': 1, 'failed': 1}


# --- After: a class that owns its own state ---
class OrderCleaner:
    def __init__(self):
        self.processed = 0
        self.failed = 0

    def clean(self, raw_order):
        try:
            cleaned = {
                "id": int(raw_order["id"]),
                "total": float(raw_order["total"]),
            }
            self.processed += 1
            return cleaned
        except (ValueError, KeyError):
            self.failed += 1
            return None

    def summary(self):
        return f"Processed: {self.processed}, Failed: {self.failed}"


cleaner = OrderCleaner()
raw_orders = [{"id": "1", "total": "45.99"}, {"id": "2", "total": "bad"}, {"id": "3", "total": "89.00"}]

cleaned_orders = []
for raw in raw_orders:
    result = cleaner.clean(raw)
    if result:
        cleaned_orders.append(result)

print(cleaned_orders)          # [{'id': 1, 'total': 45.99}, {'id': 3, 'total': 89.0}]
print(cleaner.summary())         # Processed: 2, Failed: 1

# Notice: the state (processed/failed counts) lives INSIDE the object, not
# passed around as a separate argument to every call — this is the core
# benefit a class provides once state needs to persist across calls
```

## COMMON MISTAKES

**1. Forgetting `self` as the first parameter of every method**
```python
class Cleaner:
    def clean(value):        # missing self!
        return value.strip()

Cleaner().clean("abc")   # TypeError — clean() takes 1 positional argument but 2 were given
```
Fix: every method (except certain special cases not covered in this course) needs `self` as its first parameter — Python passes the instance automatically when you call `obj.method()`.

**2. Confusing class-level attributes with instance-level attributes**
```python
class Counter:
    count = 0          # CLASS attribute — shared across every instance!

    def increment(self):
        self.count += 1    # this actually creates a new INSTANCE attribute, shadowing the class one

a = Counter()
b = Counter()
a.increment()
print(a.count, b.count)   # 1 0 — they DON'T share state the way you might expect
```
Fix: for state that should be unique per instance (almost always what you want), set it inside `__init__` with `self.attribute = value`, not as a bare class-level variable.

**3. Reaching for a class when a plain function (or a dictionary) would be simpler**
```python
class Adder:
    def add(self, a, b):
        return a + b
```
If there's no state to carry between calls, this is unnecessary — a plain function `def add(a, b): return a + b` does the same job with less ceremony. Fix: default to functions; reach for a class specifically when you have state that needs to persist, or several operations that clearly belong together around that shared state.

**4. Mutating a class-level mutable default (the same trap as Lesson 1.6's mutable default arguments)**
```python
class Cart:
    items = []          # DANGEROUS — shared across every instance, same root cause as before

    def add(self, item):
        self.items.append(item)   # modifies the SHARED list!
```
Fix: initialize mutable attributes inside `__init__` (`self.items = []`), never as a bare class-level attribute.

**5. Not implementing `__repr__`, and getting an unhelpful default when printing/debugging**
```python
print(Product("Mouse", 24.99))   # <__main__.Product object at 0x7f8b1c0a4d90> — not useful!
```
Fix: adding a simple `__repr__` method (as in the Syntax Reference) makes debugging and logging dramatically more pleasant — this is a small habit worth building early.

**Data Engineer Tip:** A very common real pattern: a class representing "a connection to a specific data source" (wrapping a database connection, retry configuration, and a set of related query methods together) — this is genuinely a case where state (the open connection) persists across multiple method calls, making a class the right tool rather than passing a connection object into a dozen separate functions. Module 9's capstone introduces exactly this kind of class.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create a class `Rectangle` with an `__init__` that takes `width` and `height`, and a method `area()` that returns `width * height`. Create an instance with `width=4, height=5` and print its area.

<details>
<summary>Hint</summary>
`self.width = width` inside `__init__`, then `return self.width * self.height` inside `area()`.
</details>

<details>
<summary>Solution</summary>

```python
class Rectangle:
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

rect = Rectangle(4, 5)
print(rect.area())   # 20
```
</details>

---

**Exercise 2 (Easy):** Add a `__repr__` method to the `Rectangle` class from Exercise 1 that returns a string like `"Rectangle(4x5)"`. Print an instance directly to confirm it uses your `__repr__`.

<details>
<summary>Hint</summary>
`return f"Rectangle({self.width}x{self.height})"`
</details>

<details>
<summary>Solution</summary>

```python
class Rectangle:
    def __init__(self, width, height):
        self.width = width
        self.height = height

    def area(self):
        return self.width * self.height

    def __repr__(self):
        return f"Rectangle({self.width}x{self.height})"

rect = Rectangle(4, 5)
print(rect)   # Rectangle(4x5)
```
</details>

---

**Exercise 3 (Medium):** Create a class `RunningTotal` with an `__init__` that sets `self.total = 0`, a method `add(amount)` that adds to the total, and a method `reset()` that sets it back to 0. Create an instance, add `10`, `25`, and `5`, print the total, then reset and confirm it's back to 0.

<details>
<summary>Hint</summary>
`self.total += amount` inside `add()`; `self.total = 0` inside `reset()`.
</details>

<details>
<summary>Solution</summary>

```python
class RunningTotal:
    def __init__(self):
        self.total = 0

    def add(self, amount):
        self.total += amount

    def reset(self):
        self.total = 0

rt = RunningTotal()
rt.add(10)
rt.add(25)
rt.add(5)
print(rt.total)   # 40
rt.reset()
print(rt.total)    # 0
```
</details>

---

**Exercise 4 (Medium):** Create a class `PriceValidator` with a class-level attribute `MIN_PRICE = 0` and `MAX_PRICE = 10000`, and a method `is_valid(price)` that returns `True` if the price is within that range (inclusive), `False` otherwise. Test it with a few values.

<details>
<summary>Hint</summary>
Class-level attributes are safe here since they're never modified — only read. Access them via `self.MIN_PRICE` or `PriceValidator.MIN_PRICE` inside the method.
</details>

<details>
<summary>Solution</summary>

```python
class PriceValidator:
    MIN_PRICE = 0
    MAX_PRICE = 10000

    def is_valid(self, price):
        return self.MIN_PRICE <= price <= self.MAX_PRICE

validator = PriceValidator()
print(validator.is_valid(50))       # True
print(validator.is_valid(-5))        # False
print(validator.is_valid(15000))      # False
```
</details>

---

**Exercise 5 (Hard):** Refactor this function-based order-cleaning code into a class `OrderBatchCleaner` that tracks `processed`, `failed`, and a list of `errors` (each error as a dict with the raw input and the error message) as instance attributes, with a `clean_batch(raw_orders)` method that processes a whole list at once and returns the list of successfully cleaned orders, and a `report()` method that returns a formatted summary string.

```python
raw_orders = [
    {"id": "1", "total": "45.99"},
    {"id": "2", "total": "not_a_number"},
    {"id": "3"},
    {"id": "4", "total": "89.00"},
]
```

<details>
<summary>Hint</summary>
Give `__init__` three instance attributes: `self.processed = 0`, `self.failed = 0`, `self.errors = []`. `clean_batch` loops through the input, using a `try/except` per row (the pattern from Lesson 1.8), appending to `self.errors` on failure and building a return list of successes.
</details>

<details>
<summary>Solution</summary>

```python
class OrderBatchCleaner:
    def __init__(self):
        self.processed = 0
        self.failed = 0
        self.errors = []

    def clean_batch(self, raw_orders):
        cleaned = []
        for raw in raw_orders:
            try:
                cleaned_order = {
                    "id": int(raw["id"]),
                    "total": float(raw["total"]),
                }
                cleaned.append(cleaned_order)
                self.processed += 1
            except (ValueError, KeyError) as e:
                self.failed += 1
                self.errors.append({"raw": raw, "error": str(e)})
        return cleaned

    def report(self):
        return (
            f"Processed: {self.processed}, Failed: {self.failed}\n"
            f"Errors: {self.errors}"
        )


raw_orders = [
    {"id": "1", "total": "45.99"},
    {"id": "2", "total": "not_a_number"},
    {"id": "3"},
    {"id": "4", "total": "89.00"},
]

cleaner = OrderBatchCleaner()
cleaned = cleaner.clean_batch(raw_orders)

print(cleaned)
# [{'id': 1, 'total': 45.99}, {'id': 4, 'total': 89.0}]
print(cleaner.report())
# Processed: 2, Failed: 2
# Errors: [{'raw': {'id': '2', 'total': 'not_a_number'}, 'error': "could not convert string to float: 'not_a_number'"}, {'raw': {'id': '3'}, 'error': "'total'"}]
```
</details>

---

**Next lesson:** [8.2 — Structuring a Project: Modules & Packages](lesson-8.2-project-structure.md)
