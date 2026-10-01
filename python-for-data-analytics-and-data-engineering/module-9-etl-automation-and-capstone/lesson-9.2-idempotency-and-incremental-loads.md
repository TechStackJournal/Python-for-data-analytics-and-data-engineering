# Lesson 9.2 — Idempotency & Incremental Loads

## CONCEPT

A pipeline is **idempotent** if running it multiple times with the same input produces the same final result as running it once — no duplicate rows, no doubled totals, regardless of how many times (or why) it re-runs. An **incremental load** is a related but distinct technique: instead of reprocessing an entire dataset every run, the pipeline only extracts and processes what's *new or changed* since the last successful run, tracked via a **watermark** — typically the maximum timestamp or ID already processed.

```python
# Idempotent load — safe to re-run, never creates duplicates
df.to_sql("orders", engine, if_exists="replace")   # one (blunt) approach: full overwrite

# Incremental extraction — only pull what's new since the watermark
last_watermark = get_last_watermark()
new_rows = pd.read_sql(
    f"SELECT * FROM orders WHERE updated_at > '{last_watermark}'", engine
)
```

## WHY IT MATTERS

Pipelines fail and get re-run constantly — a network blip, a server restart, a manual retry after fixing a bug. A pipeline that isn't idempotent accumulates duplicate data every single time this happens, silently corrupting downstream reports in a way that's often discovered far too late. Incremental loading is the performance half of the same coin: reprocessing an entire multi-million-row table on every run, every day, becomes untenable at scale — pulling only what changed since last time is usually dramatically cheaper and faster. Both concepts, together, are what separate a script that merely "works" from one that's genuinely safe and efficient to run unattended, on a schedule, indefinitely.

## SYNTAX REFERENCE

```python
import pandas as pd
from sqlalchemy import create_engine, text

# --- Idempotency pattern 1: full overwrite (simplest, works for small tables) ---
clean_df.to_sql("orders_staging", engine, if_exists="replace", index=False)
# Every run completely replaces the table — inherently idempotent, but
# expensive and impractical for large tables, and loses any history

# --- Idempotency pattern 2: delete-then-insert for a specific batch/date ---
def load_for_date(df: pd.DataFrame, target_date: str, engine) -> None:
    with engine.begin() as conn:                      # a transaction — all or nothing
        conn.execute(
            text("DELETE FROM orders WHERE order_date = :d"),
            {"d": target_date},
        )
        df.to_sql("orders", conn, if_exists="append", index=False)
# Re-running for the same target_date is now safe — the old rows for that
# date are cleared first, so nothing is duplicated

# --- Idempotency pattern 3: upsert (insert new, update existing, by a key) ---
# pandas' to_sql() doesn't support upsert directly — typically done with raw SQL
upsert_sql = text("""
    INSERT INTO orders (order_id, total, status)
    VALUES (:order_id, :total, :status)
    ON CONFLICT (order_id)
    DO UPDATE SET total = EXCLUDED.total, status = EXCLUDED.status
""")
with engine.begin() as conn:
    for row in clean_df.to_dict("records"):
        conn.execute(upsert_sql, row)

# --- Incremental extraction using a watermark ---
def get_last_watermark(engine) -> str:
    """Read the last successfully processed timestamp from a tracking table."""
    with engine.connect() as conn:
        result = conn.execute(text("SELECT MAX(loaded_through) FROM pipeline_watermark"))
        value = result.scalar()
        return value.isoformat() if value else "1970-01-01"

def extract_incremental(engine, watermark: str) -> pd.DataFrame:
    query = text("SELECT * FROM orders WHERE updated_at > :wm")
    return pd.read_sql(query, engine, params={"wm": watermark})

def update_watermark(engine, new_watermark: str) -> None:
    with engine.begin() as conn:
        conn.execute(
            text("INSERT INTO pipeline_watermark (loaded_through) VALUES (:wm)"),
            {"wm": new_watermark},
        )
```

## WORKED EXAMPLE

Building an incremental, idempotent order-loading pipeline — pulling only new/updated orders since the last run, and safely re-runnable without creating duplicates:

```python
import pandas as pd
from sqlalchemy import create_engine, text
import logging

logger = logging.getLogger(__name__)


def get_last_watermark(engine) -> str:
    """Returns the latest watermark recorded from a previous successful run."""
    with engine.connect() as conn:
        result = conn.execute(text("SELECT MAX(loaded_through) FROM pipeline_watermark"))
        value = result.scalar()
    watermark = value.isoformat() if value else "1970-01-01T00:00:00"
    logger.info(f"Last watermark: {watermark}")
    return watermark


def extract_incremental(engine, watermark: str) -> pd.DataFrame:
    """Pull only orders updated since the last watermark."""
    query = text("SELECT * FROM orders_source WHERE updated_at > :wm ORDER BY updated_at")
    df = pd.read_sql(query, engine, params={"wm": watermark})
    logger.info(f"Extracted {len(df)} new/updated rows since {watermark}")
    return df


def transform(raw: pd.DataFrame) -> pd.DataFrame:
    df = raw.copy()
    df["total"] = pd.to_numeric(df["total"], errors="coerce")
    df = df.dropna(subset=["order_id", "total"])
    return df


def load_idempotent(clean: pd.DataFrame, engine) -> None:
    """Upsert: safe to re-run — existing order_ids get updated, not duplicated."""
    if clean.empty:
        logger.info("No rows to load")
        return

    upsert_sql = text("""
        INSERT INTO orders (order_id, customer_id, total, updated_at)
        VALUES (:order_id, :customer_id, :total, :updated_at)
        ON CONFLICT (order_id)
        DO UPDATE SET
            total = EXCLUDED.total,
            updated_at = EXCLUDED.updated_at
    """)
    with engine.begin() as conn:
        for row in clean.to_dict("records"):
            conn.execute(upsert_sql, row)
    logger.info(f"Upserted {len(clean)} rows into orders")


def record_watermark(clean: pd.DataFrame, engine) -> None:
    """Only advance the watermark AFTER a successful load — critical for safety."""
    if clean.empty:
        return
    new_watermark = clean["updated_at"].max()
    with engine.begin() as conn:
        conn.execute(
            text("INSERT INTO pipeline_watermark (loaded_through) VALUES (:wm)"),
            {"wm": new_watermark},
        )
    logger.info(f"Watermark advanced to {new_watermark}")


def run_incremental_pipeline(engine) -> None:
    watermark = get_last_watermark(engine)
    raw = extract_incremental(engine, watermark)
    clean = transform(raw)
    load_idempotent(clean, engine)
    record_watermark(clean, engine)   # ONLY after load succeeds
    logger.info("Incremental pipeline run complete")


# Running this function twice in a row with no new source data extracts
# ZERO rows the second time (nothing newer than the watermark), and running
# it again after the SAME source data is still present a second time
# (e.g., a retry after a crash) safely re-upserts the same rows without
# creating duplicates, thanks to the ON CONFLICT clause
```

## COMMON MISTAKES

**1. Advancing the watermark BEFORE confirming the load succeeded**
```python
update_watermark(engine, new_watermark)   # watermark moves first...
load(clean, engine)                          # ...but THIS fails!
# Next run starts from the new watermark, permanently skipping the rows
# that were never actually loaded — silent, hard-to-detect data loss
```
Fix: always advance the watermark *last*, only after the load step has genuinely succeeded — exactly the order shown in the worked example's `run_incremental_pipeline()`.

**2. Using plain `INSERT` for a table where re-runs are expected**
```python
clean.to_sql("orders", engine, if_exists="append")   # a re-run duplicates every row!
```
Fix: `if_exists="append"` alone is NOT idempotent — use an upsert (`ON CONFLICT`), a delete-then-insert for a specific batch key, or a full `if_exists="replace"` overwrite, depending on your situation.

**3. Choosing the wrong column as a watermark**
A created-at timestamp that never changes after a row is inserted won't catch *updates* to existing rows — if `orders.status` can change from "pending" to "shipped" after creation, you need an `updated_at` column (maintained by the source system) as your watermark, not `created_at`.

**4. Forgetting that a watermark-based extraction misses rows updated exactly at the boundary**
```python
WHERE updated_at > :watermark   # a row updated at EXACTLY the watermark timestamp is missed
```
In practice this is usually an edge case worth being aware of rather than solving perfectly — a small overlap window (e.g., `updated_at > :watermark - INTERVAL '1 minute'`, combined with an idempotent upsert so reprocessing a row is harmless) is a common pragmatic mitigation.

**5. Testing idempotency only once instead of actually re-running the pipeline**
The only real way to confirm a pipeline is idempotent is to genuinely run it twice (or more) against the same data and check the row count/totals didn't change the second time — reasoning about it in your head isn't a substitute for actually verifying it, and it's a specific, valuable kind of test to automate (Module 8's `pytest` lesson).

**Data Engineer Tip:** "Is this pipeline idempotent?" is one of the first questions an experienced data engineer asks when reviewing someone else's pipeline design or pull request — it's considered a near-non-negotiable baseline requirement for anything that runs on a schedule, precisely because failures and retries are not edge cases in production, they're a routine, expected part of operating any system at scale. Framing your capstone project's design decisions log (mentioned in the SQL course conventions) around exactly this question — "how does this pipeline handle being re-run?" — is a strong, specific signal of production-mindedness to anyone reviewing your portfolio.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given a small `orders` DataFrame and a pandas-only "database" (just another DataFrame variable, standing in for a real table), write a `load_replace()` function that implements the simplest idempotency pattern — fully replacing the destination with the new data every time — and confirm running it twice in a row produces the same final row count, not double.

<details>
<summary>Hint</summary>
Just reassign the "destination" variable to a copy of the input each time — this is the full-overwrite pattern in its simplest form.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

destination = pd.DataFrame()   # stand-in for a database table

def load_replace(df: pd.DataFrame) -> None:
    global destination
    destination = df.copy()

orders = pd.DataFrame({"order_id": [1, 2, 3], "total": [10, 20, 30]})

load_replace(orders)
print(len(destination))   # 3

load_replace(orders)   # running it again
print(len(destination))   # still 3 — not 6
```
</details>

---

**Exercise 2 (Easy):** Explain, in a short comment, why `df.to_sql("orders", engine, if_exists="append")` is NOT idempotent, and what would happen if a scheduled job using this line failed halfway through and was automatically retried by the scheduler.

<details>
<summary>Hint</summary>
Think about what happens to the rows that were SUCCESSFULLY inserted before the failure, once the retry runs the whole script again from the start.
</details>

<details>
<summary>Solution</summary>

```python
# `if_exists="append"` always INSERTS every row in the DataFrame, with no
# check for whether that row already exists in the destination table.
# If a job using this line fails partway through (e.g., after successfully
# inserting rows 1-500 of 1000, then crashing), a scheduler's automatic
# retry would re-run the ENTIRE script from the start — re-extracting and
# re-inserting ALL 1000 rows, including the 500 that already succeeded
# the first time. The result: rows 1-500 now exist TWICE in the table,
# silently inflating any totals or counts computed from it.
```
</details>

---

**Exercise 3 (Medium):** Implement the "delete-then-insert for a specific batch" pattern using plain pandas (no real database needed — simulate the destination as a DataFrame variable). Write a `load_for_date(df, target_date, destination)` function that removes any existing rows in `destination` matching `target_date`, then appends the new `df` rows, returning the updated destination. Test by loading the same `target_date` twice with different data each time, confirming the second load fully replaces the first rather than appending to it.

<details>
<summary>Hint</summary>
Filter the destination with `destination[destination["order_date"] != target_date]`, then `pd.concat()` the filtered destination with the new data.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

def load_for_date(df: pd.DataFrame, target_date: str, destination: pd.DataFrame) -> pd.DataFrame:
    kept = destination[destination["order_date"] != target_date]
    return pd.concat([kept, df], ignore_index=True)

destination = pd.DataFrame(columns=["order_id", "order_date", "total"])

# First load for 2024-11-01
batch_1 = pd.DataFrame({"order_id": [1, 2], "order_date": ["2024-11-01", "2024-11-01"], "total": [10, 20]})
destination = load_for_date(batch_1, "2024-11-01", destination)
print(destination)   # 2 rows

# Re-run for the SAME date with corrected data (e.g., a bug fix + retry)
batch_1_corrected = pd.DataFrame({"order_id": [1, 2, 3], "order_date": ["2024-11-01"] * 3, "total": [11, 21, 31]})
destination = load_for_date(batch_1_corrected, "2024-11-01", destination)
print(destination)   # 3 rows — the OLD 2024-11-01 rows were replaced, not duplicated
```
</details>

---

**Exercise 4 (Medium):** Simulate a watermark-based incremental extraction using plain pandas. Given a "source" DataFrame with an `updated_at` column (as strings, converted to datetime), write `extract_incremental(source_df, watermark)` that returns only rows with `updated_at` strictly greater than the given watermark. Then write `get_new_watermark(extracted_df)` that returns the max `updated_at` from the extracted rows (or `None` if empty). Demonstrate running extraction twice: once from an old watermark (pulling several rows), and again using the new watermark it returns (pulling zero rows, since nothing is newer).

<details>
<summary>Hint</summary>
Convert the watermark and the column to actual `datetime` objects before comparing (Module 2, Lesson 2.4), then filter with a boolean mask.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

source = pd.DataFrame({
    "order_id": [1, 2, 3, 4],
    "updated_at": ["2024-11-01", "2024-11-03", "2024-11-05", "2024-11-02"],
})
source["updated_at"] = pd.to_datetime(source["updated_at"])

def extract_incremental(source_df: pd.DataFrame, watermark: pd.Timestamp) -> pd.DataFrame:
    return source_df[source_df["updated_at"] > watermark]

def get_new_watermark(extracted_df: pd.DataFrame):
    if extracted_df.empty:
        return None
    return extracted_df["updated_at"].max()

old_watermark = pd.Timestamp("1970-01-01")
first_batch = extract_incremental(source, old_watermark)
print(len(first_batch))   # 4 — everything is newer than 1970

new_watermark = get_new_watermark(first_batch)
print(new_watermark)       # 2024-11-05

second_batch = extract_incremental(source, new_watermark)
print(len(second_batch))   # 0 — nothing newer than the new watermark
```
</details>

---

**Exercise 5 (Hard):** Build a complete simulated incremental + idempotent pipeline using only pandas (standing in for a real database with module-level variables for the "source" table, the "destination" table, and the "watermark"). Requirements: (1) `extract_incremental()` pulls source rows newer than the stored watermark; (2) `transform()` cleans types and drops invalid rows; (3) `load_upsert()` implements upsert behavior manually in pandas — for each incoming `order_id`, if it already exists in the destination, replace that row; otherwise append it (hint: concatenate, then `drop_duplicates(subset=["order_id"], keep="last")`); (4) the watermark only advances after a successful load. Demonstrate the full pipeline across two runs: run 1 with an initial batch of orders, run 2 after adding 2 new orders AND updating the `total` of one existing order in the simulated source — confirming the destination ends up with the correct final state (no duplicates, updated row reflects the new total) and the watermark only reflects genuinely processed data.

<details>
<summary>Hint</summary>
For the upsert step, `pd.concat([destination, new_clean_rows])` followed by `.sort_values("updated_at").drop_duplicates(subset=["order_id"], keep="last")` gives you "latest version of each order_id wins," which is exactly upsert semantics done manually in pandas.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd

destination = pd.DataFrame(columns=["order_id", "total", "updated_at"])
watermark = pd.Timestamp("1970-01-01")

source = pd.DataFrame({
    "order_id": [1, 2, 3],
    "total": [45.0, 89.0, 120.0],
    "updated_at": pd.to_datetime(["2024-11-01", "2024-11-01", "2024-11-02"]),
})


def extract_incremental(source_df, wm):
    return source_df[source_df["updated_at"] > wm]


def transform(raw):
    df = raw.copy()
    df["total"] = pd.to_numeric(df["total"], errors="coerce")
    df = df.dropna(subset=["order_id", "total"])
    return df


def load_upsert(clean, dest):
    if clean.empty:
        return dest
    combined = pd.concat([dest, clean], ignore_index=True)
    combined = combined.sort_values("updated_at").drop_duplicates(subset=["order_id"], keep="last")
    return combined.reset_index(drop=True)


def run_pipeline(source_df, dest, wm):
    raw = extract_incremental(source_df, wm)
    if raw.empty:
        print("Nothing new to process")
        return dest, wm
    clean = transform(raw)
    new_dest = load_upsert(clean, dest)
    new_wm = clean["updated_at"].max()
    return new_dest, new_wm


# Run 1
destination, watermark = run_pipeline(source, destination, watermark)
print("After run 1:")
print(destination)
print("Watermark:", watermark)

# Simulate new source data: 2 new orders + an UPDATE to order_id=1's total
source = pd.concat([
    source,
    pd.DataFrame({
        "order_id": [4, 5],
        "total": [25.0, 60.0],
        "updated_at": pd.to_datetime(["2024-11-03", "2024-11-03"]),
    }),
], ignore_index=True)
source.loc[source["order_id"] == 1, "total"] = 99.0
source.loc[source["order_id"] == 1, "updated_at"] = pd.Timestamp("2024-11-03")

# Run 2
destination, watermark = run_pipeline(source, destination, watermark)
print("\nAfter run 2:")
print(destination)   # 5 rows total, order_id 1 shows total=99.0 (updated, not duplicated)
print("Watermark:", watermark)
```
</details>

---

**Previous lesson:** [9.1 — Designing an ETL Job](lesson-9.1-etl-design.md)
**Next lesson:** [9.3 — Scheduling & an Intro to Orchestration](lesson-9.3-scheduling-and-orchestration.md)
