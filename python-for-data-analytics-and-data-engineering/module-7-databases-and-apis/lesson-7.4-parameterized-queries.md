# Lesson 7.4 — Parameterized Queries & SQL Injection Avoidance

## CONCEPT

**SQL injection** is a security vulnerability where untrusted input gets interpreted as part of a SQL query's *structure*, rather than as plain data — letting an attacker (or even an innocent user typing an unusual value) alter what the query actually does, potentially reading or destroying data far beyond what was intended. **Parameterized queries** (also called "prepared statements") prevent this by keeping user-supplied values strictly separated from the SQL command itself, at the driver level.

```python
# VULNERABLE — builds SQL by directly inserting a variable into the string
cur.execute(f"SELECT * FROM customers WHERE email = '{user_input}';")

# SAFE — the driver handles the value separately from the query structure
cur.execute("SELECT * FROM customers WHERE email = %s;", (user_input,))
```

These two versions might *look* like they do the same thing for normal input — the difference only becomes visible (and dangerous) with maliciously or unusually crafted input.

## WHY IT MATTERS

This isn't a theoretical concern — SQL injection has been one of the most common and damaging real-world security vulnerabilities in software for decades, and it's entirely preventable with one consistent habit: never build a SQL query by directly concatenating or formatting untrusted values into the query string. Any pipeline that accepts external input (a filename, a config value, a user-facing search box, an API parameter) and uses it in a query is at risk unless it's parameterized — this lesson makes the risk concrete and the fix automatic.

## SYNTAX REFERENCE

```python
import psycopg2

# psycopg2 placeholder style: %s (regardless of the underlying column's type)
cur.execute("SELECT * FROM orders WHERE customer_id = %s;", (customer_id,))
cur.execute(
    "SELECT * FROM orders WHERE customer_id = %s AND total > %s;",
    (customer_id, min_total),
)

# Named placeholders (often clearer with several parameters)
cur.execute(
    "SELECT * FROM orders WHERE customer_id = %(cust_id)s AND total > %(min_total)s;",
    {"cust_id": customer_id, "min_total": min_total},
)

# The SAME principle via SQLAlchemy's text() + named parameters (Lesson 7.2/7.3 style)
from sqlalchemy import text

query = text("SELECT * FROM orders WHERE customer_id = :cust_id;")
with engine.connect() as conn:
    result = conn.execute(query, {"cust_id": customer_id})

# pandas' read_sql also supports parameters the same safe way (from Lesson 7.2)
pd.read_sql(
    "SELECT * FROM orders WHERE customer_id = %(cust_id)s;",
    engine,
    params={"cust_id": customer_id},
)

# What NOT to do — string formatting/concatenation with untrusted input:
# cur.execute(f"SELECT * FROM orders WHERE customer_id = {customer_id};")     # f-string
# cur.execute("SELECT * FROM orders WHERE customer_id = " + str(customer_id))  # concatenation
# cur.execute("SELECT * FROM orders WHERE customer_id = %s;" % customer_id)     # % formatting
# ALL THREE are vulnerable, regardless of how "safe" the value looks
```

## WORKED EXAMPLE

Demonstrating the actual mechanism of an injection attack against a vulnerable query, then the safe fix — using a deliberately crafted "malicious-looking" input to make the risk concrete:

```python
import psycopg2

conn = psycopg2.connect(host="localhost", dbname="ecommerce", user="postgres", password="your_password")
cur = conn.cursor()

# --- VULNERABLE VERSION ---
def find_customer_vulnerable(email_input):
    query = f"SELECT * FROM customers WHERE email = '{email_input}';"
    print("Actual query sent to the database:", query)
    cur.execute(query)
    return cur.fetchall()

# Normal input works fine and looks harmless
find_customer_vulnerable("maria@example.com")
# Actual query sent: SELECT * FROM customers WHERE email = 'maria@example.com';

# A crafted, malicious input breaks OUT of the intended string entirely
malicious_input = "' OR '1'='1"
find_customer_vulnerable(malicious_input)
# Actual query sent: SELECT * FROM customers WHERE email = '' OR '1'='1';
# '1'='1' is ALWAYS true — this returns EVERY customer row, not just one
# matching email, because the attacker's input closed the quote early and
# added their own condition directly into the query's logic

# An even more damaging crafted input (shown for education — do not run against
# a real database, this would genuinely delete data if executed)
destructive_input = "'; DROP TABLE customers; --"
# Actual query that WOULD be sent:
# SELECT * FROM customers WHERE email = ''; DROP TABLE customers; --';
# The semicolon ends the SELECT, then a SECOND statement drops the entire
# table, and -- comments out the rest of the original query so it doesn't
# cause a syntax error

# --- SAFE VERSION ---
def find_customer_safe(email_input):
    query = "SELECT * FROM customers WHERE email = %s;"
    cur.execute(query, (email_input,))
    return cur.fetchall()

# The exact same malicious input, now handled safely
find_customer_safe(malicious_input)
# The driver sends the LITERAL STRING "' OR '1'='1" as a single parameter
# value to search for — not as part of the query's structure. Since no
# customer actually has that string as their email, this safely returns
# an empty result, exactly as it should for a non-matching search

cur.close()
conn.close()
```

## COMMON MISTAKES

**1. Assuming f-strings are "modern" and therefore automatically safe**
```python
cur.execute(f"SELECT * FROM orders WHERE id = {order_id};")   # f-strings are JUST AS vulnerable as % or +
```
Fix: the *syntax* used to build the string (f-string, `.format()`, `%`, concatenation) is irrelevant — what matters is whether the value ends up embedded directly in the query text at all. If it does, by any method, it's vulnerable.

**2. Believing "this value is safe because it's not directly from a user"**
```python
config_value = load_from_config_file("some_filter.txt")   # not from a live user...
cur.execute(f"SELECT * FROM orders WHERE status = '{config_value}';")   # ...still vulnerable!
```
Fix: any value that isn't a hardcoded literal you wrote directly into the code — a config file, a command-line argument, an API response, another database's data — should be treated as untrusted for this purpose, parameterized regardless of how trustworthy the source *seems*.

**3. Parameterizing values but not table/column names**
```python
table_name = get_table_name_somehow()
cur.execute("SELECT * FROM %s;", (table_name,))   # does NOT work — %s can't parameterize identifiers!
```
Parameterized placeholders work for *values* (things that go where data belongs in the query), not for *identifiers* (table names, column names) — those are part of the query's structure, and databases don't support parameterizing them the same way. Fix: if a table/column name genuinely needs to be dynamic, validate it strictly against a known allow-list of legitimate names in your own code before using it — never accept an arbitrary table name from external input.

**4. Skipping parameterization "just this once" because the query is simple**
Even a query that looks too simple to matter is worth parameterizing consistently — the discipline of *always* parameterizing, with no exceptions for "safe-looking" cases, is what actually prevents the mistake, since a future edit to add a genuinely risky input to an already-vulnerable query is an easy, easy-to-miss regression.

**5. Not realizing `.format()` and string concatenation are equally vulnerable, not just `%`-style formatting**
All four of Python's string-building approaches — f-strings, `.format()`, `%` operator, plain `+` concatenation — insert the value directly into the text before the database ever sees it, so all four share the exact same vulnerability. Only the driver's own placeholder mechanism (`%s`/`:name` passed as a *separate* argument to `.execute()`) is safe, because the value and the query structure stay genuinely separate all the way to the database.

**Data Engineer Tip:** Even in a pipeline where "the input" is just internal config values you wrote yourself, consistently parameterizing every query is a habit worth having automatically — pipelines evolve, and a query that starts out only ever receiving trusted, hardcoded values can easily end up receiving genuinely external input (a value from an API response, a file someone else dropped into a folder) after a future change, at which point an un-parameterized query silently becomes a real vulnerability. Building the habit now, while the stakes feel low, is what makes it automatic later when the stakes are real.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `customer_id = 42`, write a parameterized query using `%s` to select all orders for that customer, and explain in a comment why this is safer than an f-string.

<details>
<summary>Hint</summary>
`cur.execute("SELECT * FROM orders WHERE customer_id = %s;", (customer_id,))`
</details>

<details>
<summary>Solution</summary>

```python
customer_id = 42

# Parameterized — safe, because the value and query structure stay separate
cur.execute("SELECT * FROM orders WHERE customer_id = %s;", (customer_id,))

# vs. an f-string, which is vulnerable because it embeds the value directly
# into the query TEXT itself, where a crafted value could alter the query's
# actual structure rather than just being treated as data to search for
```
</details>

---

**Exercise 2 (Easy):** Rewrite this vulnerable query using proper parameterization: `cur.execute(f"SELECT * FROM products WHERE category = '{category}';")`

<details>
<summary>Hint</summary>
Replace the f-string with `%s` and pass `category` as a tuple argument.
</details>

<details>
<summary>Solution</summary>

```python
category = "Electronics"

cur.execute("SELECT * FROM products WHERE category = %s;", (category,))
```
</details>

---

**Exercise 3 (Medium):** Write a function `search_products_safe(cur, name_fragment)` that safely searches for products whose name contains a given fragment, using SQL's `LIKE` operator with wildcards (hint: `%%` needs escaping in psycopg2's `%s` style, or build the wildcard into the parameter value itself, e.g. `f"%{name_fragment}%"` passed as the PARAMETER, not embedded in the query string). Test it with `"Mouse"`.

<details>
<summary>Hint</summary>
Build the wildcard pattern as a Python string first (`pattern = f"%{name_fragment}%"`), then pass `pattern` as the parameter — the wildcard characters are part of the VALUE, not the query structure, so this stays safe.
</details>

<details>
<summary>Solution</summary>

```python
def search_products_safe(cur, name_fragment):
    pattern = f"%{name_fragment}%"   # the wildcard pattern is built here, as plain data
    cur.execute("SELECT * FROM products WHERE name LIKE %s;", (pattern,))
    return cur.fetchall()

results = search_products_safe(cur, "Mouse")
print(results)
# Safe, because %s still keeps `pattern` as a single parameter value handed
# to the driver separately — the LIKE wildcard characters (%) are part of
# the search VALUE, not injected into the query's SQL structure
```
</details>

---

**Exercise 4 (Medium):** Demonstrate (using a print statement, NOT an actual database call) what the final query string would look like for a vulnerable f-string version of `SELECT * FROM customers WHERE email = '{email}';`, given the malicious input `email = "' OR '1'='1"`. Then explain in 2-3 sentences why this input would return every row in the table instead of zero or one.

<details>
<summary>Hint</summary>
Build the f-string and print it — you'll see the quote gets closed early, and `'1'='1'` is always true.
</details>

<details>
<summary>Solution</summary>

```python
email = "' OR '1'='1"
vulnerable_query = f"SELECT * FROM customers WHERE email = '{email}';"
print(vulnerable_query)
# SELECT * FROM customers WHERE email = '' OR '1'='1';

# Explanation: the malicious input closes the opening quote early (making the
# email comparison check against an empty string), then adds a completely
# separate condition, '1'='1', joined with OR. Since '1' always equals '1',
# this condition is always true for EVERY row in the table — so instead of
# matching a specific customer's email, the WHERE clause effectively becomes
# "true for everyone," and the query returns the entire customers table.
```
</details>

---

**Exercise 5 (Hard):** Write a function `get_orders_in_range(cur, start_date, end_date, min_total=None)` that safely builds a query to fetch orders within a date range, with an OPTIONAL minimum total filter — meaning the query's structure genuinely needs to change (adding or omitting the `AND total >= ...` clause) depending on whether `min_total` is provided, while every actual VALUE stays parameterized regardless. (Hint: it's safe to conditionally include or omit a whole clause in your Python code based on a trusted internal flag like "was `min_total` provided" — the injection risk is specifically about embedding untrusted *values* into the query text, not about your own code's control flow.) Test it once with `min_total=None` and once with `min_total=50`.

<details>
<summary>Hint</summary>
Build the base query and parameter list/dict, then conditionally append `AND total >= %s` (and its corresponding parameter) only if `min_total is not None` — the branching logic itself is fine in Python; just make sure every actual value, once included, goes through a placeholder.
</details>

<details>
<summary>Solution</summary>

```python
def get_orders_in_range(cur, start_date, end_date, min_total=None):
    query = "SELECT * FROM orders WHERE order_date BETWEEN %s AND %s"
    params = [start_date, end_date]

    if min_total is not None:
        query += " AND total >= %s"
        params.append(min_total)

    query += ";"

    cur.execute(query, tuple(params))
    return cur.fetchall()

# Without a minimum total
all_in_range = get_orders_in_range(cur, "2024-01-01", "2024-12-31")
print(f"Orders in range: {len(all_in_range)}")

# With a minimum total
large_in_range = get_orders_in_range(cur, "2024-01-01", "2024-12-31", min_total=50)
print(f"Orders in range with total >= 50: {len(large_in_range)}")

# This is safe because:
# 1. start_date, end_date, and min_total are ALWAYS passed as parameters,
#    never embedded directly in the query string
# 2. The only thing that changes about the QUERY TEXT itself (whether the
#    "AND total >= %s" clause is present) is controlled entirely by our own
#    trusted Python code (checking `min_total is not None`), not by any
#    value that came from outside the program
```
</details>

---

**Previous lesson:** [7.3 — Writing DataFrames Back to a Database](lesson-7.3-writing-to-a-database.md)
**Next lesson:** [7.5 — Managing Credentials with python-dotenv](lesson-7.5-credentials-and-dotenv.md)
