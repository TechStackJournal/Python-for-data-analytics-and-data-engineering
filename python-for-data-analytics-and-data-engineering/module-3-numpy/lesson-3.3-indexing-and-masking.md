# Lesson 3.3 — Indexing, Slicing & Boolean Masking

## CONCEPT

**Indexing** and **slicing** NumPy arrays work similarly to Python lists (Module 2) for single dimensions, but extend naturally to multiple dimensions. **Boolean masking** is the technique unique to (and central in) NumPy: using an array of `True`/`False` values to select only the elements that meet a condition — no explicit loop required.

```python
import numpy as np

prices = np.array([24.99, 12.50, 89.00, 34.00, 199.99])
prices[0]                # 24.99 — single element
prices[1:3]               # [12.5 89. ] — slice
prices > 50                # [False False  True False  True] — a boolean mask
prices[prices > 50]        # [ 89.   199.99] — elements matching the mask
```

This last line — filtering an array using a condition built from itself — is one of the single most important patterns in all of NumPy and pandas.

## WHY IT MATTERS

"Give me only the orders over $100" or "flag every negative price" are filtering operations you'll perform constantly. Boolean masking replaces the `for`/`if` loop pattern from Module 1 with a single vectorized expression — faster, and arguably more readable once it clicks. This exact syntax reappears nearly unchanged as `df[df["price"] > 100]` in pandas (Module 4) — mastering it here means Module 4's filtering syntax won't feel new at all.

## SYNTAX REFERENCE

```python
import numpy as np

arr = np.array([10, 20, 30, 40, 50])

# 1D indexing and slicing (same rules as Python lists)
arr[0]           # 10 — first element
arr[-1]           # 50 — last element
arr[1:3]           # [20 30] — slice, stop index exclusive
arr[:3]             # [10 20 30] — from the start
arr[2:]              # [30 40 50] — to the end

# 2D indexing
grid = np.array([[1, 2, 3], [4, 5, 6], [7, 8, 9]])
grid[0]               # [1 2 3] — entire first row
grid[0, 1]             # 2 — row 0, column 1
grid[:, 1]              # [2 5 8] — entire second column (all rows, column 1)
grid[0:2, 1:3]           # [[2 3] [5 6]] — a sub-grid slice

# Boolean masking
arr > 25                    # array([False, False,  True,  True,  True])
arr[arr > 25]                 # array([30, 40, 50]) — only matching elements

# Combining conditions — use & (and), | (or), ~ (not) — NOT the keywords and/or
arr[(arr > 15) & (arr < 45)]    # array([20, 30, 40])
arr[(arr < 20) | (arr > 45)]     # array([10, 50])
arr[~(arr > 25)]                  # array([10, 20]) — everything NOT matching

# Modifying values via a mask (in place)
arr[arr > 25] = 0
print(arr)   # [10 20  0  0  0]

# Getting the count/indices of matches
np.sum(arr > 25)          # count of True values (True counts as 1)
np.where(arr > 25)          # tuple of index positions where True
```

## WORKED EXAMPLE

Filtering and flagging order data using boolean masks — no explicit loops:

```python
import numpy as np

order_totals = np.array([45.99, 120.50, 89.99, 15.00, 200.00, 65.50, 8.25])
is_prime_member = np.array([True, False, True, True, False, True, False])

# Which orders are over $100?
large_orders_mask = order_totals > 100
print(large_orders_mask)
# [False  True False False  True False False]

large_orders = order_totals[large_orders_mask]
print(large_orders)
# [120.5 200. ]

# Combine two conditions: large orders from non-prime members (candidates for upsell)
upsell_candidates_mask = (order_totals > 50) & (~is_prime_member)
print(order_totals[upsell_candidates_mask])
# [120.5]

# Count how many orders are "small" (under $20)
small_order_count = np.sum(order_totals < 20)
print(f"Small orders: {small_order_count}")   # Small orders: 2

# Find the INDEX positions of orders that qualify for free shipping (prime AND over $30)
free_shipping_mask = is_prime_member & (order_totals > 30)
qualifying_indices = np.where(free_shipping_mask)[0]
print(f"Order indices qualifying for free shipping: {qualifying_indices}")
# [0 2 5]

# Modify values in place: cap any order total above $150 at exactly $150 (e.g. a promo rule)
capped_totals = order_totals.copy()   # copy first, so we don't lose the original data
capped_totals[capped_totals > 150] = 150
print(capped_totals)
# [ 45.99 120.5   89.99  15.    150.    65.5    8.25]
```

## COMMON MISTAKES

**1. Using Python's `and`/`or` instead of `&`/`|` for combining array conditions**
```python
arr[(arr > 10) and (arr < 50)]   # ValueError — Python's `and` doesn't work element-wise on arrays
arr[(arr > 10) & (arr < 50)]     # correct — & works element-wise
```
Fix: always use `&`, `|`, and `~` for boolean array logic — and always wrap each condition in parentheses, since these operators have different precedence than `and`/`or`.

**2. Forgetting parentheses around each condition when combining with `&`/`|`**
```python
arr[arr > 10 & arr < 50]      # TypeError or wrong result — operator precedence issue
arr[(arr > 10) & (arr < 50)]  # correct
```

**3. Accidentally mutating the original array when you meant to keep it unchanged**
```python
capped = order_totals
capped[capped > 150] = 150     # this ALSO changed order_totals! Same underlying array.
```
Fix: use `.copy()` explicitly when you need an independent copy — simple assignment (`capped = order_totals`) creates another reference to the *same* array, not a new one (this mirrors the mutable-object behavior covered in Lesson 1.7).

**4. Confusing `grid[0, 1]` (2D single-element indexing) with `grid[0][1]`**
Both usually produce the same result, but `grid[0, 1]` is the idiomatic NumPy way and is more efficient — `grid[0][1]` first creates an intermediate 1D array (`grid[0]`) and then indexes into that, doing extra unnecessary work.

**5. Expecting `np.where(condition)` to return the matching values directly**
```python
np.where(arr > 25)   # returns (array([2, 3, 4]),) — the INDICES, not the values!
```
Fix: to get the values themselves, use boolean masking directly (`arr[arr > 25]`); use `np.where()` specifically when you need to know *where* (which positions) the matches occurred.

**Data Engineer Tip:** The upsell-candidates example in the worked example — combining two boolean conditions with `&` — is exactly the logic behind a `WHERE total > 50 AND is_prime = false` SQL clause or a `df[(df.total > 50) & (~df.is_prime)]` pandas filter. All three express identical logic; only the syntax differs. Getting fluent with boolean masks here means you're really learning one concept three times, in three different accents.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `arr = np.array([5, 12, 8, 20, 3, 17])`, use a boolean mask to select only the values greater than 10, and print the result.

<details>
<summary>Hint</summary>
`arr[arr > 10]`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

arr = np.array([5, 12, 8, 20, 3, 17])
result = arr[arr > 10]
print(result)   # [12 20 17]
```
</details>

---

**Exercise 2 (Easy):** Given `grid = np.array([[1, 2, 3], [4, 5, 6], [7, 8, 9]])`, print the entire second row and the entire third column.

<details>
<summary>Hint</summary>
Row: `grid[1]`. Column: `grid[:, 2]`.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

grid = np.array([[1, 2, 3], [4, 5, 6], [7, 8, 9]])
print(grid[1])      # [4 5 6]
print(grid[:, 2])    # [3 6 9]
```
</details>

---

**Exercise 3 (Medium):** Given `prices = np.array([9.99, 45.00, 120.00, 18.50, 89.99, 5.00])`, select only the prices that are between $10 and $100 (inclusive on both ends), and print the result.

<details>
<summary>Hint</summary>
Combine two conditions with `&`, each in parentheses.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

prices = np.array([9.99, 45.00, 120.00, 18.50, 89.99, 5.00])
mid_range = prices[(prices >= 10) & (prices <= 100)]
print(mid_range)   # [45.   18.5  89.99]
```
</details>

---

**Exercise 4 (Medium):** Given `stock_counts = np.array([0, 15, 3, 0, 42, 0, 8])`, count how many products are out of stock (count of zeros), then use `np.where()` to find the index positions of the out-of-stock items. Print both.

<details>
<summary>Hint</summary>
`np.sum(stock_counts == 0)` for the count; `np.where(stock_counts == 0)` for positions.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

stock_counts = np.array([0, 15, 3, 0, 42, 0, 8])

out_of_stock_count = np.sum(stock_counts == 0)
out_of_stock_indices = np.where(stock_counts == 0)[0]

print("Out of stock count:", out_of_stock_count)       # 3
print("Out of stock indices:", out_of_stock_indices)     # [0 3 5]
```
</details>

---

**Exercise 5 (Hard):** Given two arrays representing order data — `order_totals = np.array([45.99, 220.00, 12.50, 89.99, 340.00, 15.00])` and `customer_tiers = np.array(["silver", "gold", "silver", "gold", "gold", "bronze"])` — do the following without any explicit `for` loop:
1. Create a copy of `order_totals` and apply a 20% discount (multiply by 0.8) to only the orders from `"gold"` tier customers, leaving all others unchanged.
2. Find how many orders, after the discount, are still over $200.
3. Print the discounted array and both counts.

<details>
<summary>Hint</summary>
Build the gold-tier mask first (`customer_tiers == "gold"`), copy the original array, apply the discount only to the masked positions with `arr[mask] *= 0.8`, then build a second mask on the *discounted* array for the over-$200 check.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

order_totals = np.array([45.99, 220.00, 12.50, 89.99, 340.00, 15.00])
customer_tiers = np.array(["silver", "gold", "silver", "gold", "gold", "bronze"])

discounted = order_totals.copy()
gold_mask = customer_tiers == "gold"
discounted[gold_mask] = discounted[gold_mask] * 0.8

still_over_200 = np.sum(discounted > 200)

print("Discounted totals:", discounted)
# [ 45.99 176.    12.5   71.992 272.    15.   ]
print("Gold-tier orders discounted:", np.sum(gold_mask))   # 3
print("Still over $200 after discount:", still_over_200)     # 1
```
</details>

---

**Previous lesson:** [3.2 — Creating Arrays, Shapes & Dtypes](lesson-3.2-creating-arrays.md)
**Next lesson:** [3.4 — Vectorized Operations & Broadcasting](lesson-3.4-vectorization-and-broadcasting.md)
