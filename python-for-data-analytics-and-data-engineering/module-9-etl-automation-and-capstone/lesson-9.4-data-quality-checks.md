# Lesson 9.4 — Data Quality Checks

## CONCEPT

A **data quality check** is an automated assertion about what "good" data looks like, run as part of the pipeline itself — not a manual, after-the-fact inspection. Checks typically cover: completeness (are required fields non-null?), validity (are values within expected ranges/formats?), uniqueness (no unexpected duplicates?), and consistency (do related fields agree with each other?). You can write these as plain assertions, or use a dedicated library like **`pandera`**, which lets you define a reusable schema that validates an entire DataFrame at once.

```python
import pandera as pa

schema = pa.DataFrameSchema({
    "order_id": pa.Column(int, unique=True),
    "total": pa.Column(float, checks=pa.Check.gt(0)),
})
schema.validate(df)   # raises a detailed error if anything violates the schema
```

## WHY IT MATTERS

A pipeline that runs "successfully" but silently loads corrupted, incomplete, or nonsensical data is arguably worse than one that crashes loudly — a crash gets noticed immediately; silently bad data gets discovered weeks later, usually by a confused stakeholder questioning a report, at which point tracing the problem back to its source is far harder. Automated data quality checks are what close this gap: the pipeline itself refuses to load data that fails basic sanity checks, failing loudly and immediately instead of passing a problem downstream.

## SYNTAX REFERENCE

```python
import pandas as pd
import pandera as pa
from pandera import Column, Check, DataFrameSchema

# --- Manual assertions (no extra library needed — fine for simple cases) ---
def validate_orders(df: pd.DataFrame) -> None:
    assert df["order_id"].notna().all(), "Found null order_id values"
    assert df["order_id"].is_unique, "Found duplicate order_id values"
    assert (df["total"] > 0).all(), "Found non-positive order totals"
    assert df["order_date"].notna().all(), "Found null order_date values"

# --- pandera: a declarative schema, validated in one call ---
order_schema = DataFrameSchema({
    "order_id": Column(int, unique=True, nullable=False),
    "total": Column(float, checks=Check.gt(0), nullable=False),
    "status": Column(str, checks=Check.isin(["pending", "shipped", "delivered", "cancelled"])),
    "order_date": Column("datetime64[ns]", nullable=False),
})

order_schema.validate(df)                      # raises pandera.errors.SchemaError on failure
order_schema.validate(df, lazy=True)              # collects ALL violations before raising,
                                                      # instead of stopping at the first one

# Column-level checks available in pandera
Check.gt(0)                 # greater than
Check.ge(0)                   # greater than or equal to
Check.in_range(0, 100)          # within a range
Check.isin(["a", "b", "c"])       # must be one of these values
Check.str_matches(r"^\d{5}$")      # regex match (e.g. a 5-digit zip code)

# Handling a validation failure gracefully in a pipeline (rather than crashing outright)
try:
    order_schema.validate(df, lazy=True)
except pa.errors.SchemaErrors as e:
    logger.error(f"Data quality check failed:\n{e.failure_cases}")
    raise   # often still re-raise — a failed quality check should usually STOP the pipeline
```

## WORKED EXAMPLE

Adding a data quality gate to the ETL pipeline from Lesson 9.1 — the pipeline refuses to load data that fails validation, and logs exactly what went wrong:

```python
import pandas as pd
import pandera as pa
from pandera import Column, Check, DataFrameSchema
import logging

logger = logging.getLogger(__name__)

# Define what "good" order data looks like, once, as a reusable schema
order_schema = DataFrameSchema({
    "order_id": Column(int, unique=True, nullable=False),
    "customer_id": Column(int, nullable=False),
    "total": Column(float, checks=Check.gt(0), nullable=False),
    "order_date": Column("datetime64[ns]", nullable=False),
})


def extract(path: str) -> pd.DataFrame:
    df = pd.read_csv(path)
    logger.info(f"Extracted {len(df)} rows")
    return df


def transform(raw: pd.DataFrame) -> pd.DataFrame:
    df = raw.copy()
    df["total"] = pd.to_numeric(df["total"], errors="coerce")
    df["order_date"] = pd.to_datetime(df["order_date"], errors="coerce")
    df = df.dropna(subset=["order_id", "customer_id", "total", "order_date"])
    df["order_id"] = df["order_id"].astype(int)
    df["customer_id"] = df["customer_id"].astype(int)
    logger.info(f"Transform complete: {len(df)} rows")
    return df


def validate(df: pd.DataFrame) -> pd.DataFrame:
    """The data quality GATE — the pipeline stops here if this fails."""
    try:
        validated = order_schema.validate(df, lazy=True)
        logger.info(f"Data quality check PASSED: {len(validated)} rows")
        return validated
    except pa.errors.SchemaErrors as e:
        logger.error("Data quality check FAILED")
        logger.error(f"Failure details:\n{e.failure_cases}")
        raise   # stop the pipeline — do NOT proceed to load bad data


def load(clean: pd.DataFrame, output_path: str) -> None:
    clean.to_csv(output_path, index=False)
    logger.info(f"Loaded {len(clean)} rows to {output_path}")


def run_pipeline(source_path: str, output_path: str) -> None:
    raw = extract(source_path)
    transformed = transform(raw)
    validated = validate(transformed)   # the quality gate sits between transform and load
    load(validated, output_path)


# --- Demonstrating a failure case ---
bad_data = pd.DataFrame({
    "order_id": [1, 2, 2],          # a DUPLICATE order_id — violates uniqueness
    "customer_id": [101, 102, 103],
    "total": [45.99, -10.00, 89.99],   # a NEGATIVE total — violates the gt(0) check
    "order_date": pd.to_datetime(["2024-11-01", "2024-11-02", "2024-11-03"]),
})

try:
    order_schema.validate(bad_data, lazy=True)
except pa.errors.SchemaErrors as e:
    print(e.failure_cases)
#   schema_context     column           check  failure_case  index
# 0         Column  order_id          unique             2      2
# 1         Column     total  greater_than(0)        -10.0      1
# Both problems are reported together (thanks to lazy=True), rather than
# stopping at just the first one found
```

## COMMON MISTAKES

**1. Writing data quality checks that are too loose to catch real problems**
```python
assert len(df) > 0   # technically a "check," but wouldn't catch almost anything meaningful
```
Fix: think specifically about what "bad" looks like for YOUR data — negative prices, impossible dates, duplicate keys, missing required fields — and write checks that would genuinely catch those specific failure modes, not just a generic "did anything come back at all" check.

**2. Checks that are too strict, rejecting legitimate edge cases**
```python
Check.in_range(1, 5)   # a rating schema — but what if a genuinely valid "0 stars, no rating yet" exists?
```
Fix: validate checks against the real, full range of legitimate data before deploying them — an overly strict check can cause a pipeline to fail on perfectly valid data, which erodes trust in the checks themselves and tempts someone to just disable them later.

**3. Using `lazy=False` (the default) and only ever seeing the FIRST failure**
```python
order_schema.validate(df)   # stops and raises at the FIRST violation found
```
If there are actually 5 different data quality problems in a batch, fixing the first one and re-running only reveals the second — a frustrating, slow debugging loop. Fix: use `lazy=True` during development and in most pipeline contexts, so every violation is reported together in one pass.

**4. Letting a validation failure crash the pipeline with no useful log message**
```python
order_schema.validate(df)   # an unhandled pandera.errors.SchemaError propagates with no pipeline-specific context
```
Fix: wrap validation in a `try/except` (as in the worked example) specifically to log a clear, pipeline-relevant message — including the actual failure details (`e.failure_cases` for pandera) — before re-raising, so whoever reviews the logs later immediately understands what went wrong and in which rows.

**5. Running data quality checks too late, after data has already been loaded**
Validating AFTER `load()` has already written to the destination table means bad data was already there, even briefly — and if nothing catches it immediately, it may get queried or reported on before anyone notices. Fix: validation should be a gate BEFORE load, exactly as positioned in the worked example's `run_pipeline()` — bad data should never reach the destination in the first place.

**Data Engineer Tip:** A pandera schema is also genuinely useful as living, enforced documentation of what a dataset is supposed to look like — anyone reading the pipeline code can see the expected shape, types, and constraints in one place, without needing to reverse-engineer it from scattered `if` statements. Many teams treat a documented, versioned data schema as important a deliverable as the pipeline code itself, and a `pandera` schema file in your capstone project's repository is a concrete, visible way to demonstrate that discipline.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Write a manual assertion-based validation function `validate_basic(df)` that checks a `"price"` column has no null values and every value is greater than zero. Test it against both a valid and an invalid DataFrame.

<details>
<summary>Hint</summary>
`assert df["price"].notna().all()` and `assert (df["price"] > 0).all()`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

def validate_basic(df: pd.DataFrame) -> None:
    assert df["price"].notna().all(), "Found null prices"
    assert (df["price"] > 0).all(), "Found non-positive prices"

good = pd.DataFrame({"price": [10.0, 20.0, 5.0]})
validate_basic(good)   # passes silently

bad = pd.DataFrame({"price": [10.0, -5.0, 5.0]})
try:
    validate_basic(bad)
except AssertionError as e:
    print(f"Validation failed: {e}")
```
</details>

---

**Exercise 2 (Easy):** Install `pandera` and define a `DataFrameSchema` for a simple `products` DataFrame with `product_id` (unique int), `price` (float, must be greater than 0), and `category` (string, must be one of `"Electronics"`, `"Home"`, `"Outdoor"`). Validate a small valid DataFrame against it.

<details>
<summary>Hint</summary>
`Column(int, unique=True)`, `Column(float, checks=Check.gt(0))`, `Column(str, checks=Check.isin([...]))`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
from pandera import Column, Check, DataFrameSchema

product_schema = DataFrameSchema({
    "product_id": Column(int, unique=True),
    "price": Column(float, checks=Check.gt(0)),
    "category": Column(str, checks=Check.isin(["Electronics", "Home", "Outdoor"])),
})

products = pd.DataFrame({
    "product_id": [1, 2, 3],
    "price": [24.99, 34.00, 19.99],
    "category": ["Electronics", "Home", "Outdoor"],
})

validated = product_schema.validate(products)
print(validated)
print("Validation passed")
```
</details>

---

**Exercise 3 (Medium):** Given the `product_schema` from Exercise 2, validate a DataFrame that has a duplicate `product_id` AND an invalid `category` value (something not in the allowed list), using `lazy=True` so both problems are reported together. Print the failure cases.

<details>
<summary>Hint</summary>
`product_schema.validate(bad_df, lazy=True)`, wrapped in a `try/except pa.errors.SchemaErrors`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import pandera as pa
from pandera import Column, Check, DataFrameSchema

product_schema = DataFrameSchema({
    "product_id": Column(int, unique=True),
    "price": Column(float, checks=Check.gt(0)),
    "category": Column(str, checks=Check.isin(["Electronics", "Home", "Outdoor"])),
})

bad_products = pd.DataFrame({
    "product_id": [1, 2, 2],
    "price": [24.99, 34.00, 19.99],
    "category": ["Electronics", "Home", "Toys"],
})

try:
    product_schema.validate(bad_products, lazy=True)
except pa.errors.SchemaErrors as e:
    print(e.failure_cases)
# Should report BOTH the duplicate product_id AND the invalid "Toys" category
```
</details>

---

**Exercise 4 (Medium):** Add a data quality gate to a small pipeline: write `transform()` (clean a raw orders DataFrame) and `validate()` (using either manual assertions or a pandera schema of your choice, checking at minimum: no null order_ids, positive totals) such that `validate()` raises and the pipeline logs an error and does NOT proceed to a (simulated) `load()` step if validation fails. Demonstrate both a passing run and a failing run.

<details>
<summary>Hint</summary>
Structure as `extract -> transform -> validate -> load`, with `validate()` raising on failure so `run_pipeline()`'s `load()` call is never reached if validation fails — wrap the whole call in a `try/except` at the top level to demonstrate the failure path without crashing your test script.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def transform(raw: pd.DataFrame) -> pd.DataFrame:
    df = raw.copy()
    df["total"] = pd.to_numeric(df["total"], errors="coerce")
    return df

def validate(df: pd.DataFrame) -> pd.DataFrame:
    try:
        assert df["order_id"].notna().all(), "Found null order_id"
        assert (df["total"] > 0).all(), "Found non-positive total"
        logger.info("Validation passed")
        return df
    except AssertionError as e:
        logger.error(f"Validation FAILED: {e}")
        raise

def load(df: pd.DataFrame) -> None:
    logger.info(f"Loaded {len(df)} rows")

def run_pipeline(raw: pd.DataFrame) -> None:
    clean = transform(raw)
    validated = validate(clean)
    load(validated)

# Passing case
good_raw = pd.DataFrame({"order_id": [1, 2, 3], "total": [10.0, 20.0, 30.0]})
run_pipeline(good_raw)

# Failing case
bad_raw = pd.DataFrame({"order_id": [1, 2, 3], "total": [10.0, -5.0, 30.0]})
try:
    run_pipeline(bad_raw)
except AssertionError:
    print("Pipeline correctly stopped before loading bad data")
```
</details>

---

**Exercise 5 (Hard):** Design a comprehensive `pandera` schema for an `orders` DataFrame with columns: `order_id` (unique, non-null int), `customer_id` (non-null int), `total` (float, must be between $0.01 and $10,000 — use `Check.in_range`), `status` (string, one of `"pending"`, `"shipped"`, `"delivered"`, `"cancelled"`), and `order_date` (datetime, non-null). Build a small test DataFrame with at least 3 distinct validation problems spread across different rows/columns, validate with `lazy=True`, and write code that logs each distinct failure as a separate, readable log line (not just the raw `failure_cases` DataFrame dump) by iterating over `e.failure_cases.itertuples()`.

<details>
<summary>Hint</summary>
`e.failure_cases` is itself a DataFrame with columns like `column`, `check`, `failure_case`, `index` — loop over its rows with `.itertuples()` and build a readable message per row.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import pandera as pa
from pandera import Column, Check, DataFrameSchema
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

order_schema = DataFrameSchema({
    "order_id": Column(int, unique=True, nullable=False),
    "customer_id": Column(int, nullable=False),
    "total": Column(float, checks=Check.in_range(0.01, 10000)),
    "status": Column(str, checks=Check.isin(["pending", "shipped", "delivered", "cancelled"])),
    "order_date": Column("datetime64[ns]", nullable=False),
})

test_orders = pd.DataFrame({
    "order_id": [1, 2, 3, 3],                                    # duplicate order_id (row 3)
    "customer_id": [101, 102, 103, 104],
    "total": [45.99, -10.00, 89.99, 15000.00],                      # negative AND over-limit totals
    "status": ["shipped", "pending", "unknown_status", "delivered"],   # invalid status value
    "order_date": pd.to_datetime(["2024-11-01", "2024-11-02", "2024-11-03", "2024-11-04"]),
})

try:
    order_schema.validate(test_orders, lazy=True)
    logger.info("All validation checks passed")
except pa.errors.SchemaErrors as e:
    logger.error(f"Data quality check failed with {len(e.failure_cases)} issue(s):")
    for row in e.failure_cases.itertuples():
        logger.error(
            f"  Row index {row.index}: column '{row.column}' failed check "
            f"'{row.check}' — value was {row.failure_case!r}"
        )
```
</details>

---

**Previous lesson:** [9.3 — Scheduling & an Intro to Orchestration](lesson-9.3-scheduling-and-orchestration.md)
**Next lesson:** [9.5 — Logging, Alerting & Failure Handling](lesson-9.5-alerting-and-failure-handling.md)
