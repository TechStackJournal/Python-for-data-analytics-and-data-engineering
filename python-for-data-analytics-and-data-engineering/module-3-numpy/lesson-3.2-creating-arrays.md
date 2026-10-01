# Lesson 3.2 — Creating Arrays, Shapes & Dtypes

## CONCEPT

Beyond converting a list with `np.array()`, NumPy provides several built-in functions to generate arrays directly — ranges of numbers, arrays of zeros/ones, evenly-spaced values. Every array also has a **shape** (its dimensions — how many rows, columns, etc.) and a **dtype** (the data type of its elements), both essential for understanding what you're working with, especially once arrays become multi-dimensional (like a table of data).

```python
import numpy as np

arr = np.array([[1, 2, 3], [4, 5, 6]])   # a 2D array — 2 rows, 3 columns
arr.shape   # (2, 3)
arr.dtype   # dtype('int64')
```

## WHY IT MATTERS

A dataset is naturally two-dimensional — rows and columns. Understanding shape and dtype now is a direct prerequisite for understanding pandas DataFrames in Module 4, which are essentially 2D NumPy arrays with row/column labels attached. Knowing how to *generate* arrays (not just convert existing lists) is also essential for creating test data, initializing placeholder arrays, and building the synthetic datasets you'll use throughout this course for practice.

## SYNTAX REFERENCE

```python
import numpy as np

# From a list (1D)
np.array([1, 2, 3])

# From a list of lists (2D)
np.array([[1, 2, 3], [4, 5, 6]])

# Ranges
np.arange(5)                # [0 1 2 3 4]
np.arange(1, 10, 2)          # [1 3 5 7 9] — start, stop (exclusive), step
np.linspace(0, 1, 5)         # [0.   0.25 0.5  0.75 1.  ] — 5 evenly-spaced values from 0 to 1

# Filled arrays
np.zeros(5)                  # [0. 0. 0. 0. 0.]
np.ones(5)                   # [1. 1. 1. 1. 1.]
np.zeros((3, 4))             # a 3x4 array of zeros
np.full(5, 7)                 # [7 7 7 7 7]

# Shape and dimensions
arr = np.array([[1, 2, 3], [4, 5, 6]])
arr.shape        # (2, 3) — (rows, columns)
arr.ndim          # 2 — number of dimensions
arr.size           # 6 — total number of elements

# Reshaping
flat = np.arange(6)             # [0 1 2 3 4 5]
flat.reshape(2, 3)               # [[0 1 2] [3 4 5]]
flat.reshape(3, 2)               # [[0 1] [2 3] [4 5]]

# Specifying dtype explicitly
np.array([1, 2, 3], dtype=float)     # force float instead of int
np.array([1.5, 2.5], dtype=int)      # force int — truncates decimals!

# Converting dtype after creation
arr.astype(float)
arr.astype(int)
```

## WORKED EXAMPLE

Building and inspecting arrays representing a small batch of product data:

```python
import numpy as np

# Generate a range of synthetic product IDs
product_ids = np.arange(1001, 1011)   # 1001 through 1010
print(product_ids)
# [1001 1002 1003 1004 1005 1006 1007 1008 1009 1010]

# Prices as a 1D array — inspect shape and dtype
prices = np.array([24.99, 12.50, 89.00, 34.00, 199.99, 15.75, 45.00, 9.99, 120.00, 67.50])
print("Shape:", prices.shape)    # (10,)
print("Dtype:", prices.dtype)     # float64
print("Size:", prices.size)        # 10

# A 2D array: rows = products, columns = [price, quantity]
inventory = np.array([
    [24.99, 100],
    [12.50, 250],
    [89.00, 30],
    [34.00, 75],
])
print("Shape:", inventory.shape)   # (4, 2) — 4 products, 2 attributes each
print("Row 0 (first product):", inventory[0])   # [24.99 100.]

# Note: mixing float prices and int quantities in one array forces BOTH to float
print("Dtype:", inventory.dtype)    # float64 — the int column got upcast

# Initializing a placeholder array before filling it with real data
daily_totals = np.zeros(7)   # one slot per day of the week
print(daily_totals)           # [0. 0. 0. 0. 0. 0. 0.]

# Reshaping a flat array of 12 monthly totals into a 3x4 grid (e.g. quarters x months)
monthly_revenue = np.arange(12) * 1000.0   # placeholder synthetic values
quarterly_view = monthly_revenue.reshape(4, 3)
print(quarterly_view)
# [[    0.  1000.  2000.]
#  [ 3000.  4000.  5000.]
#  [ 6000.  7000.  8000.]
#  [ 9000. 10000. 11000.]]
```

## COMMON MISTAKES

**1. Forgetting `reshape()` requires the total number of elements to match exactly**
```python
arr = np.arange(10)
arr.reshape(3, 3)   # ValueError — 10 elements can't fill a 3x3 (9-element) shape
```
Fix: the product of the new shape's dimensions must equal the original array's `.size`.

**2. Confusing `.shape` (a tuple) with `.size` (a single number)**
```python
arr = np.zeros((3, 4))
arr.shape   # (3, 4) — the dimensions
arr.size     # 12 — the total element count (3 * 4)
```

**3. Losing data silently by casting float to int**
```python
np.array([1.9, 2.5, 3.1], dtype=int)   # [1 2 3] — truncated, NOT rounded!
```
Fix: if you want rounding behavior, round first (`np.round(arr)`) before converting to int — casting directly truncates toward zero.

**4. Assuming a 2D array with mixed int/float columns keeps each column's original type**
As shown in the worked example, a single NumPy array can only have ONE dtype for the whole array — mixing an int-looking column with a float-looking column upcasts everything to float. This is a key limitation NumPy has that pandas DataFrames (Module 4) solve, since a DataFrame lets each column keep its own independent dtype.

**5. Using `np.arange()` with float steps and expecting exact values**
```python
np.arange(0, 1, 0.1)   # may include unexpected floating-point imprecision at the boundaries
```
Fix: `np.linspace(0, 1, 11)` is usually a more reliable choice than `np.arange()` when you need a specific *count* of evenly-spaced float values, since it doesn't accumulate floating-point rounding error the way repeated addition can.

**Data Engineer Tip:** The mixed-dtype limitation in Common Mistake #4 is precisely *why* pandas DataFrames exist as a separate structure from a raw 2D NumPy array — a DataFrame is really a collection of 1D NumPy arrays (one per column), each free to have its own dtype, presented together as a table. Understanding this now means the distinction between `df.dtypes` (per-column) and a single NumPy array's `.dtype` (whole-array) in Module 4 will make immediate sense instead of feeling arbitrary.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create an array of the numbers 0 through 9 using `np.arange()`, and print its `shape` and `dtype`.

<details>
<summary>Hint</summary>
`np.arange(10)`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

arr = np.arange(10)
print(arr.shape)   # (10,)
print(arr.dtype)     # int64
```
</details>

---

**Exercise 2 (Easy):** Create a 5-element array of all zeros, then a 5-element array of all `7`s using `np.full()`. Print both.

<details>
<summary>Hint</summary>
`np.zeros(5)` and `np.full(5, 7)`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

zeros = np.zeros(5)
sevens = np.full(5, 7)
print(zeros)     # [0. 0. 0. 0. 0.]
print(sevens)    # [7 7 7 7 7]
```
</details>

---

**Exercise 3 (Medium):** Create a flat array of the numbers 1 through 12 using `np.arange()`, then reshape it into a 3x4 array (3 rows, 4 columns). Print the reshaped array and confirm its `.shape` is `(3, 4)`.

<details>
<summary>Hint</summary>
`np.arange(1, 13).reshape(3, 4)`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

arr = np.arange(1, 13)
reshaped = arr.reshape(3, 4)
print(reshaped)
print(reshaped.shape)   # (3, 4)
```
</details>

---

**Exercise 4 (Medium):** Given `raw_prices = ["24.99", "12.50", "89.00"]` (strings), create a NumPy array with `dtype=float` directly from the list (without converting each element manually first), and print the result along with its dtype.

<details>
<summary>Hint</summary>
`np.array(raw_prices, dtype=float)` — NumPy will convert each string during array creation.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

raw_prices = ["24.99", "12.50", "89.00"]
prices = np.array(raw_prices, dtype=float)
print(prices)        # [24.99 12.5  89.  ]
print(prices.dtype)   # float64
```
</details>

---

**Exercise 5 (Hard):** Generate a 2D array representing 4 weeks of daily sales (7 days each) using `np.arange(28).reshape(4, 7)` as placeholder data. Then: (1) print the shape, confirming it's `(4, 7)`, (2) extract and print just the first week (row 0), (3) reshape the *entire* array into a completely flat 1D array of 28 values using `.reshape(-1)` (explain in a comment what `-1` does here), and (4) confirm the flattened array's `.size` still equals 28.

<details>
<summary>Hint</summary>
Row 0 access is `arr[0]`. For flattening, `.reshape(-1)` tells NumPy to figure out the single dimension automatically based on the total size — a common shorthand instead of computing it yourself.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

sales = np.arange(28).reshape(4, 7)
print("Shape:", sales.shape)          # (4, 7)

week_1 = sales[0]
print("Week 1:", week_1)              # [0 1 2 3 4 5 6]

flattened = sales.reshape(-1)
# -1 tells NumPy "figure out this dimension automatically" — since the
# array has 28 total elements, reshape(-1) produces a single 1D array
# of length 28 without needing to type 28 explicitly.
print("Flattened:", flattened)
print("Size:", flattened.size)         # 28
```
</details>

---

**Previous lesson:** [3.1 — Arrays vs. Python Lists](lesson-3.1-arrays-vs-lists.md)
**Next lesson:** [3.3 — Indexing, Slicing & Boolean Masking](lesson-3.3-indexing-and-masking.md)
