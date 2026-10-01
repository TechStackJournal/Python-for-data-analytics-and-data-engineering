# Lesson 3.4 — Vectorized Operations & Broadcasting

## CONCEPT

**Vectorization** means applying an operation to an entire array at once instead of looping element-by-element — you've already been doing this since Lesson 3.1 (`prices * 1.1`). **Broadcasting** is the specific set of rules NumPy uses to apply operations between arrays of *different* (but compatible) shapes — like adding a single number to every element of an array, or adding a 1D array of column adjustments to every row of a 2D array.

```python
import numpy as np

prices = np.array([24.99, 12.50, 89.00])
prices + 5          # broadcasting: the single value 5 is applied to every element
# array([29.99, 17.5 , 94.  ])
```

Broadcasting is what makes NumPy code so concise — it "stretches" the smaller array to match the shape of the larger one, without actually copying any data, following a consistent set of rules.

## WHY IT MATTERS

Real datasets require operations that mix different shapes constantly: applying a single tax rate to every row, adding a per-column adjustment to every row of a table, comparing every row against a single threshold. Understanding broadcasting rules means you can predict exactly what an operation will do (and get a clear error, rather than a silent wrong answer, when shapes genuinely don't match) — a skill directly reused when pandas applies the same rules to DataFrame + scalar or DataFrame + Series operations in Module 4.

## SYNTAX REFERENCE

```python
import numpy as np

# Scalar broadcasting — a single number applied to every element
arr = np.array([1, 2, 3])
arr + 10           # [11 12 13]
arr * 2             # [2 4 6]

# Array + array of the SAME shape — element-wise, position by position
a = np.array([1, 2, 3])
b = np.array([10, 20, 30])
a + b                # [11 22 33]

# Broadcasting a 1D array against each row of a 2D array
grid = np.array([[1, 2, 3], [4, 5, 6], [7, 8, 9]])
adjustments = np.array([10, 20, 30])   # shape (3,) — matches grid's number of COLUMNS
grid + adjustments
# [[11 22 33]
#  [14 25 36]
#  [17 28 39]]
# adjustments is "stretched" down to apply to every row

# Broadcasting rules (simplified): shapes are compatible if, comparing from the
# RIGHT, each dimension is either equal, or one of them is 1 (or missing)
# (3,)     and (3,)    -> compatible, element-wise
# (3, 3)   and (3,)    -> compatible, the 1D array applies to every row
# (3, 3)   and (3, 1)  -> compatible, applies to every column instead
# (3, 3)   and (4,)    -> NOT compatible — shapes don't align, raises an error

# Universal functions ("ufuncs") — vectorized math functions
np.sqrt(arr)
np.round(arr, 2)
np.abs(arr)
np.exp(arr)
np.log(arr)

# Element-wise comparison between two arrays of the same shape
a == b                  # array of True/False, one per position
np.array_equal(a, b)     # single True/False — are the ENTIRE arrays identical?
```

## WORKED EXAMPLE

Applying tax and per-category discount adjustments to a small pricing table using broadcasting:

```python
import numpy as np

# A single tax rate applied to every price — scalar broadcasting
prices = np.array([24.99, 12.50, 89.00, 34.00])
tax_rate = 0.08

prices_with_tax = prices * (1 + tax_rate)
print(np.round(prices_with_tax, 2))
# [26.99 13.5  96.12 36.72]

# A 2D table: rows = products, columns = [base_price, shipping_cost, handling_fee]
cost_breakdown = np.array([
    [24.99, 5.00, 1.50],
    [12.50, 5.00, 1.50],
    [89.00, 8.00, 2.00],
    [34.00, 5.00, 1.50],
])

# Sum each row to get the total cost per product — broadcasting isn't even
# needed here, this uses an aggregation along an axis (more in Lesson 3.5)
row_totals = cost_breakdown.sum(axis=1)
print(row_totals)
# [ 31.49  19.    99.    40.5 ]

# Now broadcast a per-column adjustment: suppose shipping costs are going up
# by a flat $2, and handling fees by $0.50 — but base price is unchanged
adjustments = np.array([0, 2.00, 0.50])   # shape (3,) matches the 3 columns
adjusted_costs = cost_breakdown + adjustments
print(adjusted_costs)
# [[24.99  7.    2.  ]
#  [12.5   7.    2.  ]
#  [89.   10.    2.5 ]
#  [34.    7.    2.  ]]
# The (3,) adjustments array was broadcast down every one of the 4 rows

# A broadcasting shape mismatch — this will raise an error, shown for learning
mismatched = np.array([1, 2])   # shape (2,) — doesn't match cost_breakdown's 3 columns
try:
    cost_breakdown + mismatched
except ValueError as e:
    print(f"Broadcasting error: {e}")
# Broadcasting error: operands could not be broadcast together with shapes (4,3) (2,)
```

## COMMON MISTAKES

**1. Assuming any two arrays can be added together**
```python
a = np.array([1, 2, 3])
b = np.array([1, 2])
a + b   # ValueError: operands could not be broadcast together with shapes (3,) (2,)
```
Fix: broadcasting has specific compatibility rules (shown in the Syntax Reference) — shapes that don't align from the right, with matching or `1`-sized dimensions, simply won't work, and NumPy raises a clear error rather than guessing.

**2. Confusing element-wise multiplication (`*`) with matrix multiplication**
```python
a = np.array([[1, 2], [3, 4]])
b = np.array([[5, 6], [7, 8]])
a * b     # element-wise: [[5, 12], [21, 32]]
a @ b     # matrix multiplication: [[19, 22], [43, 50]] — a DIFFERENT operation
```
For this course's purposes (data analytics, not linear algebra), you'll almost always want `*` (element-wise) — just be aware `@` exists and means something different, since it's easy to encounter in examples online without realizing the distinction.

**3. Broadcasting silently succeeding with an unintended shape**
```python
grid = np.zeros((3, 3))
row = np.array([1, 2, 3])      # shape (3,) — broadcasts across ROWS (as intended)
col = np.array([[1], [2], [3]])  # shape (3, 1) — broadcasts across COLUMNS instead!
```
Fix: a plain 1D array of shape `(3,)` and a "column" array of shape `(3, 1)` broadcast completely differently against a 2D grid — if the result looks unexpected, print `.shape` on every array involved to confirm what's actually happening.

**4. Forgetting broadcasting doesn't modify either original array**
```python
result = prices + 5   # creates a NEW array; `prices` itself is unchanged
```
This is standard NumPy behavior — most operations return new arrays rather than modifying in place, *unless* you specifically use an in-place operator like `arr += 5` or indexed assignment like `arr[mask] = value` (seen in Lesson 3.3).

**Data Engineer Tip:** When pandas applies `df["total"] = df["price"] * df["quantity"]`, it's relying on the same element-wise, same-shape broadcasting rule shown here — both columns must have the same number of rows for this to work cleanly. When you later see a pandas error about "unable to align" two Series, it's the exact same underlying broadcasting rule refusing to guess how to combine mismatched shapes — now you'll know why, instead of it feeling like an arbitrary pandas quirk.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `prices = np.array([10.00, 25.00, 40.00])`, add a flat $3 handling fee to every price using broadcasting, and print the result.

<details>
<summary>Hint</summary>
`prices + 3`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

prices = np.array([10.00, 25.00, 40.00])
with_fee = prices + 3
print(with_fee)   # [13. 28. 43.]
```
</details>

---

**Exercise 2 (Easy):** Given `values = np.array([4, 9, 16, 25])`, use `np.sqrt()` to compute the square root of every element, and print the result.

<details>
<summary>Hint</summary>
`np.sqrt(values)`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

values = np.array([4, 9, 16, 25])
roots = np.sqrt(values)
print(roots)   # [2. 3. 4. 5.]
```
</details>

---

**Exercise 3 (Medium):** Given a 2D array `sales = np.array([[100, 200, 150], [80, 120, 90]])` representing 2 stores' sales across 3 products, broadcast-add a per-product promotional bonus `bonus = np.array([10, 0, 5])` to every row, and print the result.

<details>
<summary>Hint</summary>
`sales + bonus` — `bonus` has shape `(3,)`, matching the number of columns.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

sales = np.array([[100, 200, 150], [80, 120, 90]])
bonus = np.array([10, 0, 5])

result = sales + bonus
print(result)
# [[110 200 155]
#  [ 90 120  95]]
```
</details>

---

**Exercise 4 (Medium):** Given `a = np.array([1, 2, 3])` and `b = np.array([1, 2, 4])`, use element-wise comparison to find which positions differ, and use `np.array_equal()` to check if the two arrays are entirely identical. Print both results.

<details>
<summary>Hint</summary>
`a == b` for positional comparison; `np.array_equal(a, b)` for a single overall answer.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

a = np.array([1, 2, 3])
b = np.array([1, 2, 4])

print(a == b)                  # [ True  True False]
print(np.array_equal(a, b))     # False
```
</details>

---

**Exercise 5 (Hard):** Given a 2D array representing 3 products' pricing across 4 regions (rows = products, columns = regions): `base_prices = np.array([[20.0, 20.0, 20.0, 20.0], [50.0, 50.0, 50.0, 50.0], [15.0, 15.0, 15.0, 15.0]])`, and a per-region tax rate array `region_tax_rates = np.array([0.05, 0.08, 0.10, 0.0])` (one rate per column/region):
1. Compute the final price for every product in every region, applying the correct regional tax rate (final = base * (1 + tax_rate)) using a single broadcasting operation.
2. Print the resulting 3x4 array, rounded to 2 decimals.
3. Identify (using boolean masking) which product/region combinations have a final price over $50, and print the count.

<details>
<summary>Hint</summary>
`base_prices * (1 + region_tax_rates)` broadcasts the `(4,)` tax array across each of the 3 rows automatically. For the count, build a boolean mask on the final array and use `.sum()`.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

base_prices = np.array([
    [20.0, 20.0, 20.0, 20.0],
    [50.0, 50.0, 50.0, 50.0],
    [15.0, 15.0, 15.0, 15.0],
])
region_tax_rates = np.array([0.05, 0.08, 0.10, 0.0])

final_prices = np.round(base_prices * (1 + region_tax_rates), 2)
print(final_prices)
# [[21.   21.6  22.   20.  ]
#  [52.5  54.   55.   50.  ]
#  [15.75 16.2  16.5  15.  ]]

over_50_count = np.sum(final_prices > 50)
print("Combinations over $50:", over_50_count)   # 3
```
</details>

---

**Previous lesson:** [3.3 — Indexing, Slicing & Boolean Masking](lesson-3.3-indexing-and-masking.md)
**Next lesson:** [3.5 — Aggregations](lesson-3.5-aggregations.md)
