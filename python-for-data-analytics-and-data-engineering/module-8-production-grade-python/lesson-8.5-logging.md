# Lesson 8.5 — Logging Instead of `print()`

## CONCEPT

Every debugging `print()` statement you've written throughout this course has worked fine for scripts you run and watch yourself. Python's built-in **`logging`** module is the production-grade replacement: it adds a **severity level** to every message (so you can filter what matters), a timestamp, and the ability to send output to multiple destinations at once (the terminal, a file, a monitoring system) — all without changing your actual logging calls.

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

logger.info("Pipeline started")
logger.warning("Order 1042 has a suspiciously high total")
logger.error("Failed to connect to the database")
```

## WHY IT MATTERS

A scheduled pipeline runs with nobody watching the terminal — `print()` output usually just vanishes. Logging is how you leave a trail: when something goes wrong at 3am, the log file (or monitoring dashboard) is what tells you — and whoever's on call — what actually happened, in what order, and how severe it was. The severity levels let you run in verbose mode while developing and quiet mode in production, without touching your code, and route different severities to different places (errors to an alerting system, everything to a log file).

## SYNTAX REFERENCE

```python
import logging

# Basic setup — call this ONCE, typically at the start of your application
logging.basicConfig(
    level=logging.INFO,                                          # minimum severity to actually show
    format="%(asctime)s - %(name)s - %(levelname)s - %(message)s",
    handlers=[
        logging.StreamHandler(),                # prints to the terminal
        logging.FileHandler("pipeline.log"),      # ALSO writes to a file
    ],
)

# Get a logger for the current module (the standard, idiomatic pattern)
logger = logging.getLogger(__name__)

# The five standard severity levels, from least to most severe
logger.debug("Detailed diagnostic info, useful only while developing")
logger.info("Normal operational messages — 'pipeline started', 'processed 500 rows'")
logger.warning("Something unexpected, but not breaking anything")
logger.error("A real problem occurred — this operation failed")
logger.critical("A severe problem — the application likely can't continue")

# Only messages at or above the configured `level` actually get shown/recorded
# e.g. with level=logging.INFO: debug() messages are silently ignored,
#      but info(), warning(), error(), and critical() all show

# Logging an exception WITH its full traceback
try:
    risky_operation()
except Exception:
    logger.exception("risky_operation failed")   # automatically includes the traceback

# f-strings work fine, but the %-style lazy formatting is the traditional
# convention and slightly more efficient (the string isn't built unless the
# message will actually be shown at the current log level)
logger.info("Processed %d orders in %.2f seconds", order_count, elapsed_time)
logger.info(f"Processed {order_count} orders in {elapsed_time:.2f} seconds")   # also fine, simpler

# Changing the level dynamically (e.g. a "verbose" command-line flag)
logger.setLevel(logging.DEBUG)
```

## WORKED EXAMPLE

Converting a `print()`-based batch processing script into one with proper logging — including exception logging:

```python
import logging

# --- Setup, once, typically in your main entry point ---
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler("order_pipeline.log"),
    ],
)
logger = logging.getLogger(__name__)


def clean_order(raw_order):
    order_id = raw_order.get("id", "unknown")
    try:
        cleaned = {
            "id": int(raw_order["id"]),
            "total": float(raw_order["total"]),
        }
        logger.debug("Cleaned order %s successfully", order_id)
        return cleaned
    except (ValueError, KeyError) as e:
        logger.warning("Order %s failed cleaning: %s", order_id, e)
        return None


def process_batch(raw_orders):
    logger.info("Starting batch of %d orders", len(raw_orders))

    cleaned_orders = []
    for raw in raw_orders:
        result = clean_order(raw)
        if result:
            cleaned_orders.append(result)

    success_rate = len(cleaned_orders) / len(raw_orders) * 100
    logger.info("Batch complete: %d/%d orders succeeded (%.1f%%)",
                len(cleaned_orders), len(raw_orders), success_rate)

    if success_rate < 50:
        logger.error("Success rate below 50%% — this batch likely has a systemic data quality issue")

    return cleaned_orders


raw_orders = [
    {"id": "1", "total": "45.99"},
    {"id": "2", "total": "not_a_number"},
    {"id": "3", "total": "89.00"},
]

result = process_batch(raw_orders)

# With level=INFO, the terminal (and order_pipeline.log) show:
# 2024-11-03 14:22:01,123 - INFO - Starting batch of 3 orders
# 2024-11-03 14:22:01,124 - WARNING - Order 2 failed cleaning: could not convert string to float: 'not_a_number'
# 2024-11-03 14:22:01,124 - INFO - Batch complete: 2/3 orders succeeded (66.7%)
#
# Note: the debug() calls from clean_order() do NOT appear, since the
# configured level is INFO — switching to level=logging.DEBUG would reveal
# them too, without changing a single logging call in the code itself


# --- Logging a genuine exception, with full traceback ---
def connect_to_database():
    raise ConnectionError("Could not reach database at db.internal:5432")

try:
    connect_to_database()
except ConnectionError:
    logger.exception("Database connection failed during startup")
    # logger.exception() automatically includes the full traceback in the log output,
    # which logger.error() alone would NOT do
```

## COMMON MISTAKES

**1. Using `print()` for anything that matters in a real pipeline**
```python
print("Processing order 1042")   # vanishes the moment the terminal closes, unattended
```
Fix: `logging` by default, `print()` only for truly throwaway debugging you'll delete before committing — this distinction matters far more once code actually runs unattended.

**2. Setting the level too low (or too high) for the situation**
```python
logging.basicConfig(level=logging.DEBUG)   # in PRODUCTION — floods the log with excessive detail
logging.basicConfig(level=logging.ERROR)     # while DEVELOPING — hides useful info() messages you need
```
Fix: `DEBUG` while actively developing/troubleshooting, `INFO` for normal production operation (shows meaningful progress without excessive noise), reserving `WARNING`/`ERROR` exclusively for genuinely output-worthy production logs on a very high-volume system.

**3. Calling `logging.basicConfig()` more than once, expecting it to reconfigure things**
```python
logging.basicConfig(level=logging.INFO)
# ... later, elsewhere in the same program ...
logging.basicConfig(level=logging.DEBUG)   # has NO EFFECT — basicConfig() only works the FIRST time called
```
Fix: call `basicConfig()` exactly once, early, typically in your `main.py` entry point (Lesson 8.2) — not scattered across multiple modules.

**4. Using `logger.error()` instead of `logger.exception()` inside an `except` block**
```python
except Exception as e:
    logger.error(f"Failed: {e}")   # only shows the exception's MESSAGE, not the full traceback
```
Fix: `logger.exception()` (called only from within an `except` block) automatically attaches the complete traceback, which is usually essential for actually diagnosing what happened — reserve `logger.error()` for cases where you're deliberately constructing your own error message without an active exception.

**5. Creating a logger with `getLogger()` using a hardcoded string instead of `__name__`**
```python
logger = logging.getLogger("my_logger")     # same name everywhere, regardless of which file it's in
logger = logging.getLogger(__name__)          # BEST PRACTICE — automatically reflects the module name
```
Fix: `__name__` automatically becomes the dotted module path (e.g., `ecommerce_pipeline.transform`), which makes it immediately obvious in the log output exactly which part of a multi-module project produced each message — genuinely useful once a project has more than one file.

**Data Engineer Tip:** In a real deployed pipeline, log output is frequently shipped somewhere beyond a local file — a centralized logging platform, a monitoring tool that can alert on `ERROR`-level messages automatically, or a cloud platform's native logging service (Azure Monitor, in your specific roadmap). None of that infrastructure cares about `print()` output at all — it's specifically built around structured logging with severity levels, which is precisely why `logging` (not `print`) is the non-negotiable standard in any codebase meant to run unattended in production.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Set up basic logging with `level=logging.INFO`, create a logger, and log one `info()` message and one `warning()` message.

<details>
<summary>Hint</summary>
`logging.basicConfig(level=logging.INFO)`, then `logging.getLogger(__name__)`.
</details>

<details>
<summary>Solution</summary>

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

logger.info("This is an info message")
logger.warning("This is a warning message")
```
</details>

---

**Exercise 2 (Easy):** Configure logging with `level=logging.WARNING` this time, and log a `debug()`, an `info()`, and a `warning()` message. Predict which will actually appear before running it.

<details>
<summary>Hint</summary>
Only messages at or above the configured level show — with `WARNING`, both `debug()` and `info()` are suppressed.
</details>

<details>
<summary>Solution</summary>

```python
import logging

logging.basicConfig(level=logging.WARNING)
logger = logging.getLogger(__name__)

logger.debug("Debug message")      # NOT shown
logger.info("Info message")         # NOT shown
logger.warning("Warning message")    # shown — this is the only one visible
```
</details>

---

**Exercise 3 (Medium):** Configure logging to write to BOTH the terminal and a file called `app.log`, using a format that includes the timestamp and level. Log a few messages of different severities, then open `app.log` and confirm the messages were written there too.

<details>
<summary>Hint</summary>
Use `handlers=[logging.StreamHandler(), logging.FileHandler("app.log")]` inside `basicConfig()`.
</details>

<details>
<summary>Solution</summary>

```python
import logging

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s - %(levelname)s - %(message)s",
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler("app.log"),
    ],
)
logger = logging.getLogger(__name__)

logger.info("Application started")
logger.warning("Something looks off")
logger.error("Something failed")
```
After running, `app.log` should contain the same three lines seen in the terminal, each prefixed with a timestamp and level.
</details>

---

**Exercise 4 (Medium):** Write a function `divide_safely(a, b)` that attempts `a / b`, using `logger.exception()` to log the full traceback if a `ZeroDivisionError` occurs, and returning `None` in that case. Test it with a valid and an invalid call.

<details>
<summary>Hint</summary>
`logger.exception()` must be called from inside an `except` block to capture the traceback correctly.
</details>

<details>
<summary>Solution</summary>

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def divide_safely(a, b):
    try:
        return a / b
    except ZeroDivisionError:
        logger.exception("Division failed")
        return None

print(divide_safely(10, 2))   # 5.0, no log output
print(divide_safely(10, 0))    # None, PLUS a full traceback logged via logger.exception()
```
</details>

---

**Exercise 5 (Hard):** Build a small batch-processing function `process_batch(raw_values)` that attempts to convert each value in a list to a float, logging: an `info()` message at the start showing the batch size, a `warning()` for each value that fails conversion (including the bad value itself in the message), an `info()` summary at the end showing success/failure counts, and an `error()` message specifically if MORE THAN half the batch failed. Test with `raw_values = ["10.5", "abc", "20.0", "xyz", "30.5"]` and confirm the right severity levels appear for this specific input (3/5 succeed, so the error-level "more than half failed" message should NOT trigger — adjust your test data or re-verify the logic if you want to see it trigger).

<details>
<summary>Hint</summary>
Loop through `raw_values`, using `try/except ValueError` per item, logging a `warning()` on failure. After the loop, compute the failure rate and conditionally log an `error()` if it exceeds 50%.
</details>

<details>
<summary>Solution</summary>

```python
import logging

logging.basicConfig(level=logging.INFO, format="%(levelname)s - %(message)s")
logger = logging.getLogger(__name__)

def process_batch(raw_values):
    logger.info("Starting batch of %d values", len(raw_values))

    successes = []
    failures = 0

    for value in raw_values:
        try:
            successes.append(float(value))
        except ValueError:
            logger.warning("Failed to convert value: %r", value)
            failures += 1

    logger.info("Batch complete: %d succeeded, %d failed", len(successes), failures)

    failure_rate = failures / len(raw_values)
    if failure_rate > 0.5:
        logger.error("More than half the batch failed (%.0f%%) — check upstream data quality", failure_rate * 100)

    return successes

raw_values = ["10.5", "abc", "20.0", "xyz", "30.5"]
result = process_batch(raw_values)
print(result)   # [10.5, 20.0, 30.5]

# With this specific input: 2/5 fail (40%) — below the 50% threshold, so the
# error() message does NOT appear. Testing with e.g.
# ["abc", "xyz", "10.5", "qrs"] (3/4 failing, 75%) WOULD trigger it.
```
</details>

---

**Previous lesson:** [8.4 — Robust Error Handling & Custom Exceptions](lesson-8.4-custom-exceptions.md)
**Next lesson:** [8.6 — Writing Tests with pytest](lesson-8.6-pytest.md)
