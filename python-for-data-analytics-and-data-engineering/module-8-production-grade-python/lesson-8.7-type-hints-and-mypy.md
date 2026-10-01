# Lesson 8.7 — Type Hints & Static Checking with `mypy`

## CONCEPT

A **type hint** annotates a function's parameters and return value with the types they're expected to be — documentation that a tool can actually check, rather than just a comment a human might skim past. **`mypy`** is a **static type checker**: it reads your type hints and scans your code for type mismatches *without running it*, catching a category of bug before your code ever executes.

```python
def clean_price(raw_price: str) -> float:
    return float(raw_price.replace("$", ""))
```

```bash
mypy transform.py
# transform.py:5: error: Argument 1 to "clean_price" has incompatible type "int"; expected "str"
```

Python itself never enforces type hints at runtime (they're ignored during actual execution) — `mypy` is a separate tool that reads them and checks your code for consistency *before* you run it.

## WHY IT MATTERS

As a codebase grows past a handful of functions, it becomes easy to accidentally pass the wrong type of value into a function — a string where a function expects a list, a `None` where it expects a number — and have that mistake surface confusingly far from where it actually happened. Type hints make a function's contract explicit and self-documenting (no need to read the implementation to know what it expects), and `mypy` turns that documentation into an automatic, repeatable check — genuinely catching real bugs before a single test even runs, for a category of error (`str` vs `int`, `None` where a value was assumed) that's extremely common in exactly the kind of messy real-world data cleaning this course has covered throughout.

## SYNTAX REFERENCE

```python
# Basic parameter and return type hints
def add(a: int, b: int) -> int:
    return a + b

def greet(name: str) -> str:
    return f"Hello, {name}!"

# No return value (a function that only has side effects, like printing)
def log_message(message: str) -> None:
    print(message)

# Built-in collection types (modern syntax, Python 3.9+)
def process_prices(prices: list[float]) -> float:
    return sum(prices)

def get_config() -> dict[str, str]:
    return {"host": "localhost", "port": "5432"}

def unique_categories(categories: set[str]) -> int:
    return len(categories)

# Optional — a value that might be None (very common for "might be missing" data)
from typing import Optional

def find_customer(customer_id: int) -> Optional[dict]:
    # returns a dict if found, None if not — Optional[dict] communicates this explicitly
    ...

# Union — a value that could be one of SEVERAL specific types
from typing import Union

def parse_id(raw_id: Union[str, int]) -> int:
    return int(raw_id)

# A more modern shorthand for Optional/Union (Python 3.10+)
def find_customer(customer_id: int) -> dict | None:      # same as Optional[dict]
    ...

def parse_id(raw_id: str | int) -> int:                     # same as Union[str, int]
    ...

# Type hints on variables (less common, but sometimes useful for clarity)
customer_count: int = 0
order_totals: list[float] = []

# Type-hinting a class's attributes and methods
class OrderCleaner:
    def __init__(self, tax_rate: float) -> None:
        self.tax_rate: float = tax_rate

    def clean(self, raw_order: dict) -> dict | None:
        ...

# Running mypy
# mypy my_script.py             — check a single file
# mypy src/                       — check an entire package/folder
# mypy --strict src/                — enable much stricter checking (good for new projects)
```

## WORKED EXAMPLE

Adding type hints to the `transform.py` functions from earlier lessons, then deliberately introducing a bug and seeing `mypy` catch it:

```python
# ecommerce_pipeline/transform.py — WITH type hints
from typing import Optional


def clean_price(raw_price: str) -> float:
    cleaned = raw_price.replace("$", "").replace(",", "").strip()
    return float(cleaned)


def classify_order_size(item_count: int) -> str:
    if item_count >= 10:
        return "bulk"
    elif item_count >= 3:
        return "standard"
    else:
        return "small"


def clean_order(raw_order: dict) -> Optional[dict]:
    """Returns a cleaned order dict, or None if the input is invalid."""
    try:
        return {
            "id": int(raw_order["id"]),
            "total": clean_price(str(raw_order["total"])),
        }
    except (ValueError, KeyError):
        return None


def process_batch(raw_orders: list[dict]) -> list[dict]:
    cleaned: list[dict] = []
    for raw in raw_orders:
        result = clean_order(raw)
        if result is not None:
            cleaned.append(result)
    return cleaned


# --- A deliberate bug, for demonstration ---
def calculate_total_revenue(orders: list[dict]) -> float:
    return sum(order["total"] for order in orders)

# Calling it correctly:
orders = [{"id": 1, "total": 45.99}, {"id": 2, "total": 89.00}]
print(calculate_total_revenue(orders))   # 134.99 — works fine

# Calling it INCORRECTLY — passing a single dict instead of a list of dicts:
# calculate_total_revenue({"id": 1, "total": 45.99})
#
# Running `mypy transform.py` on this line would report:
# transform.py:45: error: Argument 1 to "calculate_total_revenue" has
#   incompatible type "dict[str, object]"; expected "list[dict[Any, Any]]"
#
# mypy caught this WITHOUT ever running the code — a plain Python interpreter
# would only have discovered this bug once that exact line actually executed,
# potentially much later, and with a less specific error message
# (TypeError: 'int' object is not iterable, from deep inside sum())
```

## COMMON MISTAKES

**1. Adding type hints but never actually running `mypy` to check them**
```python
def clean_price(raw_price: str) -> float:
    ...
# type hints alone, without running mypy, are just documentation — Python itself
# never enforces them at runtime, and a mismatch will silently pass unless checked
```
Fix: type hints provide real value specifically when paired with running `mypy` regularly (ideally automated — Lesson 8.9 covers pre-commit hooks that can run it automatically) — hints alone, never checked, are better than nothing but leave most of their value on the table.

**2. Over-annotating every single local variable, adding noise without real benefit**
```python
x: int = 5
name: str = "Maria"
total: float = 0.0
```
For simple local variables where the type is already obvious from the assigned value, this is usually unnecessary noise. Fix: type hints matter most on function *signatures* (parameters and return types) — that's the actual "contract" a function promises to its callers, and where `mypy` catches real, meaningful mismatches; hint local variables only when the type genuinely isn't obvious from context.

**3. Using `Optional[X]` and then forgetting to actually handle the `None` case**
```python
def find_customer(customer_id: int) -> Optional[dict]:
    ...

customer = find_customer(42)
print(customer["name"])   # mypy WILL flag this — customer might be None, and you didn't check!
```
Fix: `mypy` specifically catches this class of bug — accessing an attribute on a value that's typed as possibly `None` without first checking — which is exactly the kind of silent `AttributeError`/`TypeError` that's easy to miss by eye but is precisely what static type checking is designed to catch.

**4. Assuming type hints make Python fully statically typed, like Java or C#**
```python
def add(a: int, b: int) -> int:
    return a + b

add("2", "3")   # Python ITSELF runs this FINE at runtime (string concatenation), ignoring the hints entirely!
```
Fix: type hints are purely optional, advisory annotations that Python's interpreter ignores completely — only `mypy` (or a similar static checker) actually enforces them, and only when you run it; this is a meaningful difference from genuinely statically-typed languages, worth understanding clearly.

**5. Fighting with overly complex generic types too early**
Nested generics like `dict[str, list[tuple[int, float]]]` can get genuinely hard to read and maintain. Fix: for this course's purposes, stick to straightforward hints (`str`, `int`, `float`, `bool`, `list[X]`, `dict[str, X]`, `Optional[X]`) — reach for more complex typing constructs only once you have a concrete need for them, rather than pre-emptively over-engineering type signatures.

**Data Engineer Tip:** Type hints and `mypy` checking are increasingly a baseline expectation on professional data engineering teams, precisely because pipeline code frequently passes data between many functions and modules (Lesson 8.2) — a type mismatch introduced in one file can otherwise surface as a confusing runtime error several function calls away, in a completely different file, making the actual root cause hard to trace. Running `mypy` as part of your development workflow (and, as Lesson 8.9 covers, automatically before every commit) catches this entire category of bug at the moment it's introduced, not days later in production.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Add type hints to `def add(a, b): return a + b`, specifying both parameters and the return value as `int`. Run `mypy` on the file to confirm it reports no errors.

<details>
<summary>Hint</summary>
`def add(a: int, b: int) -> int:`
</details>

<details>
<summary>Solution</summary>

```python
# add.py
def add(a: int, b: int) -> int:
    return a + b
```
```bash
mypy add.py
# Success: no issues found in 1 source file
```
</details>

---

**Exercise 2 (Easy):** Add type hints to `def greet(name): return f"Hello, {name}!"`, and then deliberately call it with an integer (`greet(42)`) to see what `mypy` reports.

<details>
<summary>Hint</summary>
`def greet(name: str) -> str:`, then add a call `greet(42)` below it.
</details>

<details>
<summary>Solution</summary>

```python
# greet.py
def greet(name: str) -> str:
    return f"Hello, {name}!"

greet(42)   # deliberately wrong type
```
```bash
mypy greet.py
# greet.py:4: error: Argument 1 to "greet" has incompatible type "int"; expected "str"
```
</details>

---

**Exercise 3 (Medium):** Add type hints to a function `def get_prices(): return [10.0, 20.0, 30.0]`, correctly specifying the return type as a list of floats, and to a function `def total(prices): return sum(prices)`, specifying its parameter as a list of floats and its return as a float.

<details>
<summary>Hint</summary>
`def get_prices() -> list[float]:` and `def total(prices: list[float]) -> float:`
</details>

<details>
<summary>Solution</summary>

```python
# prices.py
def get_prices() -> list[float]:
    return [10.0, 20.0, 30.0]

def total(prices: list[float]) -> float:
    return sum(prices)

result = total(get_prices())
print(result)
```
```bash
mypy prices.py
# Success: no issues found in 1 source file
```
</details>

---

**Exercise 4 (Medium):** Write a function `find_product(product_id: int, catalog: dict[int, str]) -> Optional[str]` that returns the product name if found in `catalog`, or `None` otherwise. Then write code that calls it and — WITHOUT first checking for `None` — tries to call `.upper()` on the result. Run `mypy` and observe what it reports about this specific mistake.

<details>
<summary>Hint</summary>
`from typing import Optional` at the top. The `.upper()` call without a `None` check should trigger a mypy error about a possibly-`None` value.
</details>

<details>
<summary>Solution</summary>

```python
# products.py
from typing import Optional

def find_product(product_id: int, catalog: dict[int, str]) -> Optional[str]:
    return catalog.get(product_id)

catalog = {1: "Mouse", 2: "Keyboard"}
result = find_product(3, catalog)
print(result.upper())   # mypy flags this — result might be None!
```
```bash
mypy products.py
# products.py:8: error: Item "None" of "Optional[str]" has no attribute "upper"
```
The fix (not required for this exercise, but worth knowing) would be to check `if result is not None:` before calling `.upper()`.
</details>

---

**Exercise 5 (Hard):** Take the `process_batch` function from the Worked Example (with type hints already in place: `def process_batch(raw_orders: list[dict]) -> list[dict]:`) and write THREE deliberately incorrect calls to it that `mypy` should catch as type errors: (1) passing a single dict instead of a list, (2) passing a list of strings instead of a list of dicts, (3) passing the function's own return value into something expecting a different type, e.g. treating the returned `list[dict]` as if it were a single `dict` afterward. Run `mypy` and confirm it reports all three issues, then write a short comment explaining what each error message is telling you.

<details>
<summary>Hint</summary>
For case 3, try something like `result = process_batch(orders); result["total"]` — since `result` is a `list[dict]`, indexing it with a string key is a type error, not just a runtime one.
</details>

<details>
<summary>Solution</summary>

```python
# batch_errors.py
def process_batch(raw_orders: list[dict]) -> list[dict]:
    return [order for order in raw_orders if "id" in order]

# Error 1: passing a single dict, not a list of dicts
process_batch({"id": 1, "total": 45.99})

# Error 2: passing a list of strings, not a list of dicts
process_batch(["order_1", "order_2"])

# Error 3: treating the LIST return value as if it were a single dict
result = process_batch([{"id": 1}, {"id": 2}])
print(result["total"])   # result is a list[dict], not a dict — this is a type error
```
```bash
mypy batch_errors.py
# batch_errors.py:5: error: Argument 1 to "process_batch" has incompatible
#   type "dict[str, float]"; expected "list[dict[Any, Any]]"
#   --> Error 1: a single dict was passed where a LIST of dicts was expected
#
# batch_errors.py:8: error: List item 0 has incompatible type "str";
#   expected "dict[Any, Any]"
#   --> Error 2: the list itself was fine, but its CONTENTS (strings, not
#       dicts) don't match what process_batch expects for each element
#
# batch_errors.py:12: error: Invalid index type "str" for "list[dict[Any, Any]]";
#   expected type "SupportsIndex"
#   --> Error 3: result is a LIST (which needs an integer index like result[0]),
#       but the code tried to index it with a string key ("total"), which is
#       only valid for a dict — a clear sign the return value was misunderstood
```
</details>

---

**Previous lesson:** [8.6 — Writing Tests with pytest](lesson-8.6-pytest.md)
**Next lesson:** [8.8 — Configuration Management](lesson-8.8-configuration-management.md)
