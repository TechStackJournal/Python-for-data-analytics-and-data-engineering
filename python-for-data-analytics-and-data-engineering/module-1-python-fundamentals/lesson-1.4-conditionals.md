# Lesson 1.4 — Conditional Logic

## CONCEPT

**Conditional statements** let a program make decisions — running different code depending on whether a condition is `True` or `False`. Python uses `if`, `elif` ("else if"), and `else`:

```python
if condition:
    # runs only if condition is True
elif other_condition:
    # runs only if condition was False AND other_condition is True
else:
    # runs if none of the above were True
```

Python uses **indentation** (not curly braces) to define which lines belong to which block. This isn't just a style choice — it's part of the language's syntax, and inconsistent indentation causes errors.

## WHY IT MATTERS

Almost every business rule in data analytics is a conditional: "flag this order as high-risk if...", "categorize this customer as churned if...", "assign this transaction to tier A/B/C based on amount." Conditional logic here is the direct ancestor of `CASE WHEN` in SQL and `np.where()`/`.apply()` logic in pandas — the same decision-making pattern, expressed in different syntax at each layer of your stack.

## SYNTAX REFERENCE

```python
# Basic if
if order_total > 100:
    print("Large order")

# if / else
if in_stock:
    print("Available")
else:
    print("Out of stock")

# if / elif / else chain
if tier == "gold":
    discount = 0.20
elif tier == "silver":
    discount = 0.10
elif tier == "bronze":
    discount = 0.05
else:
    discount = 0.0

# Nested conditionals
if is_member:
    if order_total > 50:
        print("Free shipping")
    else:
        print("Standard shipping")

# Ternary (conditional) expression — a compact one-line if/else
status = "active" if login_count > 0 else "inactive"

# Truthy / falsy values (used implicitly in "if value:")
# Falsy: 0, 0.0, "", [], {}, None, False
# Truthy: everything else
if discount_list:      # True only if the list is non-empty
    apply_discounts()
```

## WORKED EXAMPLE

Categorizing an order into a shipping tier, based on order total and membership status:

```python
order_total = 84.50
is_prime_member = True
item_count = 4

if is_prime_member and order_total >= 35:
    shipping = "Free (Prime)"
elif order_total >= 75:
    shipping = "Free (order threshold)"
elif order_total >= 35:
    shipping = "Discounted ($4.99)"
else:
    shipping = "Standard ($8.99)"

print(f"Shipping tier: {shipping}")   # "Free (Prime)"

# Classify order size using a ternary expression
order_size = "bulk" if item_count >= 10 else "standard"
print(f"Order size: {order_size}")    # "standard"

# Nested conditional: flag orders needing manual review
flagged_reason = None
if order_total > 500:
    if not is_prime_member:
        flagged_reason = "High value, non-member — review"
    else:
        flagged_reason = None   # trusted member, no review needed
elif item_count > 20:
    flagged_reason = "Unusually high item count — review"

if flagged_reason:
    print(f"FLAGGED: {flagged_reason}")
else:
    print("No review needed")
```

## COMMON MISTAKES

**1. Inconsistent indentation**
```python
if order_total > 100:
    print("Large order")
      print("Free gift included")   # IndentationError
```
Fix: use a consistent number of spaces (4 is the Python standard) for every line in a block. Most editors handle this automatically — never mix tabs and spaces.

**2. Using `=` instead of `==` in a condition**
```python
if order_total = 100:   # SyntaxError
```
See Lesson 1.2 — this is one of the most common beginner errors.

**3. Writing unreachable `elif` branches**
```python
if order_total > 50:
    tier = "A"
elif order_total > 100:   # unreachable! anything > 100 already matched > 50 above
    tier = "B"
```
Fix: order conditions from most specific to least specific (or most restrictive first).

**4. Forgetting that `elif`/`else` only run if *no earlier* condition matched**
```python
tier = "gold"
if tier == "gold":
    print("Gold tier")
if tier == "silver":       # this is a SEPARATE if, not elif — still gets checked
    print("Silver tier")
```
This isn't a syntax error, but it's a logic bug — using separate `if` statements when you meant `elif` can cause multiple branches to run when you expected only one.

**5. Misreading truthy/falsy checks**
```python
count = 0
if count:            # False, because 0 is falsy — even though count "exists"
    print("Has items")
```
Fix: be explicit when you mean to check for zero specifically — `if count > 0:` is clearer than relying on truthiness, especially for beginners.

**Data Engineer Tip:** This exact `if/elif/else` chain becomes `CASE WHEN ... THEN ... ELSE ... END` in SQL and `np.select()` in pandas when you're categorizing thousands of rows at once. Writing the row-by-row logic clearly here makes translating it into a vectorized version later much easier — you're not learning new logic, just a new syntax for the same decision tree.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `stock_count = 0`, print `"Out of stock"` if it's zero, otherwise print `"In stock"`.

<details>
<summary>Hint</summary>
A simple `if/else` on `stock_count == 0`.
</details>

<details>
<summary>Solution</summary>

```python
stock_count = 0
if stock_count == 0:
    print("Out of stock")
else:
    print("In stock")
```
</details>

---

**Exercise 2 (Easy):** Given `age = 17`, print `"Minor"` if under 18, `"Adult"` otherwise.

<details>
<summary>Hint</summary>
`if age < 18:`
</details>

<details>
<summary>Solution</summary>

```python
age = 17
if age < 18:
    print("Minor")
else:
    print("Adult")
```
</details>

---

**Exercise 3 (Medium):** Given `order_total = 45`, assign a `shipping_cost` using this rule: free if `order_total >= 75`, `$5.99` if between `35` and `74.99`, otherwise `$9.99`. Print the result.

<details>
<summary>Hint</summary>
Use `if/elif/else`, checking the highest threshold first.
</details>

<details>
<summary>Solution</summary>

```python
order_total = 45

if order_total >= 75:
    shipping_cost = 0
elif order_total >= 35:
    shipping_cost = 5.99
else:
    shipping_cost = 9.99

print(f"Shipping cost: ${shipping_cost}")
```
</details>

---

**Exercise 4 (Medium):** Given `login_count = 0` and `days_since_signup = 45`, classify a customer as `"churned"` if `login_count == 0` and `days_since_signup > 30`, as `"new"` if `days_since_signup <= 30`, and `"active"` otherwise. Print the classification.

<details>
<summary>Hint</summary>
Check the churned condition first (it's the most specific combination), then the new condition, then default to active.
</details>

<details>
<summary>Solution</summary>

```python
login_count = 0
days_since_signup = 45

if login_count == 0 and days_since_signup > 30:
    status = "churned"
elif days_since_signup <= 30:
    status = "new"
else:
    status = "active"

print(status)   # "churned"
```
</details>

---

**Exercise 5 (Hard):** Write a discount calculator: given `order_total`, `customer_tier` (one of `"gold"`, `"silver"`, `"bronze"`, `"none"`), and `is_holiday_sale` (bool), compute a `discount_pct` using these rules, applied in order:
- If `is_holiday_sale` is `True`, everyone gets at least a 15% discount, regardless of tier.
- Otherwise: gold = 20%, silver = 10%, bronze = 5%, none = 0%.
- Additionally, regardless of the above, if `order_total > 200`, add an extra 5 percentage points on top (e.g., 20% becomes 25%).

Test with `order_total = 250`, `customer_tier = "silver"`, `is_holiday_sale = False`, and print the final `discount_pct`.

<details>
<summary>Hint</summary>
Handle the base discount first with if/elif/else, then apply the order_total bonus as a separate, independent `if` afterward (not part of the same chain, since it can apply on top of any tier).
</details>

<details>
<summary>Solution</summary>

```python
order_total = 250
customer_tier = "silver"
is_holiday_sale = False

if is_holiday_sale:
    discount_pct = 15
elif customer_tier == "gold":
    discount_pct = 20
elif customer_tier == "silver":
    discount_pct = 10
elif customer_tier == "bronze":
    discount_pct = 5
else:
    discount_pct = 0

# Separate check — can stack on top of any branch above
if order_total > 200:
    discount_pct += 5

print(f"Final discount: {discount_pct}%")   # 15%
```
</details>

---

**Previous lesson:** [1.3 — Strings & String Methods](lesson-1.3-strings.md)
**Next lesson:** [1.5 — Loops & Loop Control](lesson-1.5-loops.md)
