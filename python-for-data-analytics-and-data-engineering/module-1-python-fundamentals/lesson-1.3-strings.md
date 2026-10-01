# Lesson 1.3 — Strings & String Methods

## CONCEPT

A **string** (`str`) is a sequence of characters used to represent text. Strings can be created with single or double quotes, and Python treats them identically:

```python
name = "Maria Chen"
name = 'Maria Chen'   # same thing
```

Strings are **immutable** — once created, you can't change individual characters in place. Every "modification" actually creates a brand-new string. Strings also come with a large set of built-in **methods** (functions attached to the string itself) for cleaning, searching, and reformatting text.

## WHY IT MATTERS

Real-world data is messy text: inconsistent capitalization (`"electronics"` vs `"Electronics"` vs `"ELECTRONICS"`), stray whitespace (`" Maria Chen "`), embedded symbols (`"$24.99"`), and inconsistent delimiters. String cleaning is arguably the single most common task in data analytics — before you can group, filter, or join anything, the text has to be standardized. Everything you learn here maps directly onto pandas' `.str` accessor methods in Module 4.

## SYNTAX REFERENCE

```python
# Basics
len(text)                   # length of the string
text[0]                     # first character (indexing starts at 0)
text[-1]                    # last character
text[0:5]                   # slicing: characters 0 through 4

# Case
text.lower()
text.upper()
text.title()                # Capitalizes Each Word

# Whitespace
text.strip()                # removes leading/trailing whitespace
text.lstrip()  / text.rstrip()

# Searching
text.startswith("A")
text.endswith(".csv")
"mouse" in text              # membership check
text.find("mouse")           # index of substring, or -1 if not found

# Replacing
text.replace("old", "new")

# Splitting and joining
text.split(",")               # string -> list
",".join(["a", "b", "c"])     # list -> string

# Formatting
f"{name} bought {qty} items"      # f-string (preferred, modern)
"{} bought {} items".format(name, qty)   # .format() (older style)

# Type checks
text.isdigit()
text.isalpha()
```

## WORKED EXAMPLE

Cleaning a messy raw product name and category, as they might arrive straight from a CSV export:

```python
raw_name = "  Wireless MOUSE  "
raw_category = "electronics"
raw_price_text = "$24.99"
raw_sku = "SKU-1042-ELEC"

# Clean the name: strip whitespace, standardize casing
clean_name = raw_name.strip().title()
print(clean_name)          # "Wireless Mouse"

# Standardize category for consistent grouping
clean_category = raw_category.strip().title()
print(clean_category)      # "Electronics"

# Strip currency symbol before converting to a number
price_number = float(raw_price_text.replace("$", ""))
print(price_number)        # 24.99

# Parse a structured SKU string into parts
parts = raw_sku.split("-")
print(parts)                          # ['SKU', '1042', 'ELEC']
product_id = int(parts[1])
category_code = parts[2]
print(product_id, category_code)      # 1042 ELEC

# Build a readable summary using an f-string
summary = f"{clean_name} ({clean_category}) — ${price_number:.2f}"
print(summary)   # Wireless Mouse (Electronics) — $24.99
```

## COMMON MISTAKES

**1. Forgetting strings are immutable**
```python
name = "maria"
name[0] = "M"     # TypeError — strings can't be modified in place
```
Fix: create a new string — `name = "M" + name[1:]` or use `.title()`/`.capitalize()`.

**2. Comparing text without normalizing case first**
```python
"Electronics" == "electronics"   # False — comparison is case-sensitive
```
Fix: `.lower()` both sides before comparing: `"Electronics".lower() == "electronics".lower()`.

**3. Not stripping whitespace before comparing or converting**
```python
raw = " 24.99 "
float(raw)      # actually works fine — float() ignores surrounding whitespace
" 24.99 " == "24.99"    # False — this one does NOT ignore whitespace
```
Fix: always `.strip()` text before equality checks or grouping.

**4. Using `.split()` without expecting extra/missing parts**
```python
"SKU-1042".split("-")        # ['SKU', '1042'] — only 2 parts, not 3
parts[2]                     # IndexError if you assumed 3 parts
```
Fix: check `len(parts)` or use `.split("-", maxsplit=1)` when you know the exact structure you expect.

**Data Engineer Tip:** In production pipelines, string cleaning is usually where most of your code lives — not the "interesting" analysis. A robust pipeline normalizes case, strips whitespace, and validates format *before* any row is trusted downstream. Many real data-quality bugs (duplicate categories like `"Electronics"` and `"electronics "` being treated as different groups) trace back to a missing `.strip().lower()` somewhere upstream.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `raw = "  Home & Garden  "`, clean it by stripping whitespace and converting to title case. Print the result.

<details>
<summary>Hint</summary>
Chain `.strip()` and `.title()`.
</details>

<details>
<summary>Solution</summary>

```python
raw = "  Home & Garden  "
clean = raw.strip().title()
print(clean)   # "Home & Garden"
```
</details>

---

**Exercise 2 (Easy):** Given `email = "Maria.Chen@Example.COM"`, print a lowercased version suitable for consistent storage/comparison.

<details>
<summary>Hint</summary>
`.lower()`
</details>

<details>
<summary>Solution</summary>

```python
email = "Maria.Chen@Example.COM"
print(email.lower())   # "maria.chen@example.com"
```
</details>

---

**Exercise 3 (Medium):** Given `price_text = "USD 129.99"`, extract just the numeric part and convert it to a float. (Hint: `.split()` on whitespace, then take the right piece.)

<details>
<summary>Hint</summary>
`price_text.split(" ")` gives `["USD", "129.99"]` — grab index 1.
</details>

<details>
<summary>Solution</summary>

```python
price_text = "USD 129.99"
numeric_part = price_text.split(" ")[1]
price = float(numeric_part)
print(price)   # 129.99
```
</details>

---

**Exercise 4 (Medium):** Given `full_name = "chen, maria"` (lastname, firstname format, messy casing), transform it into `"Maria Chen"` (firstname lastname, title case).

<details>
<summary>Hint</summary>
Split on `", "`, then rearrange, then `.title()`.
</details>

<details>
<summary>Solution</summary>

```python
full_name = "chen, maria"
last, first = full_name.split(", ")
formatted = f"{first.title()} {last.title()}"
print(formatted)   # "Maria Chen"
```
</details>

---

**Exercise 5 (Hard):** Given a raw log line `log_line = "2024-11-03 | user_id=204 | event=purchase | amount=$59.99"`, extract the `user_id` as an int, the `event` as a string, and the `amount` as a float, into three separate variables. Print all three with their types.

<details>
<summary>Hint</summary>
Split on `" | "` first to get four pieces. Each piece is `key=value` — split each on `"="`. For `amount`, also strip the `$` before converting to float.
</details>

<details>
<summary>Solution</summary>

```python
log_line = "2024-11-03 | user_id=204 | event=purchase | amount=$59.99"

parts = log_line.split(" | ")
# parts = ['2024-11-03', 'user_id=204', 'event=purchase', 'amount=$59.99']

user_id_part = parts[1].split("=")[1]        # "204"
event_part = parts[2].split("=")[1]          # "purchase"
amount_part = parts[3].split("=")[1]         # "$59.99"

user_id = int(user_id_part)
event = event_part
amount = float(amount_part.replace("$", ""))

print(type(user_id), user_id)
print(type(event), event)
print(type(amount), amount)
```
</details>

---

**Previous lesson:** [1.2 — Operators & Expressions](lesson-1.2-operators-and-expressions.md)
**Next lesson:** [1.4 — Conditional Logic](lesson-1.4-conditionals.md)
