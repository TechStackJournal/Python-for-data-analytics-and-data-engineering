# Lesson 2.2 — Dictionaries & Nested Structures

## CONCEPT

A **dictionary** (`dict`) stores data as **key-value pairs** — instead of accessing items by numeric position (like a list), you access them by a meaningful label (the key).

```python
product = {
    "id": 1042,
    "name": "Wireless Mouse",
    "price": 24.99,
    "in_stock": True,
}

print(product["name"])   # "Wireless Mouse"
```

Dictionaries can be **nested** — values can themselves be lists, dictionaries, or any combination — which is exactly how real-world structured data (JSON, API responses) is shaped.

## WHY IT MATTERS

Dictionaries are how you'll represent a single record — one row of `products`, one customer, one API response — before it becomes a DataFrame row. Nested dictionaries and lists-of-dictionaries are *the* standard shape of JSON, which you'll encounter constantly: API responses, config files, and semi-structured data like the `ecommerce.events` table (which often stores flexible event properties as JSON). Fluency here is a direct prerequisite for Lesson 2.6 (JSON) and for working with any real API later in the course.

## SYNTAX REFERENCE

```python
# Creating and accessing
product = {"id": 1042, "name": "Mouse", "price": 24.99}
product["name"]                  # "Mouse" — raises KeyError if missing
product.get("name")              # "Mouse" — returns None if missing (safer)
product.get("color", "unknown")  # "unknown" — custom default if missing

# Adding / updating
product["price"] = 22.99         # update existing key
product["color"] = "black"       # add a new key

# Removing
del product["color"]
product.pop("color", None)       # remove safely, no error if missing

# Checking membership
"price" in product               # True — checks KEYS by default

# Iterating
for key in product:
    print(key)

for key, value in product.items():
    print(key, value)

for value in product.values():
    print(value)

# Nested structures
order = {
    "id": 501,
    "customer": {"id": 12, "name": "Maria Chen"},
    "items": [
        {"product": "Mouse", "qty": 2},
        {"product": "Cable", "qty": 1},
    ],
}
order["customer"]["name"]         # "Maria Chen" — chained access
order["items"][0]["product"]      # "Mouse" — list inside a dict inside a dict

# Merging dictionaries
defaults = {"currency": "USD", "tax_rate": 0.08}
overrides = {"tax_rate": 0.0}
merged = {**defaults, **overrides}   # {'currency': 'USD', 'tax_rate': 0.0}

# Dictionary length and keys/values as lists
len(product)
list(product.keys())
list(product.values())
```

## WORKED EXAMPLE

Representing an order with a nested customer and multiple line items — the shape you'd actually see from a real API or JSON export:

```python
order = {
    "order_id": 501,
    "order_date": "2024-11-03",
    "customer": {
        "id": 12,
        "name": "Maria Chen",
        "tier": "gold",
    },
    "items": [
        {"product": "Wireless Mouse", "unit_price": 24.99, "quantity": 2},
        {"product": "USB-C Cable", "unit_price": 12.50, "quantity": 1},
    ],
}

# Accessing nested values
print(order["customer"]["name"])          # "Maria Chen"
print(order["customer"]["tier"])          # "gold"
print(order["items"][0]["product"])       # "Wireless Mouse"

# Computing the order total by iterating the nested list
order_total = 0
for item in order["items"]:
    order_total += item["unit_price"] * item["quantity"]

print(f"Order total: ${order_total:.2f}")  # $62.48

# Using .get() defensively for a field that might not exist
discount_code = order.get("discount_code", "NONE")
print(discount_code)                        # "NONE"

# Building a flat summary dict from the nested structure
# (This is exactly the kind of "flattening" pandas does for you with json_normalize)
summary = {
    "order_id": order["order_id"],
    "customer_name": order["customer"]["name"],
    "customer_tier": order["customer"]["tier"],
    "item_count": len(order["items"]),
    "order_total": round(order_total, 2),
}
print(summary)
# {'order_id': 501, 'customer_name': 'Maria Chen', 'customer_tier': 'gold',
#  'item_count': 2, 'order_total': 62.48}
```

## COMMON MISTAKES

**1. Using `[]` on a key that might not exist**
```python
product = {"id": 1, "name": "Mouse"}
print(product["color"])   # KeyError: 'color'
```
Fix: use `.get("color", default)` when a key might legitimately be missing.

**2. Confusing `.keys()` membership with `.values()` membership**
```python
"Mouse" in product           # checks KEYS — False, "Mouse" is a value, not a key
"Mouse" in product.values()  # True — this checks values
```

**3. Modifying a dictionary while iterating over it**
```python
for key in product:
    if key == "color":
        del product[key]   # RuntimeError: dictionary changed size during iteration
```
Fix: iterate over a copy of the keys: `for key in list(product.keys()):`.

**4. Deep nesting typos causing silent `KeyError`s far from the actual mistake**
```python
order["customer"]["Name"]   # KeyError — the key is "name", lowercase
```
Fix: when exploring unfamiliar nested data, print `.keys()` at each level first to confirm exact key names and casing before chaining access.

**5. Assuming dictionaries are unordered (outdated assumption)**
As of Python 3.7+, dictionaries **do** preserve insertion order — a common point of confusion for anyone who learned Python from older material. This differs from sets, which still don't guarantee order.

**Data Engineer Tip:** The `order` dictionary in the worked example is structurally identical to what a REST API would return as JSON, and to what MongoDB or a "variant" column in a modern warehouse stores. The "flattening" step at the end — pulling nested fields into a flat `summary` dict — is exactly the transformation logic behind `pandas.json_normalize()`, which you'll use directly in Module 4 once you're working with real API responses.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create a dictionary `customer` with keys `id`, `name`, and `email`. Print the `name` value using both `[]` and `.get()`.

<details>
<summary>Hint</summary>
`customer["name"]` and `customer.get("name")`
</details>

<details>
<summary>Solution</summary>

```python
customer = {"id": 1, "name": "Maria Chen", "email": "maria@example.com"}
print(customer["name"])
print(customer.get("name"))
```
</details>

---

**Exercise 2 (Easy):** Given `product = {"id": 1, "name": "Mouse", "price": 24.99}`, safely access a `"color"` key that doesn't exist, defaulting to `"unspecified"`. Print the result.

<details>
<summary>Hint</summary>
`.get("color", "unspecified")`
</details>

<details>
<summary>Solution</summary>

```python
product = {"id": 1, "name": "Mouse", "price": 24.99}
color = product.get("color", "unspecified")
print(color)   # "unspecified"
```
</details>

---

**Exercise 3 (Medium):** Given `order = {"id": 501, "customer": {"name": "Maria Chen", "tier": "gold"}}`, print the customer's name and tier by chaining dictionary access.

<details>
<summary>Hint</summary>
`order["customer"]["name"]`
</details>

<details>
<summary>Solution</summary>

```python
order = {"id": 501, "customer": {"name": "Maria Chen", "tier": "gold"}}
print(order["customer"]["name"])
print(order["customer"]["tier"])
```
</details>

---

**Exercise 4 (Medium):** Given a list of product dictionaries:
```python
products = [
    {"name": "Mouse", "price": 24.99, "category": "Electronics"},
    {"name": "Lamp", "price": 34.00, "category": "Home"},
    {"name": "Keyboard", "price": 59.99, "category": "Electronics"},
]
```
loop through the list and print the name of every product in the `"Electronics"` category.

<details>
<summary>Hint</summary>
Loop through the list, check `product["category"]` inside an `if`.
</details>

<details>
<summary>Solution</summary>

```python
products = [
    {"name": "Mouse", "price": 24.99, "category": "Electronics"},
    {"name": "Lamp", "price": 34.00, "category": "Home"},
    {"name": "Keyboard", "price": 59.99, "category": "Electronics"},
]

for product in products:
    if product["category"] == "Electronics":
        print(product["name"])
```
</details>

---

**Exercise 5 (Hard):** Given this nested order structure:
```python
order = {
    "order_id": 501,
    "customer": {"id": 12, "name": "Maria Chen", "tier": "gold"},
    "items": [
        {"product": "Mouse", "unit_price": 24.99, "quantity": 2},
        {"product": "Cable", "unit_price": 12.50, "quantity": 3},
        {"product": "Lamp", "unit_price": 34.00, "quantity": 1},
    ],
}
```
Write code that: (1) computes the order total from `items`, (2) builds a flat summary dictionary with `order_id`, `customer_name`, `item_count`, and `order_total`, and (3) additionally adds a `"large_order"` key to the summary that is `True` if `order_total > 100`, `False` otherwise. Print the final summary dictionary.

<details>
<summary>Hint</summary>
Loop through `order["items"]` accumulating a total, then build the summary dict, then add the boolean key afterward based on the computed total.
</details>

<details>
<summary>Solution</summary>

```python
order = {
    "order_id": 501,
    "customer": {"id": 12, "name": "Maria Chen", "tier": "gold"},
    "items": [
        {"product": "Mouse", "unit_price": 24.99, "quantity": 2},
        {"product": "Cable", "unit_price": 12.50, "quantity": 3},
        {"product": "Lamp", "unit_price": 34.00, "quantity": 1},
    ],
}

order_total = 0
for item in order["items"]:
    order_total += item["unit_price"] * item["quantity"]

summary = {
    "order_id": order["order_id"],
    "customer_name": order["customer"]["name"],
    "item_count": len(order["items"]),
    "order_total": round(order_total, 2),
}
summary["large_order"] = summary["order_total"] > 100

print(summary)
# {'order_id': 501, 'customer_name': 'Maria Chen', 'item_count': 3,
#  'order_total': 121.48, 'large_order': True}
```
</details>

---

**Previous lesson:** [2.1 — Lists, Tuples & Sets](lesson-2.1-lists-tuples-sets.md)
**Next lesson:** [2.3 — Comprehensions](lesson-2.3-comprehensions.md)
