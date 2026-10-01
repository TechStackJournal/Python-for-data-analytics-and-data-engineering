# Lesson 7.1 — Connecting to PostgreSQL

## CONCEPT

To run SQL from Python, you need a **database driver** — a library that speaks PostgreSQL's specific network protocol. **`psycopg2`** is the standard, most widely-used PostgreSQL driver for Python. You open a **connection** (an active link to the database), create a **cursor** (an object that executes queries and retrieves results through that connection), and always close both when done.

```python
import psycopg2

conn = psycopg2.connect(host="localhost", dbname="ecommerce", user="postgres", password="...")
cur = conn.cursor()
cur.execute("SELECT * FROM products LIMIT 5;")
rows = cur.fetchall()
cur.close()
conn.close()
```

(`pyodbc` is the equivalent driver commonly used for SQL Server — mentioned here since your Azure roadmap includes SQL Server, but this lesson focuses on `psycopg2` for PostgreSQL, matching the SQL course's database.)

## WHY IT MATTERS

Every pipeline that reads from or writes to a database ultimately goes through a connection like this one — even when you're using a higher-level tool like SQLAlchemy (Lesson 7.2) or pandas' `.read_sql()`, those tools are using a driver like `psycopg2` underneath. Understanding the raw connection/cursor pattern first means you'll actually understand what's happening when a higher-level abstraction fails, instead of treating database connectivity as an opaque black box.

## SYNTAX REFERENCE

```python
import psycopg2

# Establishing a connection
conn = psycopg2.connect(
    host="localhost",
    port=5432,
    dbname="ecommerce",
    user="postgres",
    password="your_password",
)

# Creating a cursor and running a query
cur = conn.cursor()
cur.execute("SELECT product_id, name, price FROM products LIMIT 5;")

# Retrieving results
rows = cur.fetchall()      # every row, as a list of tuples
row = cur.fetchone()         # just the next single row
some_rows = cur.fetchmany(10) # the next 10 rows

# Column names (useful since fetchall() gives plain tuples, no labels)
column_names = [desc[0] for desc in cur.description]

# Always close what you open
cur.close()
conn.close()

# The context-manager pattern (RECOMMENDED — auto-closes even on error)
with psycopg2.connect(host="localhost", dbname="ecommerce", user="postgres", password="...") as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT COUNT(*) FROM orders;")
        result = cur.fetchone()
        print(result)
# both conn and cur are automatically closed when each 'with' block ends

# Checking connection status
conn.closed        # 0 if open, non-zero if closed

# A quick test query to confirm a connection actually works
cur.execute("SELECT version();")
print(cur.fetchone())
```

## WORKED EXAMPLE

Connecting, running a simple query, and properly closing the connection — first the manual way, then with context managers:

```python
import psycopg2

# --- Manual open/close (works, but risks leaking a connection on error) ---
conn = psycopg2.connect(
    host="localhost",
    port=5432,
    dbname="ecommerce",
    user="postgres",
    password="your_password_here",   # Lesson 7.5 replaces this with a safer approach
)

cur = conn.cursor()
cur.execute("SELECT category_id, name FROM categories ORDER BY name;")
rows = cur.fetchall()

for row in rows:
    print(row)
# [(3, 'Books'), (1, 'Electronics'), (2, 'Home'), (4, 'Outdoor')]
# Note: fetchall() returns plain TUPLES, not dicts — position-based, not label-based

cur.close()
conn.close()

# --- The safer, recommended pattern: context managers ---
with psycopg2.connect(
    host="localhost", port=5432, dbname="ecommerce",
    user="postgres", password="your_password_here",
) as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT COUNT(*) FROM orders;")
        order_count = cur.fetchone()[0]     # fetchone() returns a tuple — [0] gets the first value
        print(f"Total orders: {order_count}")

        cur.execute("SELECT product_id, name, price FROM products WHERE price > 100;")
        expensive_products = cur.fetchall()
        print(f"Products over $100: {len(expensive_products)}")

# Both the cursor and connection are automatically closed here, even if
# an error had occurred partway through the queries above
```

## COMMON MISTAKES

**1. Forgetting to close connections and cursors**
```python
conn = psycopg2.connect(...)
cur = conn.cursor()
cur.execute("SELECT ...")
# script continues, never calls cur.close() / conn.close()
```
Fix: use the `with` context-manager pattern shown above — it's the reliable way to guarantee cleanup even when an error occurs partway through, and it should be your default rather than an afterthought.

**2. Hardcoding credentials directly in the script**
```python
conn = psycopg2.connect(host="localhost", password="MyRealPassword123")   # visible in plain text!
```
This is a genuine security risk, especially the moment this script gets committed to a Git repository (even a private one) — the password is now in the commit history permanently, even if you later remove it from the current version of the file. Fix: this exact problem is what Lesson 7.5 solves properly — hold off on real credentials until then, or at minimum keep them out of any file you plan to commit.

**3. Confusing `fetchone()`'s tuple return with a plain scalar value**
```python
cur.execute("SELECT COUNT(*) FROM orders;")
count = cur.fetchone()          # (42,) — a ONE-ELEMENT TUPLE, not the number 42 itself!
print(count + 1)                  # TypeError — can't add an int to a tuple
```
Fix: index into it — `cur.fetchone()[0]` — to get the actual value, as shown in the worked example.

**4. Running `fetchall()` on a query that returns millions of rows**
```python
cur.execute("SELECT * FROM events;")   # a huge table
rows = cur.fetchall()                     # attempts to load EVERYTHING into memory at once!
```
Fix: for genuinely large result sets, either add a `LIMIT` to your query, or use `fetchmany(n)` in a loop to process results in smaller batches rather than pulling everything into memory simultaneously — the same principle behind chunked file reading.

**5. Not handling a connection failure explicitly**
```python
conn = psycopg2.connect(host="wrong-host", ...)   # raises psycopg2.OperationalError
# if unhandled, the whole script crashes with a sometimes-unclear error
```
Fix: wrap connection attempts in a `try`/`except` (Lesson 1.8) specifically catching `psycopg2.OperationalError`, and print/log a clear message about what failed — a wrong host, wrong credentials, or the database simply being unreachable all raise this same exception type, so a clear custom message genuinely helps future debugging.

**Data Engineer Tip:** The context-manager pattern here (`with psycopg2.connect(...) as conn:`) is a direct preview of the resource-management discipline that matters even more once you're writing scheduled pipeline code in Module 8–9 — a connection that's never properly closed in a job that runs every hour will eventually exhaust the database's available connection pool, causing failures that are often initially confusing to diagnose, since the *symptom* (connection refused) shows up far from the *cause* (a forgotten `.close()` several runs earlier).

## PRACTICE EXERCISES

*(These exercises assume you have a local PostgreSQL instance running with the `ecommerce` schema from the SQL course. If you don't have one available, read through each solution to understand the pattern — you'll get hands-on practice with a live database starting in Lesson 7.2's worked examples, and throughout the rest of this module.)*

**Exercise 1 (Easy):** Write code that connects to your local `ecommerce` database, runs `SELECT version();`, prints the result, and closes the connection properly using the manual (non-context-manager) pattern.

<details>
<summary>Hint</summary>
`psycopg2.connect(...)`, then `.cursor()`, `.execute()`, `.fetchone()`, then close both.
</details>

<details>
<summary>Solution</summary>

```python
import psycopg2

conn = psycopg2.connect(host="localhost", dbname="ecommerce", user="postgres", password="your_password")
cur = conn.cursor()
cur.execute("SELECT version();")
result = cur.fetchone()
print(result)
cur.close()
conn.close()
```
</details>

---

**Exercise 2 (Easy):** Rewrite Exercise 1 using the `with` context-manager pattern instead.

<details>
<summary>Hint</summary>
Nest a `with conn.cursor() as cur:` block inside a `with psycopg2.connect(...) as conn:` block.
</details>

<details>
<summary>Solution</summary>

```python
import psycopg2

with psycopg2.connect(host="localhost", dbname="ecommerce", user="postgres", password="your_password") as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT version();")
        result = cur.fetchone()
        print(result)
```
</details>

---

**Exercise 3 (Medium):** Query the `categories` table (`SELECT category_id, name FROM categories;`), fetch all rows, and print each row's `name` value only (not the whole tuple) using a loop.

<details>
<summary>Hint</summary>
Each row is a tuple like `(1, "Electronics")` — index `[1]` for the name.
</details>

<details>
<summary>Solution</summary>

```python
import psycopg2

with psycopg2.connect(host="localhost", dbname="ecommerce", user="postgres", password="your_password") as conn:
    with conn.cursor() as cur:
        cur.execute("SELECT category_id, name FROM categories;")
        rows = cur.fetchall()
        for row in rows:
            print(row[1])
```
</details>

---

**Exercise 4 (Medium):** Write a function `get_order_count(conn)` that takes an open connection, runs `SELECT COUNT(*) FROM orders;`, and returns just the count as a plain integer (not a tuple). Call it and print the result.

<details>
<summary>Hint</summary>
`cur.fetchone()[0]` to extract the value from the one-element tuple.
</details>

<details>
<summary>Solution</summary>

```python
import psycopg2

def get_order_count(conn):
    with conn.cursor() as cur:
        cur.execute("SELECT COUNT(*) FROM orders;")
        return cur.fetchone()[0]

with psycopg2.connect(host="localhost", dbname="ecommerce", user="postgres", password="your_password") as conn:
    count = get_order_count(conn)
    print(f"Total orders: {count}")
```
</details>

---

**Exercise 5 (Hard):** Write a function `safe_connect(host, dbname, user, password)` that attempts to connect to the database, catching `psycopg2.OperationalError` specifically. On success, print `"Connected successfully"` and return the connection object; on failure, print a clear error message including the exception text, and return `None`. Test it once with correct credentials and once with a deliberately wrong `host` (e.g. `"nonexistent-host"`), confirming the function handles both cases without crashing the script.

<details>
<summary>Hint</summary>
Wrap `psycopg2.connect(...)` in a `try`/`except psycopg2.OperationalError as e:` block, matching the pattern from Lesson 1.8.
</details>

<details>
<summary>Solution</summary>

```python
import psycopg2

def safe_connect(host, dbname, user, password):
    try:
        conn = psycopg2.connect(host=host, dbname=dbname, user=user, password=password)
        print("Connected successfully")
        return conn
    except psycopg2.OperationalError as e:
        print(f"Failed to connect: {e}")
        return None

# Successful connection
good_conn = safe_connect("localhost", "ecommerce", "postgres", "your_password")
if good_conn:
    good_conn.close()

# Deliberately broken connection
bad_conn = safe_connect("nonexistent-host", "ecommerce", "postgres", "your_password")
print(bad_conn)   # None — the function returned gracefully instead of crashing
```
</details>

---

**Next lesson:** [7.2 — SQLAlchemy: Engines & pd.read_sql()](lesson-7.2-sqlalchemy-and-read-sql.md)
