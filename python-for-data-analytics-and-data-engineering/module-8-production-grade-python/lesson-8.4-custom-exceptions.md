# Lesson 8.4 — Robust Error Handling & Custom Exceptions

## CONCEPT

Lesson 1.8 covered catching built-in exceptions (`ValueError`, `KeyError`). A **custom exception** is your own exception class — usually a small subclass of Python's built-in `Exception` — that gives a specific, meaningful name to a specific failure mode in *your* domain, rather than relying on generic built-in exception types that don't communicate what actually went wrong.

```python
class InvalidOrderError(Exception):
    """Raised when an order record fails validation."""
    pass

def validate_order(order):
    if order["total"] < 0:
        raise InvalidOrderError(f"Order {order['id']} has a negative total: {order['total']}")
```

Catching code can then handle `InvalidOrderError` specifically, differently from, say, a `ConnectionError` from a failed database call — rather than every kind of failure looking identical as a generic `Exception`.

## WHY IT MATTERS

As a pipeline grows, "something went wrong" isn't specific enough to act on automatically. Should a failure retry (a transient network blip)? Skip this one record and continue (bad data in a single row)? Halt the entire pipeline (a fundamentally broken configuration)? Custom exceptions let your code — and a human reading the logs later — immediately understand *which kind* of problem occurred, because the exception's name and message say so directly, rather than every failure being an undifferentiated `Exception` that requires reading a stack trace to understand.

## SYNTAX REFERENCE

```python
# Defining a custom exception — usually just a subclass with a docstring
class DataQualityError(Exception):
    """Raised when incoming data fails a quality check."""
    pass

class ConnectionFailedError(Exception):
    """Raised when a connection to an external system cannot be established."""
    pass

# Building a small HIERARCHY of related exceptions (useful for a real pipeline)
class PipelineError(Exception):
    """Base class for all errors raised by this pipeline."""
    pass

class ExtractError(PipelineError):
    """Raised when data extraction fails."""
    pass

class TransformError(PipelineError):
    """Raised when a transformation step fails."""
    pass

class LoadError(PipelineError):
    """Raised when writing data to the destination fails."""
    pass

# Raising a custom exception, often WITH additional context attached
class OrderValidationError(Exception):
    def __init__(self, order_id, reason):
        self.order_id = order_id
        self.reason = reason
        super().__init__(f"Order {order_id} failed validation: {reason}")

raise OrderValidationError(order_id=1042, reason="negative total")

# Catching a custom exception specifically
try:
    validate_order(order)
except OrderValidationError as e:
    print(f"Skipping order {e.order_id}: {e.reason}")

# Catching a BASE exception catches its subclasses too (the hierarchy pattern)
try:
    run_pipeline_step()
except PipelineError as e:      # catches ExtractError, TransformError, OR LoadError
    log_and_alert(e)

# Re-raising an exception after doing something with it (preserves the original traceback)
try:
    risky_step()
except ValueError as e:
    log.error(f"Step failed: {e}")
    raise                          # re-raises the SAME exception, doesn't swallow it silently

# Chaining exceptions — wrapping a low-level error in a more meaningful one,
# while preserving the original cause for debugging
try:
    connect_to_database()
except OSError as original_error:
    raise ConnectionFailedError("Could not reach the database") from original_error
```

## WORKED EXAMPLE

Building a small exception hierarchy for a batch order-processing step, and using it to make failure handling specific and actionable:

```python
class PipelineError(Exception):
    """Base class for all pipeline-related errors."""
    pass


class OrderValidationError(PipelineError):
    """Raised when a single order fails business-rule validation."""
    def __init__(self, order_id, reason):
        self.order_id = order_id
        self.reason = reason
        super().__init__(f"Order {order_id} invalid: {reason}")


class DataSourceUnavailableError(PipelineError):
    """Raised when the upstream data source cannot be reached at all."""
    pass


def validate_order(order):
    if order.get("total") is None:
        raise OrderValidationError(order.get("id", "unknown"), "missing total")
    if order["total"] < 0:
        raise OrderValidationError(order["id"], f"negative total: {order['total']}")
    if order["total"] > 100_000:
        raise OrderValidationError(order["id"], f"implausibly large total: {order['total']}")
    return order


def process_batch(raw_orders):
    valid_orders = []
    skipped = []

    for raw in raw_orders:
        try:
            valid = validate_order(raw)
            valid_orders.append(valid)
        except OrderValidationError as e:
            # A validation error on ONE order shouldn't stop the whole batch —
            # log it and continue to the next order
            skipped.append({"order_id": e.order_id, "reason": e.reason})

    return valid_orders, skipped


raw_orders = [
    {"id": 1, "total": 45.99},
    {"id": 2, "total": -10.00},
    {"id": 3},
    {"id": 4, "total": 89.99},
    {"id": 5, "total": 250000},
]

valid, skipped = process_batch(raw_orders)
print(f"Valid: {len(valid)}, Skipped: {len(skipped)}")
for s in skipped:
    print(f"  Order {s['order_id']}: {s['reason']}")

# Valid: 2, Skipped: 3
#   Order 2: negative total: -10.0
#   Order unknown: missing total
#   Order 5: implausibly large total: 250000

# A DIFFERENT failure mode entirely — something that SHOULD stop everything,
# rather than being skipped and continued past
def fetch_orders_from_source():
    raise DataSourceUnavailableError("Order source system returned no response after 3 retries")

try:
    raw = fetch_orders_from_source()
except DataSourceUnavailableError as e:
    print(f"FATAL: cannot proceed without a data source — {e}")
    # In a real pipeline: alert on-call, halt the job, do NOT attempt to process an empty batch
except PipelineError as e:
    # A safety net for any OTHER PipelineError subclass we didn't specifically anticipate
    print(f"Unexpected pipeline error: {e}")
```

## COMMON MISTAKES

**1. Catching generic `Exception` when a specific custom exception would communicate more**
```python
try:
    validate_order(order)
except Exception as e:      # catches literally ANYTHING, including real bugs, not just validation failures
    print("Something went wrong")
```
Fix: catch the specific exception type(s) you're actually prepared to handle — `OrderValidationError`, not bare `Exception` — so a genuine bug (like a typo causing a `NameError`) doesn't get silently swallowed alongside expected validation failures.

**2. Defining a custom exception but never actually using its extra attributes**
```python
class OrderValidationError(Exception):
    def __init__(self, order_id, reason):
        self.order_id = order_id     # attached, but...
        self.reason = reason
        super().__init__(f"...")

except OrderValidationError as e:
    print(str(e))    # only using the generic message — never accessing e.order_id or e.reason directly!
```
Fix: if you've gone to the trouble of attaching structured data to a custom exception, actually use it in your handling code (as the worked example does) — that's the whole point of a custom exception over a plain `raise ValueError(f"...")`.

**3. Conflating "skip this one record and continue" errors with "halt everything" errors**
As the worked example shows, `OrderValidationError` (bad data in one row) and `DataSourceUnavailableError` (nothing to process at all) call for fundamentally different responses. Fix: design your exception hierarchy — and your handling code's structure — around *how each failure should actually be responded to*, not just what technically went wrong.

**4. Swallowing an exception silently with a bare `except: pass`**
```python
try:
    process_order(order)
except Exception:
    pass    # the failure vanishes with NO trace anywhere — the same Module 1 mistake, still costly here
```
Fix: at minimum, log the exception (Lesson 8.5 covers this properly) — a pipeline that fails silently is far more dangerous than one that fails loudly, since nobody notices until the downstream damage is already done.

**5. Losing the original traceback/cause when wrapping an exception**
```python
try:
    connect_to_database()
except OSError as e:
    raise ConnectionFailedError("Could not reach the database")   # the ORIGINAL OSError's details are lost!
```
Fix: use `raise NewException(...) from original_error` (shown in the Syntax Reference) — this preserves the full chain of causation in the traceback, which is often essential for actually diagnosing what happened, rather than just seeing your own wrapper message.

**Data Engineer Tip:** A well-designed exception hierarchy is what lets a pipeline's top-level error handling make smart, automatic decisions: "if this is a `DataQualityError`, log it and skip the row; if this is a `ConnectionFailedError`, retry with `tenacity` (Lesson 7.7); if this is anything else entirely, halt immediately and page whoever's on call." Without custom exceptions, that decision tree has no reliable way to distinguish between these fundamentally different situations — everything just looks like "an error happened," forcing either overly cautious halting on every failure, or overly permissive swallowing of genuine bugs.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Define a custom exception `NegativePriceError` (a simple subclass of `Exception`, no extra attributes needed). Write a function `validate_price(price)` that raises it if `price < 0`. Test it with a negative and a positive value.

<details>
<summary>Hint</summary>
`class NegativePriceError(Exception): pass`, then `raise NegativePriceError(...)` inside an `if`.
</details>

<details>
<summary>Solution</summary>

```python
class NegativePriceError(Exception):
    pass

def validate_price(price):
    if price < 0:
        raise NegativePriceError(f"Price cannot be negative: {price}")
    return price

print(validate_price(24.99))   # 24.99

try:
    validate_price(-5)
except NegativePriceError as e:
    print(f"Caught: {e}")
```
</details>

---

**Exercise 2 (Easy):** Define a custom exception `MissingFieldError` with an `__init__` that takes `field_name` and stores it as `self.field_name`, building a message like `"Missing required field: email"`. Raise and catch it, printing `e.field_name` specifically (not just the message).

<details>
<summary>Hint</summary>
`super().__init__(f"Missing required field: {field_name}")` inside `__init__`.
</details>

<details>
<summary>Solution</summary>

```python
class MissingFieldError(Exception):
    def __init__(self, field_name):
        self.field_name = field_name
        super().__init__(f"Missing required field: {field_name}")

try:
    raise MissingFieldError("email")
except MissingFieldError as e:
    print(f"Field that was missing: {e.field_name}")
```
</details>

---

**Exercise 3 (Medium):** Build a small exception hierarchy: a base `PipelineError`, and two subclasses `ExtractError` and `TransformError`. Write a function that raises `ExtractError` if given an empty list, and a `try/except` block that catches `PipelineError` (the base class) generically, printing which specific subclass was actually caught using `type(e).__name__`.

<details>
<summary>Hint</summary>
`type(e).__name__` gives the exact class name of the caught exception, even when you caught it via its base class.
</details>

<details>
<summary>Solution</summary>

```python
class PipelineError(Exception):
    pass

class ExtractError(PipelineError):
    pass

class TransformError(PipelineError):
    pass

def fetch_data(source):
    if not source:
        raise ExtractError("Source is empty — nothing to extract")
    return source

try:
    fetch_data([])
except PipelineError as e:
    print(f"Caught a {type(e).__name__}: {e}")
# Caught a ExtractError: Source is empty — nothing to extract
```
</details>

---

**Exercise 4 (Medium):** Write a function `process_batch(items)` that validates each item has a positive `"value"` key, using a custom `InvalidItemError(item_id, reason)` exception (with `item_id` and `reason` attributes). The function should NOT stop on the first bad item — collect valid items into one list and failures (as dicts with `item_id` and `reason`) into another, returning both. Test with a mixed batch.

<details>
<summary>Hint</summary>
This mirrors the worked example's `process_batch` pattern — wrap the per-item validation in a `try/except InvalidItemError` inside the loop.
</details>

<details>
<summary>Solution</summary>

```python
class InvalidItemError(Exception):
    def __init__(self, item_id, reason):
        self.item_id = item_id
        self.reason = reason
        super().__init__(f"Item {item_id} invalid: {reason}")

def validate_item(item):
    if item.get("value", -1) <= 0:
        raise InvalidItemError(item.get("id", "unknown"), "value must be positive")
    return item

def process_batch(items):
    valid = []
    failures = []
    for item in items:
        try:
            valid.append(validate_item(item))
        except InvalidItemError as e:
            failures.append({"item_id": e.item_id, "reason": e.reason})
    return valid, failures

items = [{"id": 1, "value": 10}, {"id": 2, "value": -3}, {"id": 3, "value": 0}, {"id": 4, "value": 5}]
valid, failures = process_batch(items)
print("Valid:", valid)
print("Failures:", failures)
```
</details>

---

**Exercise 5 (Hard):** Design and implement a small exception hierarchy for a hypothetical order pipeline with THREE distinct failure categories that should be handled differently: (1) `RetryableError` (a base for transient problems that are worth retrying — e.g. `TemporaryConnectionError`), (2) `SkippableError` (a base for problems with one specific record that shouldn't stop the batch — e.g. `InvalidRecordError`), and (3) `FatalError` (a base for problems that should halt the entire pipeline immediately — e.g. `ConfigurationError`). Write a function `handle_pipeline_error(e)` that takes any exception, checks which category it belongs to using `isinstance()`, and prints a DIFFERENT response message for each category (e.g. "Retrying...", "Skipping this record...", "HALTING PIPELINE..."), plus a fallback message for anything that isn't one of your three categories at all.

<details>
<summary>Hint</summary>
Define the three base classes and at least one concrete subclass of each. `handle_pipeline_error` should check `isinstance(e, RetryableError)`, then `isinstance(e, SkippableError)`, then `isinstance(e, FatalError)`, each as a separate `if`/`elif` — with a final `else` for anything uncategorized.
</details>

<details>
<summary>Solution</summary>

```python
class RetryableError(Exception):
    """Transient problems worth retrying."""
    pass

class TemporaryConnectionError(RetryableError):
    pass


class SkippableError(Exception):
    """A problem with one record — skip it, keep processing the batch."""
    pass

class InvalidRecordError(SkippableError):
    pass


class FatalError(Exception):
    """A problem serious enough to halt the entire pipeline."""
    pass

class ConfigurationError(FatalError):
    pass


def handle_pipeline_error(e):
    if isinstance(e, RetryableError):
        print(f"Retrying... ({type(e).__name__}: {e})")
    elif isinstance(e, SkippableError):
        print(f"Skipping this record... ({type(e).__name__}: {e})")
    elif isinstance(e, FatalError):
        print(f"HALTING PIPELINE. ({type(e).__name__}: {e})")
    else:
        print(f"Uncategorized error, treating as fatal by default: {type(e).__name__}: {e}")


# Test with one of each category, plus an uncategorized built-in exception
errors_to_test = [
    TemporaryConnectionError("connection reset"),
    InvalidRecordError("row 42 missing required field"),
    ConfigurationError("database URL not set"),
    ValueError("some unrelated built-in error"),
]

for err in errors_to_test:
    handle_pipeline_error(err)

# Retrying... (TemporaryConnectionError: connection reset)
# Skipping this record... (InvalidRecordError: row 42 missing required field)
# HALTING PIPELINE. (ConfigurationError: database URL not set)
# Uncategorized error, treating as fatal by default: ValueError: some unrelated built-in error
```
</details>

---

**Previous lesson:** [8.3 — Dependency Management with Poetry](lesson-8.3-poetry.md)
**Next lesson:** [8.5 — Logging Instead of print()](lesson-8.5-logging.md)
