# Lesson 7.2 — SQLAlchemy: Engines & `pd.read_sql()`

## CONCEPT

**SQLAlchemy** is a higher-level database toolkit that sits on top of drivers like `psycopg2`, providing a consistent interface across different database engines (PostgreSQL, SQL Server, MySQL, SQLite) so your code doesn't need to change much if the underlying database ever does. Its core object is the **Engine** — a reusable, connection-pooling gateway to the database. Paired with pandas, it makes reading query results directly into a DataFrame a single line of code.

```python
from sqlalchemy import create_engine
import pandas as pd

engine = create_engine("postgresql://user:password@localhost:5432/ecommerce")
df = pd.read_sql("SELECT * FROM products;", engine)
```

## WHY IT MATTERS

Lesson 7.1's raw `psycopg2` pattern returns plain tuples with no column labels — useful to understand the fundamentals, but tedious for real analytics work. `pd.read_sql()` collapses "connect, query, fetch, build a DataFrame with the right column names and types" into one line, and is genuinely how most data analysts and engineers pull query results into pandas day to day. The Engine object is also reused efficiently across many queries, rather than opening and closing a fresh connection every time — an important performance and resource-management detail once you're running many queries in a pipeline.

## SYNTAX REFERENCE

```python
from sqlalchemy import create_engine
import pandas as pd

# Creating an engine — the connection STRING format:
# postgresql://username:password@host:port/database_name
engine = create_engine("postgresql://postgres:mypassword@localhost:5432/ecommerce")

# Reading a query directly into a DataFrame
df = pd.read_sql("SELECT * FROM products;", engine)

# Reading an entire table without writing SQL at all
df = pd.read_sql_table("products", engine)

# Reading with a parameterized query (safe — see Lesson 7.4 for why this matters)
df = pd.read_sql(
    "SELECT * FROM orders WHERE customer_id = %(cust_id)s;",
    engine,
    params={"cust_id": 42},
)

# Explicitly specifying dtypes or parsing dates on read
df = pd.read_sql("SELECT * FROM orders;", engine, parse_dates=["order_date"])

# Using the engine as a context manager for a raw connection (rarely needed
# once you're using pd.read_sql, but useful for non-SELECT statements)
with engine.connect() as conn:
    from sqlalchemy import text
    conn.execute(text("SELECT 1"))

# Disposing of the engine's connection pool when truly done with it
# (usually only relevant at the very end of a long-running script)
engine.dispose()

# A quick way to test an engine actually connects successfully
try:
    with engine.connect() as conn:
        print("Connected successfully")
except Exception as e:
    print(f"Connection failed: {e}")
```

## WORKED EXAMPLE

Reading several different queries from the `ecommerce` database directly into pandas DataFrames:

```python
from sqlalchemy import create_engine
import pandas as pd

engine = create_engine("postgresql://postgres:your_password@localhost:5432/ecommerce")

# 1. Read an entire table
products = pd.read_sql("SELECT * FROM products;", engine)
print(products.head())
print(products.dtypes)   # confirm pandas inferred sensible types automatically

# 2. Read the result of a more complex query — this is where read_sql really shines,
# since arbitrary SQL (joins, aggregations) becomes a DataFrame in one step
category_revenue = pd.read_sql("""
    SELECT
        c.name AS category_name,
        SUM(oi.quantity * oi.unit_price) AS total_revenue
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    JOIN categories c ON p.category_id = c.category_id
    GROUP BY c.name
    ORDER BY total_revenue DESC;
""", engine)
print(category_revenue)

# 3. A parameterized query — reading orders for a SPECIFIC customer safely
customer_orders = pd.read_sql(
    "SELECT * FROM orders WHERE customer_id = %(cust_id)s;",
    engine,
    params={"cust_id": 12},
)
print(f"Orders for customer 12: {len(customer_orders)}")

# 4. Parsing a date column on read, so it arrives as a proper datetime,
# not a plain string (avoiding the manual pd.to_datetime step from Module 5)
orders_with_dates = pd.read_sql(
    "SELECT order_id, order_date, total FROM orders;",
    engine,
    parse_dates=["order_date"],
)
print(orders_with_dates.dtypes)
# order_id                int64
# order_date     datetime64[ns]
# total                 float64

# One engine, reused across all four queries above — no need to reconnect each time
```

## COMMON MISTAKES

**1. Creating a new engine for every single query**
```python
def get_products():
    engine = create_engine("postgresql://...")   # wasteful — reconnects every call
    return pd.read_sql("SELECT * FROM products;", engine)
```
Fix: create the engine **once**, at the top of your script or module, and reuse it across every query — SQLAlchemy's connection pooling is specifically designed for this reuse pattern, and recreating the engine repeatedly defeats its main performance benefit.

**2. Forgetting the connection string format, especially the driver prefix**
```python
create_engine("postgres://...")     # wrong prefix for the default driver — often needs "postgresql://"
```
Fix: the exact required prefix depends on which driver is installed (`postgresql://` for the standard `psycopg2` driver) — if you get an unclear error immediately when creating the engine, this is one of the first things worth checking.

**3. Putting the raw password directly in the connection string, in code you might commit**
```python
engine = create_engine("postgresql://postgres:MyRealPassword123@localhost:5432/ecommerce")
```
Same risk as Lesson 7.1 — Lesson 7.5 shows the proper fix (building this connection string from environment variables instead of a literal string).

**4. Using string formatting/concatenation to build a query with a parameter, instead of `params=`**
```python
customer_id = 42
query = f"SELECT * FROM orders WHERE customer_id = {customer_id};"   # DON'T do this
pd.read_sql(query, engine)
```
This works for a trusted, hardcoded value like this example, but the exact same pattern becomes a serious SQL injection vulnerability the moment `customer_id` comes from user input rather than your own code — covered in full in Lesson 7.4. Fix: build the habit of using `params=` now, even when it feels unnecessary for a safe, hardcoded value.

**5. Assuming `pd.read_sql()` always correctly infers every column's type**
```python
df = pd.read_sql("SELECT order_date FROM orders;", engine)
df.dtypes   # order_date might come back as 'object' (string) rather than datetime, depending on driver/DB
```
Fix: don't assume — check `.dtypes` after reading, and use the explicit `parse_dates=` parameter (shown in the worked example) for any date/timestamp column, rather than relying on automatic inference and finding out later that a date column is secretly still text.

**Data Engineer Tip:** The single-engine, many-queries pattern shown here is exactly how a real pipeline's "extract" step is structured: one Engine object created at the start of a job, reused for every query that job needs to run, and disposed of (or simply allowed to go out of scope) once the job finishes. This is a direct, practical preview of the class-based pipeline structure you'll build formally in Module 8, where a pipeline object holds its database engine as an attribute, created once in `__init__`, rather than a global variable or a repeatedly-recreated connection.

## PRACTICE EXERCISES

*(These exercises assume a running local PostgreSQL instance with the `ecommerce` schema.)*

**Exercise 1 (Easy):** Create a SQLAlchemy engine for your local `ecommerce` database, then use `pd.read_sql()` to load the entire `categories` table into a DataFrame and print it.

<details>
<summary>Hint</summary>
`create_engine("postgresql://...")`, then `pd.read_sql("SELECT * FROM categories;", engine)`.
</details>

<details>
<summary>Solution</summary>

```python
from sqlalchemy import create_engine
import pandas as pd

engine = create_engine("postgresql://postgres:your_password@localhost:5432/ecommerce")
categories = pd.read_sql("SELECT * FROM categories;", engine)
print(categories)
```
</details>

---

**Exercise 2 (Easy):** Using the same engine, read the `products` table with `pd.read_sql_table()` instead of writing a SQL query, and print its `.dtypes`.

<details>
<summary>Hint</summary>
`pd.read_sql_table("products", engine)`
</details>

<details>
<summary>Solution</summary>

```python
from sqlalchemy import create_engine
import pandas as pd

engine = create_engine("postgresql://postgres:your_password@localhost:5432/ecommerce")
products = pd.read_sql_table("products", engine)
print(products.dtypes)
```
</details>

---

**Exercise 3 (Medium):** Write a query that joins `orders` and `customers` to get each order's `order_id`, `total`, and the customer's `name`, and load the result directly into a DataFrame using `pd.read_sql()`. Print the first 5 rows.

<details>
<summary>Hint</summary>
A standard `JOIN` inside the SQL string passed to `pd.read_sql()`.
</details>

<details>
<summary>Solution</summary>

```python
from sqlalchemy import create_engine
import pandas as pd

engine = create_engine("postgresql://postgres:your_password@localhost:5432/ecommerce")

query = """
    SELECT o.order_id, o.total, c.name AS customer_name
    FROM orders o
    JOIN customers c ON o.customer_id = c.customer_id;
"""
df = pd.read_sql(query, engine)
print(df.head())
```
</details>

---

**Exercise 4 (Medium):** Write a parameterized query that reads only orders with a `total` greater than a given threshold, using `params=` rather than string formatting. Test it with a threshold of `100`.

<details>
<summary>Hint</summary>
`pd.read_sql("SELECT * FROM orders WHERE total > %(min_total)s;", engine, params={"min_total": 100})`
</details>

<details>
<summary>Solution</summary>

```python
from sqlalchemy import create_engine
import pandas as pd

engine = create_engine("postgresql://postgres:your_password@localhost:5432/ecommerce")

large_orders = pd.read_sql(
    "SELECT * FROM orders WHERE total > %(min_total)s;",
    engine,
    params={"min_total": 100},
)
print(large_orders)
```
</details>

---

**Exercise 5 (Hard):** Write a function `load_category_summary(engine)` that runs a query joining `order_items`, `products`, and `categories` to compute, per category: total revenue, total quantity sold, and the average unit price of items sold — using SQL aggregation (not pandas `groupby`), and returns the result as a DataFrame sorted by total revenue descending. Then write a second function `load_orders_since(engine, start_date)` that uses a parameterized query to return every order placed on or after a given date, with `order_date` correctly parsed as a datetime column. Call both functions and print their results.

<details>
<summary>Hint</summary>
For the first function, write the aggregation directly in SQL (`SUM()`, `AVG()`, `GROUP BY`, `ORDER BY ... DESC`) and pass it to `pd.read_sql()`. For the second, use `params={"start_date": start_date}` in the `WHERE` clause and `parse_dates=["order_date"]`.
</details>

<details>
<summary>Solution</summary>

```python
from sqlalchemy import create_engine
import pandas as pd

engine = create_engine("postgresql://postgres:your_password@localhost:5432/ecommerce")

def load_category_summary(engine):
    query = """
        SELECT
            c.name AS category_name,
            SUM(oi.quantity * oi.unit_price) AS total_revenue,
            SUM(oi.quantity) AS total_quantity,
            AVG(oi.unit_price) AS avg_unit_price
        FROM order_items oi
        JOIN products p ON oi.product_id = p.product_id
        JOIN categories c ON p.category_id = c.category_id
        GROUP BY c.name
        ORDER BY total_revenue DESC;
    """
    return pd.read_sql(query, engine)


def load_orders_since(engine, start_date):
    query = "SELECT * FROM orders WHERE order_date >= %(start_date)s;"
    return pd.read_sql(query, engine, params={"start_date": start_date}, parse_dates=["order_date"])


summary = load_category_summary(engine)
print(summary)

recent_orders = load_orders_since(engine, "2024-11-01")
print(recent_orders.dtypes)
print(f"Orders since Nov 1: {len(recent_orders)}")
```
</details>

---

**Previous lesson:** [7.1 — Connecting to PostgreSQL](lesson-7.1-connecting-to-postgresql.md)
**Next lesson:** [7.3 — Writing DataFrames Back to a Database](lesson-7.3-writing-to-a-database.md)
