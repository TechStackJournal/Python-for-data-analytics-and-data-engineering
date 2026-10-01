# Lesson 8.6 — Writing Tests with `pytest`

## CONCEPT

A **test** is code that checks whether another piece of code actually does what it's supposed to — automatically, repeatedly, without you manually re-verifying by hand every time you make a change. **`pytest`** is the standard Python testing framework: you write functions starting with `test_`, each containing one or more `assert` statements, and `pytest` discovers and runs them all, reporting exactly which passed and which failed.

```python
# test_cleaning.py
from cleaning import clean_price

def test_clean_price_strips_dollar_sign():
    assert clean_price("$24.99") == 24.99
```

```bash
pytest   # discovers and runs every test_*.py file, printing a pass/fail summary
```

## WHY IT MATTERS

Every "does this function actually do the right thing?" question you've answered informally throughout this course — by running a script and eyeballing the printed output — is exactly what a test automates, permanently. This matters enormously once a pipeline grows: a test suite lets you change a transformation function with confidence that you haven't silently broken something else, catches a regression immediately rather than weeks later in production, and — as covered in <learnings-and-conventions>, genuinely — is one of the concrete engineering quality signals that differentiates a strong portfolio repository from a purely exploratory one.

## SYNTAX REFERENCE

```python
# A test file, typically named test_<module>.py, in a tests/ folder
# test_transform.py

from ecommerce_pipeline.transform import clean_price, classify_order_size

def test_clean_price_basic():
    assert clean_price("$24.99") == 24.99

def test_clean_price_no_symbol():
    assert clean_price("24.99") == 24.99

def test_classify_order_size_small():
    assert classify_order_size(1) == "small"

def test_classify_order_size_bulk():
    assert classify_order_size(15) == "bulk"

# Running pytest
# pytest                        — run every test in the project
# pytest tests/test_transform.py  — run just one file
# pytest -v                      — verbose output, shows each test name individually
# pytest -k "price"                — run only tests whose name contains "price"

# Testing that an exception is correctly raised
import pytest

def test_clean_price_raises_on_invalid_input():
    with pytest.raises(ValueError):
        clean_price("not_a_price")

# Fixtures — reusable setup shared across multiple tests
@pytest.fixture
def sample_orders():
    return [
        {"id": 1, "total": 45.99},
        {"id": 2, "total": 89.00},
    ]

def test_batch_total(sample_orders):     # pytest automatically provides the fixture's return value
    total = sum(order["total"] for order in sample_orders)
    assert total == 134.99

# Parametrized tests — run the SAME test logic against multiple input/output pairs
@pytest.mark.parametrize("raw_input,expected", [
    ("$24.99", 24.99),
    ("$0.00", 0.00),
    ("$1,299.99", 1299.99),
])
def test_clean_price_multiple_cases(raw_input, expected):
    assert clean_price(raw_input) == expected

# Common assertion patterns
assert result == expected
assert result is None
assert len(result) == 3
assert "key" in result
assert result > 0
```

## WORKED EXAMPLE

Testing the `transform.py` cleaning functions from Lesson 8.2's project structure — covering normal cases, edge cases, and error cases:

```python
# ecommerce_pipeline/transform.py
def clean_price(raw_price):
    if isinstance(raw_price, str):
        cleaned = raw_price.replace("$", "").replace(",", "").strip()
    else:
        cleaned = raw_price
    return float(cleaned)


def classify_order_size(item_count):
    if item_count >= 10:
        return "bulk"
    elif item_count >= 3:
        return "standard"
    else:
        return "small"


def apply_discount(price, discount_pct):
    if not (0 <= discount_pct <= 100):
        raise ValueError(f"discount_pct must be between 0 and 100, got {discount_pct}")
    return round(price * (1 - discount_pct / 100), 2)
```

```python
# tests/test_transform.py
import pytest
from ecommerce_pipeline.transform import clean_price, classify_order_size, apply_discount


class TestCleanPrice:
    """Grouping related tests in a class is a common pytest convention — purely organizational."""

    def test_strips_dollar_sign(self):
        assert clean_price("$24.99") == 24.99

    def test_strips_comma_thousands_separator(self):
        assert clean_price("$1,299.99") == 1299.99

    def test_handles_plain_numeric_string(self):
        assert clean_price("24.99") == 24.99

    def test_raises_on_genuinely_invalid_input(self):
        with pytest.raises(ValueError):
            clean_price("not_a_price")


@pytest.mark.parametrize("item_count,expected_label", [
    (1, "small"),
    (2, "small"),
    (3, "standard"),      # boundary case — exactly at the threshold
    (9, "standard"),
    (10, "bulk"),          # boundary case — exactly at the threshold
    (50, "bulk"),
])
def test_classify_order_size(item_count, expected_label):
    assert classify_order_size(item_count) == expected_label


def test_apply_discount_typical_case():
    assert apply_discount(100.0, 20) == 80.0

def test_apply_discount_zero_percent_is_a_no_op():
    assert apply_discount(50.0, 0) == 50.0

def test_apply_discount_rejects_invalid_percentage():
    with pytest.raises(ValueError):
        apply_discount(100.0, 150)    # 150% discount doesn't make sense — should raise


# Running `pytest -v` from the project root would show:
# tests/test_transform.py::TestCleanPrice::test_strips_dollar_sign PASSED
# tests/test_transform.py::TestCleanPrice::test_strips_comma_thousands_separator PASSED
# tests/test_transform.py::TestCleanPrice::test_handles_plain_numeric_string PASSED
# tests/test_transform.py::TestCleanPrice::test_raises_on_genuinely_invalid_input PASSED
# tests/test_transform.py::test_classify_order_size[1-small] PASSED
# tests/test_transform.py::test_classify_order_size[3-standard] PASSED
# ... (one line per parametrized case) ...
# tests/test_transform.py::test_apply_discount_typical_case PASSED
# tests/test_transform.py::test_apply_discount_zero_percent_is_a_no_op PASSED
# tests/test_transform.py::test_apply_discount_rejects_invalid_percentage PASSED
#
# ==================== 10 passed in 0.03s ====================
```

## COMMON MISTAKES

**1. Testing only the "happy path" and skipping edge cases and error cases**
```python
def test_clean_price():
    assert clean_price("$24.99") == 24.99   # only ONE case tested — what about "$0.00"? Invalid input?
```
Fix: for every function, deliberately think through at least: a typical case, a boundary/edge case (zero, empty, the exact threshold of a conditional), and an error case (invalid input that should raise an exception) — as the worked example's `classify_order_size` boundary tests (`3` and `10`, exactly at the thresholds) demonstrate.

**2. Writing tests that depend on execution order, or on each other**
```python
shared_list = []

def test_step_one():
    shared_list.append(1)
    assert len(shared_list) == 1

def test_step_two():
    assert len(shared_list) == 1   # only passes if test_step_one happened to run FIRST — fragile!
```
Fix: each test should be fully independent and runnable in any order (or entirely alone) — use fixtures (shown in the Syntax Reference) to set up fresh data for each test that needs it, rather than relying on shared mutable state between tests.

**3. Testing implementation details instead of behavior**
```python
def test_internal_variable_name():
    # testing that a function uses a specific internal variable name — meaningless
    # and breaks the moment someone refactors the function's internals, even if
    # its actual behavior is completely unchanged
```
Fix: test what a function *returns* or *does* given specific inputs — its observable behavior — not how it's internally implemented; this way, refactoring the internals doesn't break your tests as long as the behavior stays correct.

**4. Forgetting to test the exception-raising path at all**
```python
def apply_discount(price, discount_pct):
    if not (0 <= discount_pct <= 100):
        raise ValueError(...)
    ...
# but no test ever actually calls apply_discount() with an invalid discount_pct!
```
Fix: `pytest.raises()` (shown in the Syntax Reference) exists specifically to test that a function correctly raises an exception under the right conditions — an untested error path is just as much a gap in coverage as an untested normal path, and arguably more important, since it's exactly the code that's supposed to catch bad data.

**5. Writing one enormous test function that checks many unrelated things at once**
```python
def test_everything():
    assert clean_price("$24.99") == 24.99
    assert classify_order_size(5) == "standard"
    assert apply_discount(100, 10) == 90.0
    # if the FIRST assertion fails, pytest never even tells you about the other two!
```
Fix: one test function per specific behavior being verified — when a test fails, its *name* alone should immediately tell you what broke, without needing to read which specific line inside a giant function failed.

**Data Engineer Tip:** A test suite is what makes Module 9's capstone pipeline genuinely trustworthy to modify later — when you (or a teammate) change a cleaning rule six months from now, running `pytest` immediately confirms whether that change broke any other assumption the rest of the pipeline depends on, without needing to manually re-verify every transformation by hand. This is also exactly the kind of engineering signal — a real, working `tests/` folder with meaningful coverage, not just a token test or two — that differentiates a portfolio project reviewers take seriously from one that looks unfinished.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given a function `def add(a, b): return a + b` in a file `mathutils.py`, write a test file `test_mathutils.py` with a single test confirming `add(2, 3) == 5`. Run it with `pytest`.

<details>
<summary>Hint</summary>
`from mathutils import add`, then `def test_add(): assert add(2, 3) == 5`.
</details>

<details>
<summary>Solution</summary>

```python
# mathutils.py
def add(a, b):
    return a + b
```
```python
# test_mathutils.py
from mathutils import add

def test_add():
    assert add(2, 3) == 5
```
Run with `pytest -v` from the same folder.
</details>

---

**Exercise 2 (Easy):** Add a second test to `test_mathutils.py` confirming that `add(-5, 5) == 0`, and a third confirming `add(0, 0) == 0`. Run `pytest -v` and confirm all three pass.

<details>
<summary>Hint</summary>
Just add two more `def test_...():` functions, each with its own `assert`.
</details>

<details>
<summary>Solution</summary>

```python
# test_mathutils.py
from mathutils import add

def test_add_positive_numbers():
    assert add(2, 3) == 5

def test_add_negative_and_positive():
    assert add(-5, 5) == 0

def test_add_zeros():
    assert add(0, 0) == 0
```
</details>

---

**Exercise 3 (Medium):** Given a function `def classify_temperature(celsius): return "cold" if celsius < 10 else "mild" if celsius < 25 else "hot"`, write parametrized tests covering at least 5 cases, including the exact boundary values (10 and 25).

<details>
<summary>Hint</summary>
Use `@pytest.mark.parametrize("celsius,expected", [...])` with a list of tuples, including `(10, "mild")` and `(25, "hot")` as the boundary cases.
</details>

<details>
<summary>Solution</summary>

```python
# temperature.py
def classify_temperature(celsius):
    if celsius < 10:
        return "cold"
    elif celsius < 25:
        return "mild"
    else:
        return "hot"
```
```python
# test_temperature.py
import pytest
from temperature import classify_temperature

@pytest.mark.parametrize("celsius,expected", [
    (-5, "cold"),
    (9, "cold"),
    (10, "mild"),     # boundary
    (20, "mild"),
    (24, "mild"),
    (25, "hot"),        # boundary
    (35, "hot"),
])
def test_classify_temperature(celsius, expected):
    assert classify_temperature(celsius) == expected
```
</details>

---

**Exercise 4 (Medium):** Given
```python
def safe_divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
```
write tests covering: a normal division case, and a case confirming `ValueError` is correctly raised when dividing by zero (using `pytest.raises`).

<details>
<summary>Hint</summary>
`with pytest.raises(ValueError): safe_divide(10, 0)`
</details>

<details>
<summary>Solution</summary>

```python
# division.py
def safe_divide(a, b):
    if b == 0:
        raise ValueError("Cannot divide by zero")
    return a / b
```
```python
# test_division.py
import pytest
from division import safe_divide

def test_safe_divide_normal_case():
    assert safe_divide(10, 2) == 5.0

def test_safe_divide_raises_on_zero():
    with pytest.raises(ValueError):
        safe_divide(10, 0)
```
</details>

---

**Exercise 5 (Hard):** Given the `apply_discount` function from the Worked Example:
```python
def apply_discount(price, discount_pct):
    if not (0 <= discount_pct <= 100):
        raise ValueError(f"discount_pct must be between 0 and 100, got {discount_pct}")
    return round(price * (1 - discount_pct / 100), 2)
```
Write a comprehensive test suite using a `fixture` for a sample price (e.g. `100.0`), including: (1) at least 3 parametrized "typical" cases covering different discount percentages, (2) a boundary test for exactly `0%` and exactly `100%` discount, (3) a test confirming a `ValueError` is raised for a negative percentage, and (4) a test confirming a `ValueError` is raised for a percentage over 100. Organize the tests inside a class for clarity.

<details>
<summary>Hint</summary>
Use `@pytest.fixture` for the sample price, have test methods accept it as a parameter (pytest will inject it automatically), and combine `@pytest.mark.parametrize` for the typical cases with separate individual test methods for the boundary and error cases.
</details>

<details>
<summary>Solution</summary>

```python
# discounts.py
def apply_discount(price, discount_pct):
    if not (0 <= discount_pct <= 100):
        raise ValueError(f"discount_pct must be between 0 and 100, got {discount_pct}")
    return round(price * (1 - discount_pct / 100), 2)
```
```python
# test_discounts.py
import pytest
from discounts import apply_discount


@pytest.fixture
def sample_price():
    return 100.0


class TestApplyDiscount:

    @pytest.mark.parametrize("discount_pct,expected", [
        (10, 90.0),
        (25, 75.0),
        (50, 50.0),
    ])
    def test_typical_discounts(self, sample_price, discount_pct, expected):
        assert apply_discount(sample_price, discount_pct) == expected

    def test_zero_percent_discount(self, sample_price):
        assert apply_discount(sample_price, 0) == sample_price

    def test_full_hundred_percent_discount(self, sample_price):
        assert apply_discount(sample_price, 100) == 0.0

    def test_raises_on_negative_percentage(self, sample_price):
        with pytest.raises(ValueError):
            apply_discount(sample_price, -10)

    def test_raises_on_over_hundred_percentage(self, sample_price):
        with pytest.raises(ValueError):
            apply_discount(sample_price, 150)
```
Running `pytest -v` shows 7 distinct test results (3 parametrized + 4 individual), each independently readable in the output — if `test_raises_on_negative_percentage` fails, the test name alone tells you exactly what broke, without needing to dig through a combined test function.
</details>

---

**Previous lesson:** [8.5 — Logging Instead of print()](lesson-8.5-logging.md)
**Next lesson:** [8.7 — Type Hints & Static Checking with mypy](lesson-8.7-type-hints-and-mypy.md)
