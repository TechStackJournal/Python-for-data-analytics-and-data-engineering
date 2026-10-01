# Lesson 7.3 — Writing DataFrames Back to a Database

## CONCEPT

Just as `pd.read_sql()` reads a database table into a DataFrame, **`.to_sql()`** does the reverse — writing a DataFrame out to a database table. This is the "load" step of an ETL pipeline: after extracting and cleaning data, you need to actually get it back into the database.

```python
df.to_sql("staging_orders", engine, if_exists="replace", index=False)
```

The `if_exists` parameter controls what happens if the target table already exists — a critical decision that shapes whether your pipeline is safely rerunnable (idempotent, as introduced in Lesson 4.8) or one that silently duplicates data every time it runs.

## WHY IT MATTERS

Loading data is where a pipeline either behaves predictably on every rerun, or slowly accumulates duplicate/inconsistent data over time. Getting comfortable with `.to_sql()`'s options — and with **upserts** (insert-or-update logic) for cases where simple replace/append isn't the right behavior — is directly what separates a script that works once from a pipeline you can trust to run on a schedule, which is exactly the standard your portfolio principles call for.

## SYNTAX REFERENCE

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql://postgres:password@localhost:5432/ecommerce")

# Basic write — creates the table if it doesn't exist
df.to_sql("staging_orders", engine, index=False)

# if_exists controls behavior when the table ALREADY exists:
df.to_sql("staging_orders", engine, if_exists="fail", index=False)      # default — raises an error
df.to_sql("staging_orders", engine, if_exists="replace", index=False)     # DROPS and recreates the table
df.to_sql("staging_orders", engine, if_exists="append", index=False)       # adds rows to the existing table

# index=False is almost always what you want — otherwise pandas writes the
# DataFrame's row index as an extra, usually meaningless column
df.to_sql("staging_orders", engine, index=False)      # no extra index column
df.to_sql("staging_orders", engine)                      # WITHOUT index=False — adds an "index" column!

# Writing in batches, for large DataFrames (reduces memory pressure during the write)
df.to_sql("staging_orders", engine, if_exists="append", index=False, chunksize=1000)

# Specifying column types explicitly (rather than relying on pandas' auto-inference)
from sqlalchemy import types
df.to_sql(
    "staging_orders",
    engine,
    if_exists="replace",
    index=False,
    dtype={"order_id": types.Integer(), "total": types.Numeric(10, 2)},
)

# An UPSERT (insert new rows, update existing ones by a key) — pandas has no
# built-in upsert, so this is typically done with raw SQL via SQLAlchemy directly
from sqlalchemy import text

upsert_query = text("""
    INSERT INTO customers (customer_id, name, email)
    VALUES (:customer_id, :name, :email)
    ON CONFLICT (customer_id)
    DO UPDATE SET name = EXCLUDED.name, email = EXCLUDED.email;
""")

with engine.connect() as conn:
    for row in df.to_dict("records"):
        conn.execute(upsert_query, row)
    conn.commit()
```

## WORKED EXAMPLE

Loading a cleaned DataFrame into a staging table, then performing a proper upsert into a "real" table — the two most common load patterns in practice:

```python
import pandas as pd
from sqlalchemy import create_engine, text

engine = create_engine("postgresql://postgres:your_password@localhost:5432/ecommerce")

# A cleaned batch of new/updated customer records (as if freshly extracted and cleaned)
customers_batch = pd.DataFrame({
    "customer_id": [101, 102, 103],
    "name": ["Maria Chen", "James Lee", "Priya Patel"],
    "email": ["maria@example.com", "james@example.com", "priya@example.com"],
})

# --- Pattern 1: Full replace of a staging table ---
# Appropriate for a temporary landing area that gets fully rebuilt on every run
customers_batch.to_sql("staging_customers", engine, if_exists="replace", index=False)
print("Staging table rebuilt with", len(customers_batch), "rows")

# --- Pattern 2: Append-only load (only correct if you're SURE these are all new rows) ---
new_orders = pd.DataFrame({
    "order_id": [5001, 5002],
    "customer_id": [101, 102],
    "total": [45.99, 89.99],
})
new_orders.to_sql("orders", engine, if_exists="append", index=False)
# DANGEROUS if this script re-runs and these same order_ids already exist —
# it would insert duplicates, exactly the problem covered in Lesson 4.8

# --- Pattern 3: A proper UPSERT — the safest choice for most real loads ---
# Assumes customers.customer_id has a PRIMARY KEY or UNIQUE constraint
upsert_query = text("""
    INSERT INTO customers (customer_id, name, email)
    VALUES (:customer_id, :name, :email)
    ON CONFLICT (customer_id)
    DO UPDATE SET name = EXCLUDED.name, email = EXCLUDED.email;
""")

with engine.connect() as conn:
    for row in customers_batch.to_dict("records"):
        conn.execute(upsert_query, row)
    conn.commit()   # SQLAlchemy requires an EXPLICIT commit for write operations

print("Upsert complete — safe to re-run this exact script without creating duplicates")

# Confirm: running the upsert AGAIN with the same data changes nothing extra —
# existing rows just get overwritten with identical values, no new rows added
with engine.connect() as conn:
    for row in customers_batch.to_dict("records"):
        conn.execute(upsert_query, row)
    conn.commit()

verify = pd.read_sql("SELECT COUNT(*) as cnt FROM customers WHERE customer_id IN (101, 102, 103);", engine)
print(verify)   # cnt: 3 — NOT 6, confirming the upsert didn't duplicate anything
```

## COMMON MISTAKES

**1. Forgetting `index=False` and getting an unwanted extra column**
```python
df.to_sql("orders", engine)   # writes an extra "index" column (0, 1, 2, ...) — usually not wanted!
```
Fix: make `index=False` a near-automatic habit unless you specifically have a meaningful DataFrame index you want preserved as a column.

**2. Using `if_exists="replace"` on a table other systems also depend on**
```python
orders_df.to_sql("orders", engine, if_exists="replace", index=False)   # DROPS the real "orders" table!
```
`"replace"` doesn't just clear rows — it drops and recreates the entire table, losing any indexes, constraints, or foreign keys defined on it, and breaking anything else that depends on that table's structure while it's being rebuilt. Fix: reserve `"replace"` for genuinely disposable staging/temp tables; use `"append"` (with proper deduplication logic) or explicit `INSERT`/upsert SQL for tables that matter.

**3. Using `if_exists="append"` for a re-runnable pipeline without upsert logic**
As shown in Pattern 2 above, blind appends duplicate data on every rerun — this is the exact anti-pattern the idempotency principle (Lesson 4.8, and your portfolio conventions) explicitly warns against. Fix: for anything beyond a true one-time, append-only load (like a pure event log where duplicates genuinely don't matter), prefer an upsert pattern.

**4. Forgetting SQLAlchemy write operations need an explicit `.commit()`**
```python
with engine.connect() as conn:
    conn.execute(upsert_query, row)
    # forgot conn.commit() — the change might not actually be saved!
```
Fix: always call `conn.commit()` after executing write statements (`INSERT`/`UPDATE`/`DELETE`) through a raw connection — unlike `pd.read_sql()` or `.to_sql()`, which handle this internally, manual `conn.execute()` calls for writes require you to commit explicitly.

**5. Running an upsert loop row-by-row on a very large DataFrame without batching**
```python
for row in huge_df.to_dict("records"):    # 500,000 individual INSERT statements — very slow
    conn.execute(upsert_query, row)
```
Fix: for large volumes, this row-by-row pattern becomes a real performance bottleneck — batch-loading into a staging table with `.to_sql()` and then running a single set-based `INSERT ... ON CONFLICT ...` SQL statement from the staging table into the target table is a much faster pattern at scale, worth knowing about even if this course's examples stay small for teaching clarity.

**Data Engineer Tip:** The staging-table-then-upsert pattern in the worked example — load raw/new data into a disposable staging table first, then merge it into the real table with a single upsert statement — is a genuinely standard production ETL pattern, sometimes formalized as "ELT" (extract, load, then transform/merge inside the database) rather than pure ETL. It shows up constantly in real data engineering work, and it's exactly the shape your Module 9 capstone pipeline will use.

## PRACTICE EXERCISES

*(These exercises assume a running local PostgreSQL instance with the `ecommerce` schema, and a database user with write permissions.)*

**Exercise 1 (Easy):** Create a small DataFrame with `product_id`, `name`, and `price` columns (any sample data), and write it to a new table called `staging_products` using `.to_sql()` with `if_exists="replace"` and `index=False`.

<details>
<summary>Hint</summary>
`df.to_sql("staging_products", engine, if_exists="replace", index=False)`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql://postgres:your_password@localhost:5432/ecommerce")

df = pd.DataFrame({
    "product_id": [9001, 9002],
    "name": ["Test Mouse", "Test Keyboard"],
    "price": [24.99, 59.99],
})

df.to_sql("staging_products", engine, if_exists="replace", index=False)
print("Written successfully")
```
</details>

---

**Exercise 2 (Easy):** Read the `staging_products` table back with `pd.read_sql()` to confirm the write from Exercise 1 worked, and print the result.

<details>
<summary>Hint</summary>
`pd.read_sql("SELECT * FROM staging_products;", engine)`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql://postgres:your_password@localhost:5432/ecommerce")
result = pd.read_sql("SELECT * FROM staging_products;", engine)
print(result)
```
</details>

---

**Exercise 3 (Medium):** Append 2 additional rows to `staging_products` using `if_exists="append"`, then read the table back and confirm it now has 4 rows total.

<details>
<summary>Hint</summary>
Write a new small DataFrame with different `product_id` values, using `if_exists="append"`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql://postgres:your_password@localhost:5432/ecommerce")

more_products = pd.DataFrame({
    "product_id": [9003, 9004],
    "name": ["Test Lamp", "Test Mat"],
    "price": [34.00, 19.99],
})
more_products.to_sql("staging_products", engine, if_exists="append", index=False)

result = pd.read_sql("SELECT COUNT(*) FROM staging_products;", engine)
print(result)   # should show 4
```
</details>

---

**Exercise 4 (Medium):** Explain, in a code comment, what would go wrong if you ran the Exercise 3 append code TWICE in a row without changing the data. Then demonstrate it by actually running it twice and printing the row count both times.

<details>
<summary>Hint</summary>
Appending the same rows twice creates duplicate `product_id` values, since there's no upsert/conflict logic — this is the exact anti-pattern from the lesson.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
from sqlalchemy import create_engine

engine = create_engine("postgresql://postgres:your_password@localhost:5432/ecommerce")

more_products = pd.DataFrame({
    "product_id": [9003, 9004],
    "name": ["Test Lamp", "Test Mat"],
    "price": [34.00, 19.99],
})

# Running this append code twice in a row duplicates product_id 9003 and 9004,
# since there's no primary key constraint or ON CONFLICT handling here — each
# run blindly adds 2 more rows, regardless of whether they already exist.

more_products.to_sql("staging_products", engine, if_exists="append", index=False)
count_1 = pd.read_sql("SELECT COUNT(*) FROM staging_products;", engine)
print("After 1st append:", count_1)

more_products.to_sql("staging_products", engine, if_exists="append", index=False)
count_2 = pd.read_sql("SELECT COUNT(*) FROM staging_products;", engine)
print("After 2nd append:", count_2)
# count_2 will be 2 rows higher than count_1 — confirming the duplication
```
</details>

---

**Exercise 5 (Hard):** Assume a `staging_customers` table (create it with sample data if needed) and a `customers` table with a `PRIMARY KEY` on `customer_id`. Write an upsert function `upsert_customers(engine, df)` that takes a DataFrame of `customer_id`, `name`, `email` and performs an `INSERT ... ON CONFLICT (customer_id) DO UPDATE` for each row, remembering to `commit()`. Test by upserting the same 2 customers twice in a row, confirming via a `SELECT COUNT(*)` that no duplicates were created after the second run.

<details>
<summary>Hint</summary>
Build the parameterized `text()` query once, loop through `df.to_dict("records")`, execute each, and commit once after the loop (more efficient than committing inside the loop).
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
from sqlalchemy import create_engine, text

engine = create_engine("postgresql://postgres:your_password@localhost:5432/ecommerce")

def upsert_customers(engine, df):
    upsert_query = text("""
        INSERT INTO customers (customer_id, name, email)
        VALUES (:customer_id, :name, :email)
        ON CONFLICT (customer_id)
        DO UPDATE SET name = EXCLUDED.name, email = EXCLUDED.email;
    """)
    with engine.connect() as conn:
        for row in df.to_dict("records"):
            conn.execute(upsert_query, row)
        conn.commit()

test_customers = pd.DataFrame({
    "customer_id": [9101, 9102],
    "name": ["Test Customer A", "Test Customer B"],
    "email": ["testA@example.com", "testB@example.com"],
})

upsert_customers(engine, test_customers)
count_1 = pd.read_sql("SELECT COUNT(*) FROM customers WHERE customer_id IN (9101, 9102);", engine)
print("After 1st upsert:", count_1)

upsert_customers(engine, test_customers)   # run again with the SAME data
count_2 = pd.read_sql("SELECT COUNT(*) FROM customers WHERE customer_id IN (9101, 9102);", engine)
print("After 2nd upsert:", count_2)
# Both counts should show 2 — confirming the upsert is safely re-runnable,
# unlike the plain append pattern from Exercise 4
```
</details>

---

**Previous lesson:** [7.2 — SQLAlchemy: Engines & pd.read_sql()](lesson-7.2-sqlalchemy-and-read-sql.md)
**Next lesson:** [7.4 — Parameterized Queries & SQL Injection](lesson-7.4-parameterized-queries.md)
