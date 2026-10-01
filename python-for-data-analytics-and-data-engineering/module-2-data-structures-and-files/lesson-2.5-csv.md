# Lesson 2.5 — Reading & Writing CSV

## CONCEPT

**CSV** (comma-separated values) is the most common plain-text format for tabular data — each line is a row, and commas (or another delimiter) separate the columns. Python's built-in `csv` module reads and writes these files without needing any external library, handling tricky edge cases (commas inside quoted fields, different line endings) that manual string-splitting gets wrong.

```python
import csv

with open("orders.csv") as f:
    reader = csv.reader(f)
    for row in reader:
        print(row)   # each row is a plain list of strings
```

The `csv.DictReader`/`csv.DictWriter` variants work the same way but use the header row as dictionary keys, giving you list-of-dicts instead of list-of-lists — usually the more useful shape.

## WHY IT MATTERS

CSV is everywhere in data analytics: database exports, spreadsheet downloads, API bulk-export endpoints, and the most common format you'll hand off to (or receive from) a non-technical stakeholder. Before you get to `pandas.read_csv()` in Module 4 — which does all of this automatically — understanding what's actually happening underneath (opening a file, reading rows, handling headers, closing the file) means you'll actually understand pandas' behavior and its error messages, instead of treating file I/O as magic.

## SYNTAX REFERENCE

```python
import csv

# Reading rows as plain lists
with open("products.csv", newline="") as f:
    reader = csv.reader(f)
    header = next(reader)          # first row, if there is a header
    for row in reader:
        print(row)                 # ['1042', 'Wireless Mouse', '24.99']

# Reading rows as dictionaries (uses the header row as keys)
with open("products.csv", newline="") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row)                 # {'id': '1042', 'name': 'Wireless Mouse', 'price': '24.99'}

# Writing rows from lists
with open("output.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["id", "name", "price"])         # header
    writer.writerow([1042, "Wireless Mouse", 24.99])
    writer.writerows([                                 # multiple rows at once
        [1043, "Keyboard", 59.99],
        [1044, "Monitor", 199.00],
    ])

# Writing rows from dictionaries
fieldnames = ["id", "name", "price"]
with open("output.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerow({"id": 1042, "name": "Wireless Mouse", "price": 24.99})

# Custom delimiter (e.g. tab-separated files)
csv.reader(f, delimiter="\t")
```

**Note the `with open(...) as f:` pattern** — this is a **context manager**. It automatically closes the file when the block ends, even if an error occurs inside it. Always prefer this over manually calling `f.close()`.

## WORKED EXAMPLE

Reading a raw products CSV, cleaning the data (reusing string/type-conversion skills from Module 1), and writing out a cleaned version:

Suppose `raw_products.csv` contains:
```
id,name,price,in_stock
1042,  Wireless Mouse ,$24.99,true
1043,USB-C Cable,$12.50,false
1044,Desk Lamp,$34.00,TRUE
```

```python
import csv

cleaned_products = []

with open("raw_products.csv", newline="") as f:
    reader = csv.DictReader(f)
    for row in reader:
        cleaned_products.append({
            "id": int(row["id"]),
            "name": row["name"].strip(),
            "price": float(row["price"].replace("$", "")),
            "in_stock": row["in_stock"].strip().lower() == "true",
        })

for p in cleaned_products:
    print(p)

# {'id': 1042, 'name': 'Wireless Mouse', 'price': 24.99, 'in_stock': True}
# {'id': 1043, 'name': 'USB-C Cable', 'price': 12.5, 'in_stock': False}
# {'id': 1044, 'name': 'Desk Lamp', 'price': 34.0, 'in_stock': True}

# Now write the cleaned data back out
fieldnames = ["id", "name", "price", "in_stock"]
with open("cleaned_products.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(cleaned_products)

# Filter: write only in-stock products to a separate file
in_stock_only = [p for p in cleaned_products if p["in_stock"]]
with open("in_stock_products.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=fieldnames)
    writer.writeheader()
    writer.writerows(in_stock_only)

print(f"Wrote {len(in_stock_only)} in-stock products.")   # Wrote 2 in-stock products.
```

## COMMON MISTAKES

**1. Forgetting `newline=""` when opening files for CSV**
```python
with open("products.csv") as f:   # missing newline="" — can cause extra blank rows on Windows
```
Fix: always include `newline=""` when opening a file for `csv.reader`/`csv.writer` — this is specifically required by the `csv` module's documentation to handle line endings correctly across operating systems.

**2. Assuming every value from `csv.reader`/`DictReader` is already the right type**
```python
row["price"]        # "24.99" — a STRING, even though it looks numeric
row["price"] * 2     # "24.9924.99" — string repetition, not math!
```
Fix: this is the exact Lesson 1.1 lesson again — every field from a CSV starts life as a string and needs explicit conversion.

**3. Overwriting a file accidentally by opening in `"w"` mode**
```python
with open("important_data.csv", "w") as f:   # "w" TRUNCATES the file immediately, even before writing anything!
```
Fix: `"w"` mode erases the existing file content the instant it's opened. Use `"a"` (append) if you want to add to an existing file instead, and always double-check the mode before running write code on a file you care about.

**4. Mismatched `fieldnames` when using `DictWriter`**
```python
writer = csv.DictWriter(f, fieldnames=["id", "name"])
writer.writerow({"id": 1, "name": "Mouse", "price": 24.99})   # ValueError — 'price' not in fieldnames
```
Fix: `fieldnames` must include every key present in the dictionaries you're writing — extra keys not listed will raise an error by default.

**5. Not handling a header row when using plain `csv.reader`**
```python
with open("products.csv") as f:
    reader = csv.reader(f)
    for row in reader:
        price = float(row[2])   # crashes on the header row! "price" isn't a valid float
```
Fix: either call `next(reader)` once before the loop to skip the header, or use `csv.DictReader`, which handles this automatically.

**Data Engineer Tip:** Manually writing CSV cleaning logic like the worked example above is exactly what you'll replace with `pandas.read_csv(dtype=..., converters=...)` in Module 4 — but doing it manually first means when pandas silently mis-parses a column (a very common real bug), you'll know exactly what's happening under the hood and how to fix it, rather than being stuck.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Write a CSV file `sample.csv` with a header row `id,name,price` and two data rows of your choosing, using `csv.writer`.

<details>
<summary>Hint</summary>
`writer.writerow()` for the header, then again for each row, or `writer.writerows()` for multiple rows at once.
</details>

<details>
<summary>Solution</summary>

```python
import csv

with open("sample.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["id", "name", "price"])
    writer.writerows([
        [1, "Mouse", 24.99],
        [2, "Keyboard", 59.99],
    ])
```
</details>

---

**Exercise 2 (Easy):** Read `sample.csv` (from Exercise 1) back in using `csv.DictReader` and print each row as a dictionary.

<details>
<summary>Hint</summary>
`csv.DictReader(f)`, then loop and print each row.
</details>

<details>
<summary>Solution</summary>

```python
import csv

with open("sample.csv", newline="") as f:
    reader = csv.DictReader(f)
    for row in reader:
        print(row)
```
</details>

---

**Exercise 3 (Medium):** Given `sample.csv` from before, read it in, convert the `"price"` field to a float for each row, and print the sum of all prices.

<details>
<summary>Hint</summary>
Accumulate a total inside the loop, converting `row["price"]` with `float()` each time.
</details>

<details>
<summary>Solution</summary>

```python
import csv

total = 0
with open("sample.csv", newline="") as f:
    reader = csv.DictReader(f)
    for row in reader:
        total += float(row["price"])

print(f"Total: ${total:.2f}")   # Total: $84.98
```
</details>

---

**Exercise 4 (Medium):** Given a raw CSV with messy values — write `messy.csv` with a header `id,name,in_stock` and rows `1,  Mouse ,TRUE` and `2,Keyboard,false` — read it in, clean the `name` (strip whitespace) and `in_stock` (proper boolean) fields, and print the cleaned list of dicts.

<details>
<summary>Hint</summary>
First write the messy file with `csv.writer`, then read it back with `DictReader` and apply `.strip()` and a lowercase comparison for the boolean.
</details>

<details>
<summary>Solution</summary>

```python
import csv

# Create the messy file first
with open("messy.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["id", "name", "in_stock"])
    writer.writerow([1, "  Mouse ", "TRUE"])
    writer.writerow([2, "Keyboard", "false"])

# Read and clean it
cleaned = []
with open("messy.csv", newline="") as f:
    reader = csv.DictReader(f)
    for row in reader:
        cleaned.append({
            "id": int(row["id"]),
            "name": row["name"].strip(),
            "in_stock": row["in_stock"].strip().lower() == "true",
        })

print(cleaned)
# [{'id': 1, 'name': 'Mouse', 'in_stock': True}, {'id': 2, 'name': 'Keyboard', 'in_stock': False}]
```
</details>

---

**Exercise 5 (Hard):** Given a CSV of orders — write `orders.csv` with header `order_id,customer,total,status` and rows:
```
1,Maria Chen,45.99,shipped
2,James Lee,120.50,pending
3,Priya Patel,89.99,shipped
4,Maria Chen,15.00,cancelled
```
Read it in, and write out **two separate CSV files**: `shipped_orders.csv` containing only rows with `status == "shipped"`, and `order_summary.csv` containing one row per unique customer with their total spend across only their shipped orders (columns: `customer,total_spent`).

<details>
<summary>Hint</summary>
Read all rows into a list of dicts first, converting `total` to float. Filter for `shipped_orders.csv`. For the summary, use a dictionary keyed by customer name to accumulate totals, then write that dictionary out as rows.
</details>

<details>
<summary>Solution</summary>

```python
import csv

# Create the source file
with open("orders.csv", "w", newline="") as f:
    writer = csv.writer(f)
    writer.writerow(["order_id", "customer", "total", "status"])
    writer.writerows([
        [1, "Maria Chen", 45.99, "shipped"],
        [2, "James Lee", 120.50, "pending"],
        [3, "Priya Patel", 89.99, "shipped"],
        [4, "Maria Chen", 15.00, "cancelled"],
    ])

# Read and process
orders = []
with open("orders.csv", newline="") as f:
    reader = csv.DictReader(f)
    for row in reader:
        orders.append({
            "order_id": int(row["order_id"]),
            "customer": row["customer"],
            "total": float(row["total"]),
            "status": row["status"],
        })

# 1. Write shipped-only orders
shipped = [o for o in orders if o["status"] == "shipped"]
with open("shipped_orders.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["order_id", "customer", "total", "status"])
    writer.writeheader()
    writer.writerows(shipped)

# 2. Build per-customer totals from shipped orders only
customer_totals = {}
for order in shipped:
    customer = order["customer"]
    customer_totals[customer] = customer_totals.get(customer, 0) + order["total"]

with open("order_summary.csv", "w", newline="") as f:
    writer = csv.DictWriter(f, fieldnames=["customer", "total_spent"])
    writer.writeheader()
    for customer, total in customer_totals.items():
        writer.writerow({"customer": customer, "total_spent": round(total, 2)})

print(customer_totals)   # {'Maria Chen': 45.99, 'Priya Patel': 89.99}
```
</details>

---

**Previous lesson:** [2.4 — Dates & Times](lesson-2.4-datetime.md)
**Next lesson:** [2.6 — Reading & Writing JSON](lesson-2.6-json.md)
