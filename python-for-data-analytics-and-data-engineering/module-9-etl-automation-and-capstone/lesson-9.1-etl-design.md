# Lesson 9.1 — Designing an ETL Job: Extract, Transform, Load

## CONCEPT

**ETL** (Extract, Transform, Load) is the standard pattern for moving data from a source into a destination while reshaping it along the way:

- **Extract** — pull raw data from a source (a database, an API, a file drop)
- **Transform** — clean, validate, reshape, and enrich it (everything from Modules 1–5)
- **Load** — write the result into its destination (a database table, a file, a warehouse)

```python
def extract() -> pd.DataFrame: ...
def transform(raw: pd.DataFrame) -> pd.DataFrame: ...
def load(clean: pd.DataFrame) -> None: ...

def run_pipeline():
    raw = extract()
    clean = transform(raw)
    load(clean)
```

A closely related variant, **ELT** (Extract, Load, Transform), loads raw data first and transforms it afterward, usually inside the destination warehouse itself (e.g., with dbt) rather than in Python — a pattern this course's roadmap flags as Phase 2 of your Azure track, after this course.

## WHY IT MATTERS

Every lesson in this course, from reading a CSV in Module 2 to writing to a database in Module 7, has been one piece of an ETL pipeline without necessarily being labeled as such. This lesson names the pattern explicitly and gives you a structure to organize those pieces around — the difference between "a script that happens to clean and load data" and "a pipeline with three clearly separated, independently testable stages" is almost entirely about this organizational discipline, not new technical skills.

## SYNTAX REFERENCE

```python
# A minimal three-function ETL skeleton
import pandas as pd
from sqlalchemy import create_engine

def extract(source_path: str) -> pd.DataFrame:
    """Pull raw data from its source. Does NOT clean or validate."""
    return pd.read_csv(source_path)

def transform(raw: pd.DataFrame) -> pd.DataFrame:
    """Clean, validate, and reshape. Pure function: input in, output out."""
    df = raw.copy()
    df["order_total"] = df["order_total"].astype(float)
    df["order_date"] = pd.to_datetime(df["order_date"])
    df = df.dropna(subset=["order_id"])
    return df

def load(clean: pd.DataFrame, engine) -> None:
    """Write the final result to its destination."""
    clean.to_sql("orders_staging", engine, if_exists="append", index=False)

def run_pipeline(source_path: str, engine) -> None:
    """Orchestrate the three stages in order."""
    raw = extract(source_path)
    clean = transform(raw)
    load(clean, engine)

# Each stage can be tested and run INDEPENDENTLY of the others
# e.g., testing transform() doesn't require a real database or file at all —
# just pass it a small in-memory DataFrame (this is exactly what Module 8's
# pytest lesson relies on)
```

## WORKED EXAMPLE

A complete, small ETL pipeline for the `ecommerce` dataset — extracting raw order data from a CSV, transforming it, and loading it into a staging table:

```python
import pandas as pd
from sqlalchemy import create_engine
import logging

logger = logging.getLogger(__name__)


def extract(source_path: str) -> pd.DataFrame:
    """Extract: read the raw orders export, exactly as it arrives."""
    logger.info(f"Extracting from {source_path}")
    df = pd.read_csv(source_path)
    logger.info(f"Extracted {len(df)} raw rows")
    return df


def transform(raw: pd.DataFrame) -> pd.DataFrame:
    """Transform: clean types, drop invalid rows, add a derived column."""
    df = raw.copy()

    # Type cleaning (Module 4 territory)
    df["order_total"] = pd.to_numeric(df["order_total"], errors="coerce")
    df["order_date"] = pd.to_datetime(df["order_date"], errors="coerce")

    # Drop rows that failed to parse, or have no order_id
    before = len(df)
    df = df.dropna(subset=["order_id", "order_total", "order_date"])
    dropped = before - len(df)
    if dropped:
        logger.warning(f"Dropped {dropped} invalid rows during transform")

    # A derived column — a simple enrichment
    df["order_size"] = pd.cut(
        df["order_total"],
        bins=[0, 35, 100, float("inf")],
        labels=["small", "medium", "large"],
    )

    logger.info(f"Transform complete: {len(df)} clean rows")
    return df


def load(clean: pd.DataFrame, engine) -> int:
    """Load: write the clean result into a staging table."""
    rows_written = clean.to_sql("orders_staging", engine, if_exists="append", index=False)
    logger.info(f"Loaded {len(clean)} rows into orders_staging")
    return len(clean)


def run_pipeline(source_path: str, engine) -> None:
    """Orchestrate: run each stage in sequence, with a clear summary at the end."""
    raw = extract(source_path)
    clean = transform(raw)
    rows_loaded = load(clean, engine)
    logger.info(f"Pipeline complete: {rows_loaded} rows loaded from {source_path}")


# Running it (engine setup follows Module 7's SQLAlchemy + dotenv pattern)
if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    engine = create_engine("postgresql://user:pass@localhost:5432/ecommerce")
    run_pipeline("data/raw/orders_export.csv", engine)
```

Notice each function has exactly one job and is independently callable — `transform()` can be unit tested (Module 8, Lesson 8.6) with a small hand-built DataFrame, with no file or database involved at all.

## COMMON MISTAKES

**1. Mixing extraction, transformation, and loading logic together in one function**
```python
def process_orders(path, engine):
    df = pd.read_csv(path)
    df["total"] = df["total"].astype(float)   # transform logic
    df.to_sql("orders", engine)                  # load logic
    # all three stages tangled in one function — hard to test any piece alone
```
Fix: even for a small pipeline, keep the three stages in separate functions (as in the worked example) — the discipline pays off the moment you need to test, debug, or reuse just one stage.

**2. Writing a `transform()` function with side effects (printing, writing files, modifying global state)**
```python
def transform(raw):
    print(f"Processing {len(raw)} rows")   # a side effect — makes testing noisier
    raw["total"] = raw["total"].astype(float)   # ALSO mutates the input in place!
    return raw
```
Fix: `transform()` should be a pure function wherever possible (Module 1, Lesson 1.6) — take a DataFrame in, return a new one out, with logging as the only acceptable "side effect" (and even that should go through the `logging` module, not `print()`, per Module 8).

**3. Forgetting to `.copy()` the input DataFrame before modifying it in `transform()`**
```python
def transform(raw):
    raw["total"] = raw["total"].astype(float)   # modifies the CALLER's DataFrame too!
    return raw
```
Fix: `df = raw.copy()` at the top of `transform()`, exactly as in the worked example — without it, the caller's original `raw` DataFrame gets silently mutated, which can cause confusing bugs if `raw` is reused or logged elsewhere afterward.

**4. Not logging row counts at each stage**
Without `logger.info(f"Extracted {len(df)} rows")`-style logging at each stage boundary, a silent data loss (e.g., a join that drops rows, a filter that's too aggressive) can go completely unnoticed until someone questions why a report looks wrong weeks later. Fix: log the row count in and out of every stage, as in the worked example — it costs one line and is often the first thing you'll check when debugging a pipeline that "ran successfully" but produced wrong results.

**5. Treating ETL as a rigid, one-size-fits-all template**
Not every pipeline needs exactly these three named functions — some need multiple extract sources, some need transform split into several sub-steps, some genuinely are better as ELT (transform happens in the warehouse via SQL/dbt, not in Python at all). Fix: use Extract/Transform/Load as an organizing *principle* — clear separation of concerns, testable stages — rather than a strict template every pipeline must follow exactly.

**Data Engineer Tip:** The specific `extract()` / `transform()` / `load()` function names in this lesson aren't a formal requirement — what matters is the underlying separation of concerns. In a real production codebase, you'll see this same idea expressed through many different tools and names: dbt models expressing the Transform stage declaratively in SQL, Airflow tasks expressing each stage as a node in a DAG (previewed in Lesson 9.3), or a cloud-native service like Azure Data Factory expressing the same three stages as pipeline activities in a visual designer. The three-stage thinking transfers everywhere, even when the specific code structure changes.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Write a minimal `extract()` function that reads a CSV at a given path and returns a DataFrame, logging the row count read. Test it against any small CSV you create.

<details>
<summary>Hint</summary>
`pd.read_csv(path)`, then `logger.info(...)` with `len(df)`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import logging

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)

def extract(path: str) -> pd.DataFrame:
    df = pd.read_csv(path)
    logger.info(f"Extracted {len(df)} rows from {path}")
    return df

# Test
pd.DataFrame({"id": [1, 2, 3], "total": [10.0, 20.0, 15.0]}).to_csv("test.csv", index=False)
result = extract("test.csv")
print(result)
```
</details>

---

**Exercise 2 (Easy):** Write a `transform()` function that takes a DataFrame with a `"total"` column, returns a NEW DataFrame (don't mutate the input) with `"total"` rounded to 2 decimal places. Confirm the original input DataFrame is unchanged after calling it.

<details>
<summary>Hint</summary>
`.copy()` first, then round the copy.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

def transform(raw: pd.DataFrame) -> pd.DataFrame:
    df = raw.copy()
    df["total"] = df["total"].round(2)
    return df

original = pd.DataFrame({"total": [10.456, 20.789]})
result = transform(original)
print("Original unchanged:", original["total"].tolist())   # [10.456, 20.789]
print("Transformed:", result["total"].tolist())              # [10.46, 20.79]
```
</details>

---

**Exercise 3 (Medium):** Build a 3-function pipeline (`extract`, `transform`, `load`) where `load()` simply writes the final DataFrame to a new CSV file (standing in for a database write). `transform()` should drop any rows with a missing `"total"` value and add a `"total_with_tax"` column (8% tax). Run the full pipeline end to end on a small synthetic CSV you create, and print the final loaded file's contents to confirm.

<details>
<summary>Hint</summary>
Use `df.dropna(subset=["total"])` and `df["total"] * 1.08` inside `transform()`; `load()` can just call `.to_csv()`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import logging

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)

def extract(path: str) -> pd.DataFrame:
    df = pd.read_csv(path)
    logger.info(f"Extracted {len(df)} rows")
    return df

def transform(raw: pd.DataFrame) -> pd.DataFrame:
    df = raw.copy()
    df = df.dropna(subset=["total"])
    df["total_with_tax"] = (df["total"] * 1.08).round(2)
    logger.info(f"Transform complete: {len(df)} rows")
    return df

def load(clean: pd.DataFrame, output_path: str) -> None:
    clean.to_csv(output_path, index=False)
    logger.info(f"Loaded {len(clean)} rows to {output_path}")

def run_pipeline(source_path: str, output_path: str) -> None:
    raw = extract(source_path)
    clean = transform(raw)
    load(clean, output_path)

# Set up test data and run
pd.DataFrame({"id": [1, 2, 3], "total": [10.0, None, 25.0]}).to_csv("source.csv", index=False)
run_pipeline("source.csv", "output.csv")
print(pd.read_csv("output.csv"))
```
</details>

---

**Exercise 4 (Medium):** Extend Exercise 3's `transform()` function to log a warning (not just info) whenever rows are dropped, including exactly how many. Test it with a source file that has 2 rows with missing `"total"` values out of 5 total, and confirm the warning fires with the correct count.

<details>
<summary>Hint</summary>
Capture `len(df)` before and after `dropna()`, and use `logger.warning()` only if the difference is greater than zero.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import logging

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)

def transform(raw: pd.DataFrame) -> pd.DataFrame:
    df = raw.copy()
    before = len(df)
    df = df.dropna(subset=["total"])
    dropped = before - len(df)
    if dropped:
        logger.warning(f"Dropped {dropped} rows with missing total")
    df["total_with_tax"] = (df["total"] * 1.08).round(2)
    return df

raw = pd.DataFrame({
    "id": [1, 2, 3, 4, 5],
    "total": [10.0, None, 25.0, None, 40.0],
})
result = transform(raw)
print(result)
# WARNING log should read: "Dropped 2 rows with missing total"
```
</details>

---

**Exercise 5 (Hard):** Design and implement a complete ETL pipeline for a small, slightly more realistic scenario: `extract()` reads two separate CSVs — `orders.csv` (`order_id, customer_id, total, order_date`) and `customers.csv` (`customer_id, tier`) — and returns both as a tuple of DataFrames. `transform()` merges them (an inner join on `customer_id`, reusing Module 5's `.merge()`), cleans types, drops any order with a missing or non-positive `total`, and adds a `discounted_total` column (10% off for `"gold"` tier, no discount otherwise). `load()` writes the final result to `enriched_orders.csv`. Wire all three into a `run_pipeline()` function, add logging at each stage (including a warning for dropped rows), and run it end-to-end on small synthetic data you create.

<details>
<summary>Hint</summary>
`extract()` returns `(orders_df, customers_df)` as a tuple; `transform()` takes both as separate arguments, merges with `pd.merge(orders, customers, on="customer_id", how="inner")`, then applies the cleaning and discount logic from Module 4 (type coercion, `dropna`) and Module 4's `.apply()`/`np.where()` (Lesson 4.9) for the tiered discount.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import numpy as np
import logging

logger = logging.getLogger(__name__)
logging.basicConfig(level=logging.INFO)


def extract(orders_path: str, customers_path: str) -> tuple[pd.DataFrame, pd.DataFrame]:
    orders = pd.read_csv(orders_path)
    customers = pd.read_csv(customers_path)
    logger.info(f"Extracted {len(orders)} orders and {len(customers)} customers")
    return orders, customers


def transform(orders: pd.DataFrame, customers: pd.DataFrame) -> pd.DataFrame:
    df = orders.merge(customers, on="customer_id", how="inner")

    df["total"] = pd.to_numeric(df["total"], errors="coerce")
    df["order_date"] = pd.to_datetime(df["order_date"], errors="coerce")

    before = len(df)
    df = df.dropna(subset=["total", "order_date"])
    df = df[df["total"] > 0]
    dropped = before - len(df)
    if dropped:
        logger.warning(f"Dropped {dropped} invalid or non-positive orders")

    df["discounted_total"] = np.where(
        df["tier"] == "gold",
        (df["total"] * 0.9).round(2),
        df["total"],
    )

    logger.info(f"Transform complete: {len(df)} enriched rows")
    return df


def load(clean: pd.DataFrame, output_path: str) -> None:
    clean.to_csv(output_path, index=False)
    logger.info(f"Loaded {len(clean)} rows to {output_path}")


def run_pipeline(orders_path: str, customers_path: str, output_path: str) -> None:
    orders, customers = extract(orders_path, customers_path)
    clean = transform(orders, customers)
    load(clean, output_path)


# Test data
pd.DataFrame({
    "order_id": [1, 2, 3, 4],
    "customer_id": [101, 102, 101, 103],
    "total": [45.00, -5.00, 120.00, None],
    "order_date": ["2024-11-01", "2024-11-02", "2024-11-03", "2024-11-04"],
}).to_csv("orders.csv", index=False)

pd.DataFrame({
    "customer_id": [101, 102, 103],
    "tier": ["gold", "silver", "bronze"],
}).to_csv("customers.csv", index=False)

run_pipeline("orders.csv", "customers.csv", "enriched_orders.csv")
print(pd.read_csv("enriched_orders.csv"))
```
</details>

---

**Next lesson:** [9.2 — Idempotency & Incremental Loads](lesson-9.2-idempotency-and-incremental-loads.md)
