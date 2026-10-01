# Lesson 8.2 — Structuring a Project: Modules & Packages

## CONCEPT

A **module** is simply a single `.py` file — anything you can `import`. A **package** is a folder of related modules, marked as importable by (in older Python, required; in modern Python, now optional but still a very common convention) an `__init__.py` file inside it. Together, these let you split a growing project into organized, importable pieces instead of one enormous script.

```python
# my_project/cleaning.py
def clean_price(raw_price):
    return float(str(raw_price).replace("$", ""))

# my_project/main.py
from cleaning import clean_price      # importing a function from another file

print(clean_price("$24.99"))   # 24.99
```

A typical small data pipeline project grows from "one script" into a handful of purposeful modules — a natural, standard shape once you see a few examples.

## WHY IT MATTERS

Every lesson's worked examples so far have lived in one self-contained script — fine for learning, but unrealistic for anything you'd actually ship or hand to a teammate. A single 800-line script mixing database connections, cleaning logic, and a command-line entry point is hard to navigate, hard to test in isolation (Lesson 8.6), and hard for anyone but its author to safely modify. Splitting code into modules by *responsibility* — one file for extraction, one for transformation, one for loading — is a direct precursor to the ETL structure you'll build in Module 9's capstone.

## SYNTAX REFERENCE

```python
# A typical small project layout:
#
# ecommerce_pipeline/
# ├── pyproject.toml          (Lesson 8.3)
# ├── .env                     (Lesson 8.8, never committed to Git)
# ├── src/
# │   └── ecommerce_pipeline/
# │       ├── __init__.py
# │       ├── extract.py
# │       ├── transform.py
# │       ├── load.py
# │       └── config.py
# └── tests/
#     ├── test_transform.py
#     └── test_load.py

# Importing from a sibling module in the same package
# (inside transform.py)
from ecommerce_pipeline.extract import fetch_raw_orders

# Importing everything under a package name (less common, generally avoid
# `from module import *` — it makes it unclear where a name came from)
import ecommerce_pipeline.transform as transform
transform.clean_price(raw_value)

# __init__.py can be empty, or can expose specific names at the package level
# (inside ecommerce_pipeline/__init__.py)
from ecommerce_pipeline.transform import clean_order
# now other code can do: from ecommerce_pipeline import clean_order

# Relative imports WITHIN a package (import from a sibling file, using a leading dot)
# (inside ecommerce_pipeline/transform.py)
from .extract import fetch_raw_orders      # . means "this same package"

# The "main script" entry-point pattern — code that should only run when this
# specific file is executed directly, not when it's imported by something else
def main():
    orders = fetch_raw_orders()
    print(f"Fetched {len(orders)} orders")

if __name__ == "__main__":
    main()
```

## WORKED EXAMPLE

Splitting a single monolithic script into a properly structured mini-package — a small ETL project with three clearly separated responsibilities:

```python
# --- BEFORE: everything in one file, order_pipeline.py ---
#
# def fetch_raw_orders():
#     return [{"id": "1", "total": "45.99"}, {"id": "2", "total": "89.00"}]
#
# def clean_order(raw):
#     return {"id": int(raw["id"]), "total": float(raw["total"])}
#
# def save_orders(orders):
#     for order in orders:
#         print(f"Saving order {order['id']}: ${order['total']}")
#
# raw = fetch_raw_orders()
# cleaned = [clean_order(r) for r in raw]
# save_orders(cleaned)
#
# This works, but mixes extraction, transformation, loading, and the
# "run it" logic all in one undifferentiated file — hard to test any one
# piece independently, and hard to find anything as the file grows.


# --- AFTER: split into a small package, ecommerce_pipeline/ ---

# ecommerce_pipeline/extract.py
def fetch_raw_orders():
    """Simulates pulling raw order data from a source system."""
    return [{"id": "1", "total": "45.99"}, {"id": "2", "total": "89.00"}]


# ecommerce_pipeline/transform.py
def clean_order(raw_order):
    """Convert a raw order dict into properly typed values."""
    return {
        "id": int(raw_order["id"]),
        "total": float(raw_order["total"]),
    }

def clean_batch(raw_orders):
    """Apply clean_order() to a whole batch."""
    return [clean_order(order) for order in raw_orders]


# ecommerce_pipeline/load.py
def save_orders(orders):
    """Simulates writing cleaned orders to a destination."""
    for order in orders:
        print(f"Saving order {order['id']}: ${order['total']}")


# ecommerce_pipeline/main.py — the entry point that wires everything together
from ecommerce_pipeline.extract import fetch_raw_orders
from ecommerce_pipeline.transform import clean_batch
from ecommerce_pipeline.load import save_orders

def run_pipeline():
    raw_orders = fetch_raw_orders()
    cleaned_orders = clean_batch(raw_orders)
    save_orders(cleaned_orders)
    print(f"Pipeline complete: {len(cleaned_orders)} orders processed")

if __name__ == "__main__":
    run_pipeline()

# Now extract.py, transform.py, and load.py can each be tested, read, and
# modified independently — and transform.py's clean_order() function, in
# particular, is now trivially unit-testable in isolation (Lesson 8.6),
# since it no longer depends on the other two steps having run first
```

## COMMON MISTAKES

**1. Circular imports — two modules trying to import from each other**
```python
# extract.py
from ecommerce_pipeline.transform import clean_order   # transform imports FROM extract too!

# transform.py
from ecommerce_pipeline.extract import fetch_raw_orders   # ImportError: circular dependency
```
Fix: this usually signals the responsibilities aren't cleanly separated — a function needed by both should often move to a third, shared module (or a genuinely lower-level utility module) that both of the others can depend on, rather than depending on each other.

**2. Forgetting `__init__.py` when using older Python tooling or specific import styles**
Modern Python (3.3+) supports "namespace packages" without a strictly required `__init__.py`, but many real-world tools and conventions still expect one, and an empty `__init__.py` costs nothing. Fix: as a safe default for this course, include an (even empty) `__init__.py` in every package folder.

**3. Using `from module import *`**
```python
from ecommerce_pipeline.transform import *    # imports EVERYTHING, unclear what came from where
clean_order(raw)   # where did this come from? Not obvious from reading this line alone
```
Fix: import specific names explicitly (`from ecommerce_pipeline.transform import clean_order`), or import the whole module under a clear alias (`import ecommerce_pipeline.transform as transform`, then `transform.clean_order(...)`) — both keep it obvious where a function came from when reading the code later.

**4. Forgetting the `if __name__ == "__main__":` guard, and having "run" code execute on import**
```python
# main.py, without the guard
raw_orders = fetch_raw_orders()
print(f"Fetched {len(raw_orders)} orders")

# elsewhere:
import main   # this ACCIDENTALLY runs the whole pipeline just by importing the file!
```
Fix: wrap code that should only run when the file is executed directly (not imported) inside `if __name__ == "__main__":` — this is what lets `main.py` be safely imported for testing without triggering a full pipeline run.

**5. Putting business logic directly at the top level of a module, outside any function**
```python
# transform.py
raw_data = fetch_raw_orders()   # runs IMMEDIATELY the instant this module is imported, anywhere!
cleaned = [clean_order(r) for r in raw_data]
```
Fix: code that does real work (not just defining functions/classes/constants) belongs inside a function, typically called from `main()` — a module being merely *imported* shouldn't have side effects like fetching data or writing files.

**Data Engineer Tip:** The `extract.py` / `transform.py` / `load.py` split in the worked example isn't a coincidence — it directly mirrors the ETL (Extract, Transform, Load) terminology from your Azure roadmap, and this exact folder shape is close to what Module 9's capstone project will use. Separating these responsibilities into different files isn't just tidiness — it means you can, for example, swap out `extract.py` entirely (switching a data source from CSV to an API) without touching `transform.py` or `load.py` at all, as long as the interface between them (what a function takes in and returns) stays the same.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create two files in the same folder: `greetings.py` containing a function `say_hello(name)` that returns `f"Hello, {name}!"`, and `main.py` that imports and calls it, printing the result for `"Maria"`.

<details>
<summary>Hint</summary>
`from greetings import say_hello` in `main.py` (both files in the same folder, no package structure needed for this simple case).
</details>

<details>
<summary>Solution</summary>

```python
# greetings.py
def say_hello(name):
    return f"Hello, {name}!"
```
```python
# main.py
from greetings import say_hello

print(say_hello("Maria"))
```
Run with `python3 main.py` from the same folder as both files.
</details>

---

**Exercise 2 (Easy):** Add an `if __name__ == "__main__":` guard to `main.py` from Exercise 1, wrapping the `print()` call inside a function `main()` that gets called only when the script runs directly.

<details>
<summary>Hint</summary>
Define `def main(): print(say_hello("Maria"))`, then call it only inside the guard.
</details>

<details>
<summary>Solution</summary>

```python
# main.py
from greetings import say_hello

def main():
    print(say_hello("Maria"))

if __name__ == "__main__":
    main()
```
</details>

---

**Exercise 3 (Medium):** Create a small package folder `mathutils/` containing an `__init__.py` (can be empty) and a file `operations.py` with two functions: `add(a, b)` and `multiply(a, b)`. From a separate script in the parent folder, import and use both functions.

<details>
<summary>Hint</summary>
Folder structure: `mathutils/__init__.py`, `mathutils/operations.py`, and a `main.py` in the parent folder using `from mathutils.operations import add, multiply`.
</details>

<details>
<summary>Solution</summary>

```python
# mathutils/__init__.py
# (empty file)
```
```python
# mathutils/operations.py
def add(a, b):
    return a + b

def multiply(a, b):
    return a * b
```
```python
# main.py (in the parent folder, alongside the mathutils/ directory)
from mathutils.operations import add, multiply

print(add(3, 4))         # 7
print(multiply(3, 4))     # 12
```
</details>

---

**Exercise 4 (Medium):** Given a single monolithic script that reads, cleans, and prints product data, split it into two modules — `cleaning.py` (containing a `clean_product(raw)` function) and `main.py` (which imports `clean_product` and runs it against a sample list, with a proper `if __name__ == "__main__":` guard). Starting point:
```python
# monolith.py
products = [{"name": " Mouse ", "price": "$24.99"}, {"name": "Lamp", "price": "$34.00"}]
for p in products:
    cleaned_name = p["name"].strip()
    cleaned_price = float(p["price"].replace("$", ""))
    print(f"{cleaned_name}: ${cleaned_price}")
```

<details>
<summary>Hint</summary>
Move the per-product cleaning logic into a function `clean_product(raw)` returning a dict, in `cleaning.py`. `main.py` imports it, loops over the sample list, and prints the results — guarded by `if __name__ == "__main__":`.
</details>

<details>
<summary>Solution</summary>

```python
# cleaning.py
def clean_product(raw):
    return {
        "name": raw["name"].strip(),
        "price": float(raw["price"].replace("$", "")),
    }
```
```python
# main.py
from cleaning import clean_product

def main():
    products = [{"name": " Mouse ", "price": "$24.99"}, {"name": "Lamp", "price": "$34.00"}]
    for raw in products:
        cleaned = clean_product(raw)
        print(f"{cleaned['name']}: ${cleaned['price']}")

if __name__ == "__main__":
    main()
```
</details>

---

**Exercise 5 (Hard):** Design (describe in comments, plus create the actual empty files with `__init__.py`s where needed) a package structure for a small pipeline that: extracts customer data from a CSV, cleans/validates it, computes a few summary statistics, and writes a report to a text file. Decide on sensible module names and responsibilities, create the actual folder/file structure on disk (empty function stubs are fine — the goal is the *structure*, not full implementations), and write a `main.py` that imports from each module and would (conceptually) wire them together, including the `if __name__ == "__main__":` guard.

<details>
<summary>Hint</summary>
A reasonable split: `extract.py` (reading the CSV), `transform.py` (cleaning/validating), `analyze.py` (summary statistics), `report.py` (writing the text file), all inside a package folder like `customer_pipeline/`, with `main.py` alongside it (not inside it) tying them together.
</details>

<details>
<summary>Solution</summary>

```
customer_pipeline/
├── __init__.py
├── extract.py
├── transform.py
├── analyze.py
└── report.py
main.py
```

```python
# customer_pipeline/__init__.py
# (empty)
```
```python
# customer_pipeline/extract.py
def read_customers_csv(filepath):
    """Read raw customer rows from a CSV file. Stub for this exercise."""
    pass
```
```python
# customer_pipeline/transform.py
def clean_customer(raw_row):
    """Validate and clean a single raw customer row. Stub for this exercise."""
    pass
```
```python
# customer_pipeline/analyze.py
def compute_summary_stats(customers):
    """Compute summary statistics across cleaned customer records. Stub for this exercise."""
    pass
```
```python
# customer_pipeline/report.py
def write_report(stats, output_path):
    """Write a text report of the computed statistics. Stub for this exercise."""
    pass
```
```python
# main.py
from customer_pipeline.extract import read_customers_csv
from customer_pipeline.transform import clean_customer
from customer_pipeline.analyze import compute_summary_stats
from customer_pipeline.report import write_report

def main():
    raw_customers = read_customers_csv("customers.csv")
    cleaned = [clean_customer(row) for row in raw_customers]
    stats = compute_summary_stats(cleaned)
    write_report(stats, "report.txt")
    print("Pipeline complete.")

if __name__ == "__main__":
    main()
```
Each module has one clear responsibility, `main.py` sits outside the package and wires the pieces together, and every function in `customer_pipeline/` could now be imported and unit-tested independently (Lesson 8.6) without needing the rest of the pipeline to run first.
</details>

---

**Previous lesson:** [8.1 — From Functions to Classes: Intro to OOP](lesson-8.1-oop-basics.md)
**Next lesson:** [8.3 — Dependency Management with Poetry](lesson-8.3-poetry.md)
