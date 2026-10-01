# Lesson 2.6 — Reading & Writing JSON

## CONCEPT

**JSON** (JavaScript Object Notation) is a text format for structured data that maps almost directly onto Python's dictionaries and lists — it's the standard format for REST API responses, config files, and semi-structured data. Python's built-in `json` module converts between JSON text and native Python objects.

```python
import json

json_text = '{"id": 1042, "name": "Mouse", "price": 24.99}'
data = json.loads(json_text)     # JSON text -> Python dict
print(data["name"])               # "Mouse"
```

The mapping is intuitive: JSON objects become dicts, JSON arrays become lists, JSON strings/numbers/booleans/null become their direct Python equivalents (`str`/`int`or`float`/`bool`/`None`).

## WHY IT MATTERS

JSON is the dominant format for API data — every time you call a REST API in Module 7, the response comes back as JSON. It's also common for semi-structured event data (the `ecommerce.events` table's `properties` field, for instance, is exactly the kind of flexible, nested data JSON is designed for) and config files. Since JSON objects nest naturally, this lesson builds directly on Lesson 2.2 (nested dictionaries) — JSON *is* nested dictionaries and lists, just serialized as text.

## SYNTAX REFERENCE

```python
import json

# Parsing JSON text into Python objects
json.loads(json_text)              # JSON string -> dict/list
# 'loads' = "load string"

# Reading JSON directly from a file
with open("data.json") as f:
    data = json.load(f)            # note: no 's' — reads directly from a file object

# Converting Python objects into JSON text
json.dumps(data)                    # dict/list -> JSON string
json.dumps(data, indent=2)          # pretty-printed, human-readable
# 'dumps' = "dump string"

# Writing JSON directly to a file
with open("output.json", "w") as f:
    json.dump(data, f, indent=2)    # note: no 's' — writes directly to a file object

# Type mapping (JSON <-> Python)
# JSON object   {}      <->  dict
# JSON array    []      <->  list
# JSON string   "text"  <->  str
# JSON number   42/3.14 <->  int/float
# JSON true/false       <->  bool
# JSON null             <->  None
```

## WORKED EXAMPLE

Parsing a raw JSON API response for an order, and separately writing out cleaned data as JSON:

```python
import json

# Simulating a raw JSON response, as a string (as it would arrive over the network)
raw_response = '''
{
    "order_id": 501,
    "order_date": "2024-11-03",
    "customer": {
        "id": 12,
        "name": "Maria Chen",
        "tier": "gold"
    },
    "items": [
        {"product": "Wireless Mouse", "unit_price": 24.99, "quantity": 2},
        {"product": "USB-C Cable", "unit_price": 12.50, "quantity": 1}
    ],
    "discount_code": null
}
'''

order = json.loads(raw_response)

# Now it's a plain Python dict — everything from Lesson 2.2 applies directly
print(order["customer"]["name"])           # "Maria Chen"
print(order["items"][0]["product"])        # "Wireless Mouse"
print(order["discount_code"])               # None — JSON null maps to Python None

# Compute the order total, same pattern as Lesson 2.2
order_total = sum(item["unit_price"] * item["quantity"] for item in order["items"])
print(f"Order total: ${order_total:.2f}")   # $62.48

# Build a cleaned, flattened summary
summary = {
    "order_id": order["order_id"],
    "customer_name": order["customer"]["name"],
    "customer_tier": order["customer"]["tier"],
    "order_total": round(order_total, 2),
    "has_discount": order["discount_code"] is not None,
}

# Serialize the summary back to JSON text (e.g. for an API response or a log entry)
summary_json = json.dumps(summary, indent=2)
print(summary_json)
# {
#   "order_id": 501,
#   "customer_name": "Maria Chen",
#   "customer_tier": "gold",
#   "order_total": 62.48,
#   "has_discount": false
# }

# Write it to a file
with open("order_summary.json", "w") as f:
    json.dump(summary, f, indent=2)

# Read a file of multiple orders (a JSON array) directly
# Suppose orders.json contains: [{"id": 1, ...}, {"id": 2, ...}]
with open("orders.json") as f:
    all_orders = json.load(f)     # returns a Python list of dicts directly

print(f"Loaded {len(all_orders)} orders")
```

## COMMON MISTAKES

**1. Confusing `loads`/`dumps` (strings) with `load`/`dump` (files)**
```python
with open("data.json") as f:
    data = json.loads(f)   # TypeError — loads() expects a STRING, not a file object
```
Fix: use `json.load(f)` (no "s") when reading directly from an open file; use `json.loads(text)` (with "s") when you already have a JSON string in memory. Same pattern for writing: `json.dump(data, f)` vs. `json.dumps(data)`.

**2. Forgetting JSON's `null` becomes Python's `None`, not the string `"null"`**
```python
data["discount_code"] == "null"    # False! It's the actual None object, not a string
data["discount_code"] is None      # True — this is the correct check
```

**3. Trying to serialize a Python object JSON doesn't understand**
```python
from datetime import datetime
data = {"order_date": datetime(2024, 11, 3)}
json.dumps(data)   # TypeError: Object of type datetime is not JSON serializable
```
Fix: convert non-JSON-native types (like `datetime`) to strings yourself before serializing, e.g. `data["order_date"].strftime("%Y-%m-%d")`, or pass a custom `default=` function to `json.dumps()`.

**4. Assuming JSON keys are always present (same trap as Lesson 2.2)**
```python
order["shipping_address"]   # KeyError if this order didn't include that field
```
Fix: use `.get()` defensively, exactly as with any nested dictionary — JSON from external APIs is especially prone to inconsistent or optional fields.

**5. Malformed JSON text causing a parse error**
```python
json.loads('{"id": 1, "name": "Mouse",}')   # JSONDecodeError — trailing comma not allowed in JSON
```
Fix: JSON's syntax is stricter than Python's dict literal syntax (no trailing commas, keys must be double-quoted strings, no comments). If you're generating JSON text by hand for testing, small syntax slips like this are a very common source of errors — prefer building a Python dict and calling `json.dumps()` rather than hand-typing JSON text.

**Data Engineer Tip:** Every time you call `pd.json_normalize()` or `pd.read_json()` in later modules, it's doing exactly what the worked example's "flatten nested order into summary" step does manually — pulling nested fields out into flat columns. Understanding that transformation by hand here means pandas' JSON-handling functions won't feel like unexplained magic; you'll know exactly what decision they're making about how to flatten each nested structure.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Parse this JSON string into a Python dict and print the `"name"` value: `'{"id": 1, "name": "Mouse", "price": 24.99}'`

<details>
<summary>Hint</summary>
`json.loads(text)`
</details>

<details>
<summary>Solution</summary>

```python
import json

text = '{"id": 1, "name": "Mouse", "price": 24.99}'
data = json.loads(text)
print(data["name"])   # "Mouse"
```
</details>

---

**Exercise 2 (Easy):** Create a Python dict representing a customer (`id`, `name`, `email`), then convert it to a pretty-printed JSON string using `indent=2` and print it.

<details>
<summary>Hint</summary>
`json.dumps(data, indent=2)`
</details>

<details>
<summary>Solution</summary>

```python
import json

customer = {"id": 1, "name": "Maria Chen", "email": "maria@example.com"}
print(json.dumps(customer, indent=2))
```
</details>

---

**Exercise 3 (Medium):** Given this JSON array as a string, parse it and print the `name` of every product with `"in_stock": true`:
```python
raw = '''
[
  {"name": "Mouse", "in_stock": true},
  {"name": "Lamp", "in_stock": false},
  {"name": "Keyboard", "in_stock": true}
]
'''
```

<details>
<summary>Hint</summary>
`json.loads(raw)` returns a list of dicts directly — loop and filter with `if p["in_stock"]`.
</details>

<details>
<summary>Solution</summary>

```python
import json

raw = '''
[
  {"name": "Mouse", "in_stock": true},
  {"name": "Lamp", "in_stock": false},
  {"name": "Keyboard", "in_stock": true}
]
'''

products = json.loads(raw)
for p in products:
    if p["in_stock"]:
        print(p["name"])
```
</details>

---

**Exercise 4 (Medium):** Build a Python dict representing an order with a nested `customer` dict and a list of `items` (at least 2), write it to a file `order.json` with `json.dump()`, then read the file back in with `json.load()` and print the customer's name to confirm it round-tripped correctly.

<details>
<summary>Hint</summary>
Write with `json.dump(data, f, indent=2)` inside a `with open(..., "w") as f:` block, then read with `json.load(f)` inside a separate `with open(...) as f:` block.
</details>

<details>
<summary>Solution</summary>

```python
import json

order = {
    "order_id": 1,
    "customer": {"id": 5, "name": "James Lee"},
    "items": [
        {"product": "Mouse", "price": 24.99},
        {"product": "Cable", "price": 12.50},
    ],
}

with open("order.json", "w") as f:
    json.dump(order, f, indent=2)

with open("order.json") as f:
    loaded_order = json.load(f)

print(loaded_order["customer"]["name"])   # "James Lee"
```
</details>

---

**Exercise 5 (Hard):** Given this raw JSON array of orders (as a string), parse it, and produce a cleaned list of flat summary dicts — one per order — with `order_id`, `customer_name`, and `order_total` (sum of `price * qty` across items), handling the fact that one order is missing its `discount_code` field entirely (use `.get()` to default it to `None`). Print the cleaned list.

```python
raw = '''
[
  {
    "order_id": 1,
    "customer": {"name": "Maria Chen"},
    "items": [{"price": 10, "qty": 2}, {"price": 5, "qty": 1}],
    "discount_code": "SAVE10"
  },
  {
    "order_id": 2,
    "customer": {"name": "James Lee"},
    "items": [{"price": 20, "qty": 1}]
  }
]
'''
```

<details>
<summary>Hint</summary>
Parse the whole array with `json.loads()`, then loop through it building a summary dict per order — use `order.get("discount_code")` (no default needed, defaults to `None` automatically) since the second order is missing that key entirely.
</details>

<details>
<summary>Solution</summary>

```python
import json

raw = '''
[
  {
    "order_id": 1,
    "customer": {"name": "Maria Chen"},
    "items": [{"price": 10, "qty": 2}, {"price": 5, "qty": 1}],
    "discount_code": "SAVE10"
  },
  {
    "order_id": 2,
    "customer": {"name": "James Lee"},
    "items": [{"price": 20, "qty": 1}]
  }
]
'''

orders = json.loads(raw)

summaries = []
for order in orders:
    order_total = sum(item["price"] * item["qty"] for item in order["items"])
    summaries.append({
        "order_id": order["order_id"],
        "customer_name": order["customer"]["name"],
        "order_total": order_total,
        "discount_code": order.get("discount_code"),   # None if missing
    })

for s in summaries:
    print(s)

# {'order_id': 1, 'customer_name': 'Maria Chen', 'order_total': 25, 'discount_code': 'SAVE10'}
# {'order_id': 2, 'customer_name': 'James Lee', 'order_total': 20, 'discount_code': None}
```
</details>

---

**Previous lesson:** [2.5 — Reading & Writing CSV](lesson-2.5-csv.md)
**Next lesson:** [2.7 — File Paths with pathlib](lesson-2.7-pathlib.md)
