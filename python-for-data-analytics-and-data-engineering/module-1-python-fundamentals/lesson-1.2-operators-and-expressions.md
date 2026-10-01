# Lesson 1.2 — Operators & Expressions

## CONCEPT

An **operator** is a symbol that performs an operation on one or more values (called **operands**). An **expression** is any combination of values and operators that Python can evaluate down to a single result.

```python
3 + 4        # expression: two operands (3, 4) and one operator (+), evaluates to 7
```

Python has several families of operators:

- **Arithmetic**: `+ - * / // % **`
- **Comparison**: `== != > < >= <=`
- **Logical**: `and or not`
- **Assignment**: `= += -= *= /=`
- **Membership**: `in`, `not in`
- **Identity**: `is`, `is not`

## WHY IT MATTERS

Every filtering condition, every business rule, every calculated column in a data pipeline is built from operators. "Flag orders over $100," "keep only active customers," "compute profit margin" — all of these are expressions using exactly the operators in this lesson. Getting comfortable reading and writing them fluently is a direct multiplier on how fast you can write filtering and transformation logic later in pandas and SQL.

## SYNTAX REFERENCE

```python
# Arithmetic
a + b        # addition
a - b        # subtraction
a * b        # multiplication
a / b        # division (always returns a float)
a // b       # floor division (drops the remainder)
a % b        # modulo (the remainder)
a ** b       # exponentiation

# Comparison (returns True/False)
a == b       # equal to
a != b       # not equal to
a > b        # greater than
a < b        # less than
a >= b       # greater than or equal to
a <= b       # less than or equal to

# Logical
a and b      # True only if both are True
a or b       # True if at least one is True
not a        # flips True/False

# Assignment shortcuts
total = 0
total += 5   # same as total = total + 5
total -= 2   # same as total = total - 2
total *= 3   # same as total = total * 3

# Membership
"gold" in ["bronze", "silver", "gold"]      # True
5 not in [1, 2, 3]                          # True
```

## WORKED EXAMPLE

Simulating a business rule against one `orders` row: apply free shipping if the order total is $75+ **and** the customer is in a "loyalty" tier.

```python
order_total = 82.50
customer_tier = "loyalty"
item_count = 3

# Comparison operators build the individual conditions
meets_total_threshold = order_total >= 75
is_loyalty_customer = customer_tier == "loyalty"

# Logical operators combine them
qualifies_for_free_shipping = meets_total_threshold and is_loyalty_customer
print("Free shipping?", qualifies_for_free_shipping)   # True

# Arithmetic: compute a per-item average
avg_item_price = order_total / item_count
print(f"Average item price: ${avg_item_price:.2f}")    # $27.50

# Modulo: is this the customer's 10th order (a milestone)?
order_number = 30
is_milestone_order = order_number % 10 == 0
print("Milestone order?", is_milestone_order)           # True

# Membership: is this order's category in a promo list?
promo_categories = ["Electronics", "Home", "Outdoor"]
order_category = "Electronics"
in_promo = order_category in promo_categories
print("In promo?", in_promo)                             # True
```

## COMMON MISTAKES

**1. Using `=` instead of `==`**
```python
if order_total = 75:    # SyntaxError
```
Fix: `==` compares, `=` assigns. This trips up nearly every beginner at least once.

**2. Chaining `and`/`or` without understanding short-circuiting**
```python
result = 0 and (10 / 0)   # No ZeroDivisionError! Python stops at the first False.
```
Python evaluates left to right and stops as soon as the result is determined. This is usually helpful but can hide bugs if you rely on the second expression always running.

**3. Forgetting `/` always returns a float, even for whole-number results**
```python
10 / 2    # 5.0, not 5
```
If you specifically want a whole number, use `//` (floor division) — but be careful, `//` rounds *down*, including for negative numbers (`-7 // 2` is `-4`, not `-3`).

**4. Confusing `%` (modulo) with percentage**
```python
17 % 5    # 2 — this is "remainder", not "17 percent of 5"
```

**Data Engineer Tip:** Business logic like "flag high-value orders" or "apply a discount if X and Y" starts life as exactly these boolean expressions. When you write a `WHERE` clause in SQL or a `.loc[]` filter in pandas later, you're expressing the same logic — just with different syntax. Getting fluent with `and`/`or`/comparison chains here pays off directly in both.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `price = 45` and `quantity = 3`, calculate the total, then check whether the total is greater than 100. Print both values.

<details>
<summary>Hint</summary>
`total = price * quantity`, then use `>` for the comparison.
</details>

<details>
<summary>Solution</summary>

```python
price = 45
quantity = 3
total = price * quantity
is_over_100 = total > 100
print(total, is_over_100)
```
</details>

---

**Exercise 2 (Easy):** Given `stock_count = 47`, determine how many full boxes of 12 units can be made, and how many units are left over. Print both.

<details>
<summary>Hint</summary>
Floor division (`//`) gives full boxes; modulo (`%`) gives the remainder.
</details>

<details>
<summary>Solution</summary>

```python
stock_count = 47
full_boxes = stock_count // 12
leftover = stock_count % 12
print(f"Full boxes: {full_boxes}, leftover units: {leftover}")
```
</details>

---

**Exercise 3 (Medium):** A customer qualifies for a "VIP" discount if their `total_spent` is over 500 **or** their `order_count` is at least 20. Given `total_spent = 620` and `order_count = 8`, write the boolean expression and print whether they qualify.

<details>
<summary>Hint</summary>
Use `or` to combine the two conditions — only one needs to be true.
</details>

<details>
<summary>Solution</summary>

```python
total_spent = 620
order_count = 8

is_vip = total_spent > 500 or order_count >= 20
print("VIP?", is_vip)   # True, because total_spent > 500
```
</details>

---

**Exercise 4 (Medium):** Given a list `active_categories = ["Electronics", "Books", "Home"]` and a variable `product_category = "Toys"`, check whether the product's category is **not** in the active list, and print a message accordingly.

<details>
<summary>Hint</summary>
Use `not in`.
</details>

<details>
<summary>Solution</summary>

```python
active_categories = ["Electronics", "Books", "Home"]
product_category = "Toys"

if product_category not in active_categories:
    print(f"{product_category} is not currently an active category.")
```
</details>

---

**Exercise 5 (Hard):** An order qualifies for a "priority ship" flag if **all** of these are true: `order_total >= 50`, `is_prime_member` is `True`, and the `item_count` is between 1 and 10 inclusive. Given `order_total = 60`, `is_prime_member = True`, `item_count = 12`, write a single boolean expression combining all three conditions and print the result along with which condition (if any) failed.

<details>
<summary>Hint</summary>
Combine with `and`. For "between 1 and 10 inclusive," you can chain comparisons: `1 <= item_count <= 10`.
</details>

<details>
<summary>Solution</summary>

```python
order_total = 60
is_prime_member = True
item_count = 12

meets_total = order_total >= 50
meets_membership = is_prime_member
meets_item_count = 1 <= item_count <= 10

qualifies = meets_total and meets_membership and meets_item_count
print("Priority ship?", qualifies)   # False

if not meets_item_count:
    print("Failed condition: item_count out of range")
```
</details>

---

**Previous lesson:** [1.1 — Variables & Data Types](lesson-1.1-variables-and-types.md)
**Next lesson:** [1.3 — Strings & String Methods](lesson-1.3-strings.md)
