# Lesson 1.7 — Scope & Namespaces

## CONCEPT

**Scope** determines *where* in your code a variable is visible and usable. Python has a few key scope levels:

- **Local scope**: variables created inside a function exist only within that function.
- **Enclosing scope**: variables in an outer function, visible to a nested inner function.
- **Global scope**: variables created at the top level of a script, visible everywhere in that file.
- **Built-in scope**: names Python provides automatically (`len`, `print`, `sum`, etc.).

Python looks up a name using the **LEGB rule** (Local → Enclosing → Global → Built-in) — checking each level in that order until it finds a match.

```python
x = "global"

def show():
    x = "local"
    print(x)     # "local" — the local one wins inside the function

show()
print(x)         # "global" — unaffected outside the function
```

A **namespace** is simply the mapping between names and the objects they refer to at a given scope — think of it as the "address book" Python checks when resolving a variable name.

## WHY IT MATTERS

Scope bugs are extremely common for beginners: modifying a variable inside a function and expecting the change to appear outside it (it usually doesn't, unless you're explicit), or accidentally reusing a variable name across different parts of a script and getting confused about which value is "current." As your pipeline scripts grow past a few functions, understanding scope is what lets you reason about *where a bug's data actually came from* — a skill that becomes essential once you're debugging a multi-stage ETL job in Module 9.

## SYNTAX REFERENCE

```python
# Local scope — default behavior
def process():
    total = 0          # local to process(); doesn't exist outside it
    total += 10
    return total

# Reading a global variable from inside a function works automatically
tax_rate = 0.08

def calculate_tax(amount):
    return amount * tax_rate   # reads the global tax_rate — fine

# Modifying a global variable from inside a function requires 'global'
counter = 0

def increment():
    global counter
    counter += 1        # without 'global' here, this would raise an error

# Enclosing scope (nested functions / closures)
def outer():
    message = "hello"
    def inner():
        print(message)   # inner() can read outer()'s variable
    inner()

# Function parameters are always local to that function
def greet(name):
    name = name.upper()   # only changes the local copy
    return name

original = "maria"
result = greet(original)
print(original, result)   # "maria" "MARIA" — original is untouched
```

## WORKED EXAMPLE

Demonstrating how local variables stay isolated, and the difference between reading and modifying globals:

```python
# Global configuration values (common in small scripts)
DEFAULT_TAX_RATE = 0.08
processed_order_count = 0    # a running counter we intend to update globally

def calculate_total(price, quantity):
    """Local variables here don't leak outside this function."""
    subtotal = price * quantity       # local to this function
    tax = subtotal * DEFAULT_TAX_RATE # reading a global — allowed automatically
    total = subtotal + tax
    return round(total, 2)


def record_processed_order():
    """Modifying a global requires the 'global' keyword."""
    global processed_order_count
    processed_order_count += 1


orders = [(24.99, 2), (9.99, 1), (49.99, 3)]

for price, qty in orders:
    total = calculate_total(price, qty)
    record_processed_order()
    print(f"Order total: ${total}")

print(f"Orders processed: {processed_order_count}")   # 3

# Trying to access 'subtotal' here would raise a NameError —
# it only ever existed inside calculate_total()
# print(subtotal)   # NameError: name 'subtotal' is not defined
```

## COMMON MISTAKES

**1. Expecting a function to modify a variable it was only "given"**
```python
def apply_discount(total):
    total = total * 0.9   # only changes the LOCAL copy
    return total

order_total = 100
apply_discount(order_total)
print(order_total)   # still 100! The function didn't change the original.
```
Fix: use the return value — `order_total = apply_discount(order_total)`.

**2. Forgetting `global` when trying to modify a global variable**
```python
counter = 0

def increment():
    counter += 1    # UnboundLocalError! Python sees the assignment
                     # and assumes counter is LOCAL, unless told otherwise
```
Fix: add `global counter` as the first line inside the function — though in general, relying heavily on `global` is considered poor practice (see below).

**3. Overusing global variables instead of passing values explicitly**
```python
# Harder to test and reason about:
running_total = 0
def add_order(amount):
    global running_total
    running_total += amount

# Easier to test and reuse — no hidden global state:
def add_order(amount, running_total):
    return running_total + amount
```
Fix: prefer passing values in and returning results out over relying on global state — this becomes especially important once you start writing `pytest` tests in Module 8.

**4. Variable name collisions across a long script**
Reusing a generic name like `data` or `result` in many places in one file makes it easy to accidentally use a stale value. Fix: use descriptive, specific names (`cleaned_orders`, `daily_total`) rather than generic ones once a script grows past a few dozen lines.

**Data Engineer Tip:** Production pipeline code deliberately avoids global mutable state wherever possible — each function should receive exactly the data it needs as parameters and return exactly what the next step needs, rather than reading and writing shared global variables. This "avoid hidden state" principle is *why* pure functions (mentioned in Lesson 1.6) are preferred: they don't depend on, or alter, anything outside their own inputs and outputs, which makes pipelines predictable and testable.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Predict the output before running this code, then verify it:
```python
value = 10

def change_value():
    value = 99
    return value

print(change_value())
print(value)
```

<details>
<summary>Hint</summary>
The `value` inside the function is a separate local variable from the global one.
</details>

<details>
<summary>Solution</summary>

```python
value = 10

def change_value():
    value = 99
    return value

print(change_value())   # 99 (the local value)
print(value)             # 10 (the global value, unaffected)
```
</details>

---

**Exercise 2 (Easy):** Write a function `add_five(n)` that takes a number, adds 5 to a *local copy*, and returns it — without modifying anything outside the function. Show that the original variable is unchanged after calling it.

<details>
<summary>Hint</summary>
Just use a normal local variable and `return` — no `global` needed here.
</details>

<details>
<summary>Solution</summary>

```python
def add_five(n):
    n = n + 5
    return n

original = 10
result = add_five(original)
print(original, result)   # 10 15
```
</details>

---

**Exercise 3 (Medium):** Create a global variable `total_orders_processed = 0`. Write a function `process_order()` that increments this global variable by 1 each time it's called (using the `global` keyword). Call it three times, then print the final count.

<details>
<summary>Hint</summary>
Remember `global total_orders_processed` must be the first line inside the function.
</details>

<details>
<summary>Solution</summary>

```python
total_orders_processed = 0

def process_order():
    global total_orders_processed
    total_orders_processed += 1

process_order()
process_order()
process_order()
print(total_orders_processed)   # 3
```
</details>

---

**Exercise 4 (Medium):** Refactor the function from Exercise 3 to avoid using `global` entirely — instead, have `process_order(count)` take the current count as a parameter and return the incremented value. Call it in a loop 3 times, updating a variable in the main script each time.

<details>
<summary>Hint</summary>
`return count + 1`, then reassign the outer variable each loop iteration: `count = process_order(count)`.
</details>

<details>
<summary>Solution</summary>

```python
def process_order(count):
    return count + 1

count = 0
for _ in range(3):
    count = process_order(count)

print(count)   # 3
```
</details>

---

**Exercise 5 (Hard):** Explain (in a comment) and demonstrate the difference between these two functions, then predict what each `print()` outputs before running:
```python
def broken_increment(counter):
    counter += 1
    return counter

def uses_list(counter_list):
    counter_list[0] += 1
    return counter_list

count = 5
broken_increment(count)
print(count)   # ?

count_list = [5]
uses_list(count_list)
print(count_list)   # ?
```
Then write 2–3 sentences explaining why the second one behaves differently from the first, in terms of how Python handles mutable vs. immutable objects as function arguments.

<details>
<summary>Hint</summary>
Integers are immutable — reassigning `counter` inside the function only rebinds the local name. Lists are mutable — `counter_list[0] += 1` modifies the actual list object in place, which is visible outside the function since both names point to the same object in memory.
</details>

<details>
<summary>Solution</summary>

```python
def broken_increment(counter):
    counter += 1
    return counter

def uses_list(counter_list):
    counter_list[0] += 1
    return counter_list

count = 5
broken_increment(count)
print(count)   # 5 — unchanged, integers are immutable

count_list = [5]
uses_list(count_list)
print(count_list)   # [6] — changed! Lists are mutable.

# Explanation: `count` is an int, which is immutable — `counter += 1` inside
# the function creates a brand-new int object and rebinds the LOCAL name
# `counter` to it, leaving the caller's `count` untouched. `count_list` is a
# list — mutable — so `counter_list[0] += 1` modifies the actual object in
# memory that both `count_list` and `counter_list` point to, so the change
# is visible after the function returns.
```
</details>

---

**Previous lesson:** [1.6 — Functions](lesson-1.6-functions.md)
**Next lesson:** [1.8 — Error Handling Basics](lesson-1.8-error-handling.md)
