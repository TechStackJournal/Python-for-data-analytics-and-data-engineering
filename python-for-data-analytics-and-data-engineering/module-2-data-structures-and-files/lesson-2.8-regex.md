# Lesson 2.8 — Regular Expressions for Text Cleaning

## CONCEPT

A **regular expression** (regex) is a pattern-matching language for finding, validating, and extracting text that follows a certain shape — even when the exact content varies. Python's built-in `re` module applies regex patterns to strings.

```python
import re

text = "Order #1042 shipped on 2024-11-03"
match = re.search(r"\d{4}-\d{2}-\d{2}", text)
print(match.group())   # "2024-11-03"
```

Regex is more powerful than the string methods from Lesson 1.3 (`.split()`, `.replace()`) when the pattern you're matching isn't a fixed, known substring — e.g., "find any sequence of digits" rather than "find the exact word 'mouse'."

## WHY IT MATTERS

Real-world text data is inconsistent in ways plain string methods can't handle cleanly: phone numbers formatted five different ways, prices embedded in free-text descriptions, log lines with a semi-predictable structure, email validation, extracting an order ID from a subject line. Regex is the standard tool for this class of problem across nearly every programming language, and it reappears directly in pandas as the `.str.extract()`, `.str.contains()`, and `.str.replace()` methods (Module 4) — everything here transfers directly.

## SYNTAX REFERENCE

```python
import re

# Core functions
re.search(pattern, text)     # find the FIRST match anywhere in the string; None if no match
re.match(pattern, text)      # match only at the START of the string
re.findall(pattern, text)    # find ALL matches, returned as a list
re.sub(pattern, replacement, text)   # replace all matches with something else
re.split(pattern, text)      # split a string using a regex pattern as the delimiter

# Common pattern building blocks
\d       # any digit (0-9)
\D       # any NON-digit
\w       # any "word" character (letters, digits, underscore)
\s       # any whitespace
.        # any character except newline
+        # one or more of the preceding
*        # zero or more of the preceding
?        # zero or one of the preceding (optional)
{n}      # exactly n repetitions
{n,m}    # between n and m repetitions
^        # start of string
$        # end of string
[abc]    # any one of a, b, or c
[^abc]   # any character EXCEPT a, b, or c
(...)    # a capturing group — extract just this part of the match
|        # OR

# Examples
re.search(r"\d+", "Order #1042")                 # matches "1042"
re.findall(r"\d+", "SKU-104, SKU-205")             # ["104", "205"]
re.sub(r"[^\d.]", "", "$24.99")                    # "24.99" — strip everything but digits/dot
re.match(r"^ORD-", "ORD-1042")                     # matches (starts with "ORD-")

# Using groups to extract specific parts
match = re.search(r"(\d{4})-(\d{2})-(\d{2})", "2024-11-03")
match.group(0)   # "2024-11-03" — the whole match
match.group(1)   # "2024" — first parenthesized group
match.group(2)   # "11"
match.group(3)   # "03"

# Raw strings (the r"" prefix) — always use these for regex patterns
r"\d+"    # correct — the backslash is passed to re literally
"\d+"     # works but risks warnings/bugs — Python may try to interpret \d itself
```

## WORKED EXAMPLE

Cleaning a batch of messy raw text fields using regex — extracting order IDs from free-text, validating emails, and stripping currency symbols:

```python
import re

# 1. Extracting an order ID embedded in a free-text subject line
subject_lines = [
    "Your order #1042 has shipped!",
    "Update on order 2093 - delayed",
    "Re: ORD-3301 delivery confirmation",
]

for line in subject_lines:
    match = re.search(r"\d{3,}", line)   # find a run of 3+ digits
    if match:
        print(f"Found order ID: {match.group()}")
    else:
        print(f"No order ID found in: {line}")

# Found order ID: 1042
# Found order ID: 2093
# Found order ID: 3301

# 2. Validating email format (a simple, practical check — not a fully RFC-compliant one)
emails = ["maria@example.com", "not-an-email", "james.lee@company.co.uk"]

email_pattern = r"^[\w.\-]+@[\w.\-]+\.\w+$"
for email in emails:
    is_valid = re.match(email_pattern, email) is not None
    print(f"{email}: {'valid' if is_valid else 'invalid'}")

# maria@example.com: valid
# not-an-email: invalid
# james.lee@company.co.uk: valid

# 3. Stripping everything except digits and the decimal point from messy price text
raw_prices = ["$24.99", "USD 12.50", "Price: 89.00 dollars", "  45.75  "]

cleaned_prices = []
for raw in raw_prices:
    cleaned = re.sub(r"[^\d.]", "", raw)   # keep only digits and "."
    cleaned_prices.append(float(cleaned))

print(cleaned_prices)   # [24.99, 12.5, 89.0, 45.75]

# 4. Extracting all product SKUs from a block of text
description = "This order contains SKU-1042, SKU-2093, and SKU-3301."
skus = re.findall(r"SKU-\d+", description)
print(skus)   # ['SKU-1042', 'SKU-2093', 'SKU-3301']
```

## COMMON MISTAKES

**1. Forgetting the `r""` raw string prefix**
```python
re.search("\d+", text)    # works, but risky — some escape sequences mean something else to Python itself
re.search(r"\d+", text)   # correct — always use raw strings for regex patterns
```
Fix: make `r"..."` a habit for every regex pattern, without exception.

**2. Using `re.search()` when you meant `re.findall()` (or vice versa)**
```python
re.search(r"\d+", "104, 205, 310")     # only returns the FIRST match: "104"
re.findall(r"\d+", "104, 205, 310")    # returns ALL matches: ['104', '205', '310']
```
Fix: be deliberate — `search` for "does this exist, and where's the first one," `findall` for "give me everything."

**3. Not checking whether `re.search()`/`re.match()` actually found anything before calling `.group()`**
```python
match = re.search(r"\d+", "no numbers here")
match.group()   # AttributeError: 'NoneType' object has no attribute 'group'
```
Fix: always check `if match:` before calling `.group()` — a non-match returns `None`, not an empty match.

**4. Overly greedy or overly narrow patterns**
```python
re.findall(r"\d+", "$24.99")   # ['24', '99'] — NOT '24.99'! \d+ doesn't include the decimal point
```
Fix: think carefully about what characters actually belong in your target pattern — here, `\d+\.\d+` would correctly match `"24.99"` as one piece.

**5. Trying to parse deeply nested/structured formats (like full HTML or JSON) with regex**
Regex is excellent for finding patterns in unstructured or semi-structured text, but it's the wrong tool for parsing genuinely structured formats — use `json.loads()` for JSON (Lesson 2.6), not regex. A common saying in programming: regex can create more problems than it solves when pushed to parse something that already has a proper parser available.

**Data Engineer Tip:** In pandas, `df["text_column"].str.extract(r"pattern")` and `df["text_column"].str.contains(r"pattern")` use exactly this same regex syntax, applied to an entire column at once instead of one string at a time. If you can write and test a pattern here against a single string, you already know exactly how to apply it across a whole dataset later — the regex itself doesn't change, only the surrounding syntax.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `text = "Contact us at support@example.com for help"`, use `re.search()` to find and print the email address.

<details>
<summary>Hint</summary>
A simple pattern: `r"[\w.\-]+@[\w.\-]+\.\w+"`
</details>

<details>
<summary>Solution</summary>

```python
import re

text = "Contact us at support@example.com for help"
match = re.search(r"[\w.\-]+@[\w.\-]+\.\w+", text)
print(match.group())   # "support@example.com"
```
</details>

---

**Exercise 2 (Easy):** Given `text = "Order total: $145.50"`, use `re.sub()` to strip out everything except digits and the decimal point, then convert the result to a float.

<details>
<summary>Hint</summary>
`re.sub(r"[^\d.]", "", text)`, then `float(...)`.
</details>

<details>
<summary>Solution</summary>

```python
import re

text = "Order total: $145.50"
cleaned = re.sub(r"[^\d.]", "", text)
price = float(cleaned)
print(price)   # 145.5
```
</details>

---

**Exercise 3 (Medium):** Given `log_line = "user_id=204 action=purchase amount=59.99"`, use `re.findall()` with a pattern like `r"\w+=\S+"` to extract all `key=value` pairs, then build a dictionary from them by splitting each pair on `"="`.

<details>
<summary>Hint</summary>
`re.findall(r"\w+=\S+", log_line)` gives you a list of strings like `"user_id=204"` — split each on `"="` to build the dict.
</details>

<details>
<summary>Solution</summary>

```python
import re

log_line = "user_id=204 action=purchase amount=59.99"
pairs = re.findall(r"\w+=\S+", log_line)

result = {}
for pair in pairs:
    key, value = pair.split("=")
    result[key] = value

print(result)   # {'user_id': '204', 'action': 'purchase', 'amount': '59.99'}
```
</details>

---

**Exercise 4 (Medium):** Given a list of raw phone numbers in inconsistent formats — `["555-123-4567", "(555) 987-6543", "555.246.8100", "5551239999"]` — write a function that uses `re.sub()` to strip everything except digits from each, so all four become a consistent 10-digit string. Print the cleaned list.

<details>
<summary>Hint</summary>
`re.sub(r"\D", "", phone)` removes every non-digit character.
</details>

<details>
<summary>Solution</summary>

```python
import re

phones = ["555-123-4567", "(555) 987-6543", "555.246.8100", "5551239999"]

def clean_phone(phone):
    return re.sub(r"\D", "", phone)

cleaned = [clean_phone(p) for p in phones]
print(cleaned)
# ['5551234567', '5559876543', '5552468100', '5551239999']
```
</details>

---

**Exercise 5 (Hard):** Given a block of raw customer support text:
```python
text = """
Ticket #4471: order 1042 - customer email jane.doe@example.com - amount $89.99
Ticket #4472: order 2093 - customer email not-valid-email - amount $145.00
Ticket #4473: order 3301 - customer email bob@company.co - amount $12.50
"""
```
Write code that, for each ticket line, extracts: the ticket number, order ID, email (validate it — if invalid, store `None` instead), and amount as a float. Build a list of dicts, one per ticket, and print them.

<details>
<summary>Hint</summary>
Split the text into lines with `.strip().split("\n")`, skipping empty lines. For each line, use separate `re.search()` calls with targeted patterns for the ticket number (`r"Ticket #(\d+)"`), order ID (`r"order (\d+)"`), email (validate with the pattern from the worked example), and amount (`r"\$(\d+\.\d+)"`). Use `.group(1)` to get just the captured part of each match.
</details>

<details>
<summary>Solution</summary>

```python
import re

text = """
Ticket #4471: order 1042 - customer email jane.doe@example.com - amount $89.99
Ticket #4472: order 2093 - customer email not-valid-email - amount $145.00
Ticket #4473: order 3301 - customer email bob@company.co - amount $12.50
"""

email_pattern = r"^[\w.\-]+@[\w.\-]+\.\w+$"
tickets = []

lines = [line.strip() for line in text.strip().split("\n") if line.strip()]

for line in lines:
    ticket_match = re.search(r"Ticket #(\d+)", line)
    order_match = re.search(r"order (\d+)", line)
    email_match = re.search(r"email (\S+)", line)
    amount_match = re.search(r"\$(\d+\.\d+)", line)

    raw_email = email_match.group(1) if email_match else None
    valid_email = raw_email if (raw_email and re.match(email_pattern, raw_email)) else None

    tickets.append({
        "ticket_id": int(ticket_match.group(1)) if ticket_match else None,
        "order_id": int(order_match.group(1)) if order_match else None,
        "email": valid_email,
        "amount": float(amount_match.group(1)) if amount_match else None,
    })

for t in tickets:
    print(t)

# {'ticket_id': 4471, 'order_id': 1042, 'email': 'jane.doe@example.com', 'amount': 89.99}
# {'ticket_id': 4472, 'order_id': 2093, 'email': None, 'amount': 145.0}
# {'ticket_id': 4473, 'order_id': 3301, 'email': 'bob@company.co', 'amount': 12.5}
```
</details>

---

## Module 2 Complete

You've now covered every core data structure and file format you'll encounter before pandas: lists, tuples, sets, dictionaries, comprehensions, dates, CSV, JSON, file paths, and regex-based text cleaning. Module 3 introduces NumPy — the first step toward *vectorized* thinking, where these same operations get applied to entire arrays of data at once instead of one value or row at a time.

**Previous lesson:** [2.7 — File Paths with pathlib](lesson-2.7-pathlib.md)
**Next module:** [Module 3 — NumPy](../module-3-numpy/README.md)
