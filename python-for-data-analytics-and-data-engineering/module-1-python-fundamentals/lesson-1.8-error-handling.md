# Lesson 1.8 — Error Handling Basics

## CONCEPT

An **exception** is an error that occurs while a program is running, interrupting normal execution. If left unhandled, it crashes the program and prints a **traceback**. Python lets you anticipate and gracefully handle exceptions using `try`/`except`:

```python
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Can't divide by zero!")
```

Instead of crashing, the program catches the specific error and runs alternative code. You can also use `else` (runs if no exception occurred) and `finally` (runs no matter what — errors or not).

## WHY IT MATTERS

Real data is unpredictable: a field that's supposed to be a number might be blank, a file you expect to exist might be missing, an API might time out. In a one-off script, a crash just means you rerun it. In a scheduled pipeline running unattended at 2am, an unhandled exception on row 4,000 of 10,000 means the whole job fails and nobody finds out until morning — or worse, it fails silently. Learning to anticipate specific failure modes and handle them deliberately is the difference between a script and a pipeline.

## SYNTAX REFERENCE

```python
# Basic try/except
try:
    risky_code()
except SomeErrorType:
    handle_it()

# Catching multiple specific exception types
try:
    risky_code()
except (ValueError, TypeError):
    handle_it()

# Catching the exception object itself, to inspect it
try:
    risky_code()
except ValueError as e:
    print(f"Something went wrong: {e}")

# Catching anything (use sparingly — see Common Mistakes)
try:
    risky_code()
except Exception as e:
    print(f"Unexpected error: {e}")

# else — runs only if no exception occurred
try:
    value = int("42")
except ValueError:
    print("Conversion failed")
else:
    print(f"Success: {value}")

# finally — always runs, error or not (good for cleanup)
try:
    file = open("data.csv")
    process(file)
except FileNotFoundError:
    print("File missing")
finally:
    print("Done attempting to process the file")

# Raising your own exceptions
def set_discount(pct):
    if pct < 0 or pct > 100:
        raise ValueError(f"Discount must be 0-100, got {pct}")
    return pct

# Common built-in exception types you'll encounter constantly:
# ValueError     - right type, invalid value (int("abc"))
# TypeError      - wrong type entirely ("2" + 2)
# KeyError       - missing dictionary key
# IndexError     - list index out of range
# ZeroDivisionError
# FileNotFoundError
# AttributeError - calling a method that doesn't exist on that object
```

## WORKED EXAMPLE

Processing a batch of raw order records where some fields are malformed — handling each failure gracefully instead of letting one bad row crash the whole batch:

```python
raw_orders = [
    {"id": "101", "total": "45.99"},
    {"id": "102", "total": "not_a_number"},   # bad data
    {"id": "103"},                             # missing 'total' key
    {"id": "104", "total": "89.50"},
]

def parse_order(raw):
    """Parse one raw order dict, returning a clean dict or raising a clear error."""
    order_id = int(raw["id"])          # could raise KeyError or ValueError
    total = float(raw["total"])        # could raise KeyError or ValueError
    return {"id": order_id, "total": total}


clean_orders = []
failed_orders = []

for raw in raw_orders:
    try:
        clean = parse_order(raw)
    except KeyError as e:
        failed_orders.append({"raw": raw, "reason": f"Missing field: {e}"})
        continue
    except ValueError as e:
        failed_orders.append({"raw": raw, "reason": f"Invalid value: {e}"})
        continue

    clean_orders.append(clean)

print(f"Successfully parsed: {len(clean_orders)}")
print(f"Failed to parse: {len(failed_orders)}")
for failure in failed_orders:
    print(f"  {failure['raw']} -> {failure['reason']}")

# Output:
# Successfully parsed: 2
# Failed to parse: 2
#   {'id': '102', 'total': 'not_a_number'} -> Invalid value: could not convert string to float: 'not_a_number'
#   {'id': '103'} -> Missing field: 'total'
```

Notice the batch **kept running** despite two bad rows — each failure was caught, logged, and the loop continued, rather than the whole thing crashing at row 2.

## COMMON MISTAKES

**1. Catching `Exception` (or worse, using a bare `except:`) too broadly**
```python
try:
    result = risky_calculation()
except:                     # catches EVERYTHING, including typos and bugs!
    print("Something went wrong")
```
This hides real bugs (like a misspelled variable name raising `NameError`) behind a generic message, making debugging much harder. Fix: catch the *specific* exception types you actually expect and know how to handle.

**2. Using `try/except` to silently swallow errors without logging anything**
```python
try:
    process_row(row)
except Exception:
    pass    # the error vanishes with no trace — very hard to debug later
```
Fix: at minimum, log or print what went wrong, including the specific row/input that caused it — you'll thank yourself later.

**3. Putting too much code inside one `try` block**
```python
try:
    data = load_file(path)
    cleaned = clean_data(data)
    result = save_to_db(cleaned)
except Exception as e:
    print("Something failed")   # but WHERE? Which of the 3 steps broke?
```
Fix: keep `try` blocks focused on the specific operation that can fail, so you know exactly what broke.

**4. Using exceptions for normal control flow instead of a simple `if` check**
```python
try:
    value = my_dict["key"]
except KeyError:
    value = "default"
```
This works, but for a simple lookup, `.get()` is clearer and more idiomatic:
```python
value = my_dict.get("key", "default")
```
Save `try/except` for genuinely unpredictable failures, not routine checks that a simple conditional (or a method like `.get()`) already handles cleanly.

**Data Engineer Tip:** In a production pipeline, the goal usually isn't "never fail" — it's "fail loudly and specifically, on exactly the bad row, without taking down the whole job." The pattern in the worked example — catch the error, record *which* row failed and *why*, and continue — is the foundation of resilient batch processing, and it's exactly what you'll formalize with proper logging (instead of `print`) in Module 8.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Write a `try/except` block that attempts `int("abc")` and prints `"Conversion failed"` if it raises a `ValueError`.

<details>
<summary>Hint</summary>
Wrap the conversion in `try`, catch `ValueError`.
</details>

<details>
<summary>Solution</summary>

```python
try:
    value = int("abc")
except ValueError:
    print("Conversion failed")
```
</details>

---

**Exercise 2 (Easy):** Write a function `safe_divide(a, b)` that returns the result of `a / b`, but returns `None` instead of crashing if `b` is zero.

<details>
<summary>Hint</summary>
Catch `ZeroDivisionError`.
</details>

<details>
<summary>Solution</summary>

```python
def safe_divide(a, b):
    try:
        return a / b
    except ZeroDivisionError:
        return None

print(safe_divide(10, 2))   # 5.0
print(safe_divide(10, 0))   # None
```
</details>

---

**Exercise 3 (Medium):** Given `order = {"id": 1, "total": 59.99}` (missing a `"status"` key), write code that tries to access `order["status"]`, catches the `KeyError`, and prints `"Status not set — defaulting to 'pending'"`, then sets a local variable `status = "pending"`.

<details>
<summary>Hint</summary>
Catch `KeyError` specifically, and set the default inside the `except` block.
</details>

<details>
<summary>Solution</summary>

```python
order = {"id": 1, "total": 59.99}

try:
    status = order["status"]
except KeyError:
    print("Status not set — defaulting to 'pending'")
    status = "pending"

print(status)   # "pending"
```
</details>

---

**Exercise 4 (Medium):** Write a function `parse_price(raw)` that takes a raw value (which might be a valid numeric string, an invalid string, or `None`) and returns a float, or `0.0` if parsing fails for any reason. Test it with `"24.99"`, `"invalid"`, and `None`.

<details>
<summary>Hint</summary>
`float(None)` raises `TypeError`, `float("invalid")` raises `ValueError` — catch both.
</details>

<details>
<summary>Solution</summary>

```python
def parse_price(raw):
    try:
        return float(raw)
    except (ValueError, TypeError):
        return 0.0

print(parse_price("24.99"))   # 24.99
print(parse_price("invalid")) # 0.0
print(parse_price(None))      # 0.0
```
</details>

---

**Exercise 5 (Hard):** You're given a batch of raw order dictionaries, some malformed:
```python
raw_orders = [
    {"id": "1", "price": "10.00", "qty": "2"},
    {"id": "2", "price": "abc", "qty": "1"},
    {"id": "3", "price": "5.00"},                # missing qty
    {"id": "4", "price": "20.00", "qty": "0"},
]
```
Write a function `process_batch(orders)` that loops through them, and for each one:
- Tries to compute `total = price * qty` (after converting both to the right types)
- Additionally, treats `qty == 0` as an intentional `ValueError` you raise yourself (with a custom message), since a zero-quantity order is invalid data
- Catches any failure (`ValueError`, `TypeError`, or `KeyError`), and appends a dict like `{"id": raw_id_or_unknown, "error": str(exception)}` to a `failures` list instead of crashing
- Appends successful results as `{"id": ..., "total": ...}` to a `results` list

Return both lists as a tuple `(results, failures)`, and print both after calling it on `raw_orders`.

<details>
<summary>Hint</summary>
Raise your own `ValueError` for the `qty == 0` case with `raise ValueError("...")` inside the `try` block — it'll be caught by the same `except` that handles the others. Use `raw.get("id", "unknown")` when building the failure record, in case `id` itself is missing.
</details>

<details>
<summary>Solution</summary>

```python
raw_orders = [
    {"id": "1", "price": "10.00", "qty": "2"},
    {"id": "2", "price": "abc", "qty": "1"},
    {"id": "3", "price": "5.00"},
    {"id": "4", "price": "20.00", "qty": "0"},
]

def process_batch(orders):
    results = []
    failures = []

    for raw in orders:
        try:
            price = float(raw["price"])
            qty = int(raw["qty"])

            if qty == 0:
                raise ValueError("Quantity cannot be zero")

            total = price * qty
            results.append({"id": raw["id"], "total": total})

        except (ValueError, TypeError, KeyError) as e:
            results_id = raw.get("id", "unknown")
            failures.append({"id": results_id, "error": str(e)})

    return results, failures


results, failures = process_batch(raw_orders)
print("Results:", results)
print("Failures:", failures)

# Results: [{'id': '1', 'total': 20.0}]
# Failures: [
#   {'id': '2', 'error': "could not convert string to float: 'abc'"},
#   {'id': '3', 'error': "'qty'"},
#   {'id': '4', 'error': 'Quantity cannot be zero'}
# ]
```
</details>

---

## Module 1 Complete

You've now covered variables and types, operators, strings, conditionals, loops, functions, scope, and error handling — the full core vocabulary of Python. Every concept in this module will reappear, in a more powerful form, throughout the rest of the course:

- Loops and conditionals → vectorized filtering in pandas (Module 4–5)
- Functions and scope → pipeline stage design (Module 8)
- Error handling → resilient batch ETL jobs (Module 9)

**Previous lesson:** [1.7 — Scope & Namespaces](lesson-1.7-scope.md)
**Next module:** [Module 2 — Core Data Structures & File Handling](../module-2-data-structures-and-files/README.md)
