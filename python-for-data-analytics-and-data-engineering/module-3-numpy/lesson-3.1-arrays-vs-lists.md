# Lesson 3.1 — Arrays vs. Python Lists

## CONCEPT

A NumPy **array** (`ndarray`) looks similar to a Python list on the surface — an ordered collection of values — but underneath it's fundamentally different. A list can hold mixed types (`[1, "two", 3.0]`) and stores each element as a separate Python object scattered in memory. A NumPy array requires **all elements to be the same type** and stores them in one contiguous, fixed-size block of memory — which is exactly what makes it fast.

```python
import numpy as np

prices_list = [24.99, 12.50, 89.00]        # Python list
prices_array = np.array([24.99, 12.50, 89.00])   # NumPy array
```

They look almost identical here — the difference becomes obvious the moment you try to do math on them.

## WHY IT MATTERS

Data analytics is fundamentally numerical computation over large collections — thousands or millions of prices, quantities, ratings. A `for` loop processing each value individually (Module 1's approach) is fine for small lists, but becomes a real bottleneck at scale. NumPy arrays let you express "multiply every price by 1.1" as a single operation that runs in optimized, compiled C code under the hood — often 10–100x faster than the equivalent Python loop. This is the single most important mental shift this module makes: stop thinking "for each value, do X" and start thinking "do X to the whole array at once."

## SYNTAX REFERENCE

```python
import numpy as np

# Creating an array from a list
arr = np.array([1, 2, 3, 4, 5])

# The key difference: math on lists vs. arrays
python_list = [1, 2, 3]
python_list * 2          # [1, 2, 3, 1, 2, 3] — repeats the list!

numpy_array = np.array([1, 2, 3])
numpy_array * 2           # array([2, 4, 6]) — multiplies EACH element

# Element-wise arithmetic works directly on arrays
numpy_array + 10          # array([11, 12, 13])
numpy_array ** 2           # array([1, 4, 9])

# Checking type and dtype
type(numpy_array)          # <class 'numpy.ndarray'>
numpy_array.dtype           # dtype('int64') — the shared type of every element

# Arrays enforce a single dtype (mixed input gets upcast)
np.array([1, 2, 3.5])       # array([1. , 2. , 3.5]) — all became floats
np.array([1, "two", 3])      # array(['1', 'two', '3']) — all became strings!
```

## WORKED EXAMPLE

Comparing the loop-based approach from Module 1 against the vectorized NumPy equivalent, on the same task: applying a 10% price increase to a list of product prices.

```python
import numpy as np

prices = [24.99, 12.50, 89.00, 34.00, 199.99]

# --- The Module 1 way: explicit loop ---
increased_loop = []
for price in prices:
    increased_loop.append(round(price * 1.10, 2))
print(increased_loop)
# [27.49, 13.75, 97.9, 37.4, 219.99]

# --- The NumPy way: vectorized, no explicit loop ---
prices_array = np.array(prices)
increased_array = np.round(prices_array * 1.10, 2)
print(increased_array)
# [ 27.49  13.75  97.9   37.4  219.99]

# Both produce the same values — but the array version scales dramatically
# better as the dataset grows, because the loop is implemented in optimized C
# code inside NumPy rather than Python's slower interpreted loop.

# A quick, informal timing comparison on a much larger dataset:
import time

large_list = list(range(1_000_000))
large_array = np.array(large_list)

start = time.time()
doubled_loop = [x * 2 for x in large_list]
loop_time = time.time() - start

start = time.time()
doubled_array = large_array * 2
array_time = time.time() - start

print(f"Loop time: {loop_time:.4f}s")
print(f"NumPy time: {array_time:.4f}s")
# NumPy is typically 10-50x faster for operations like this at this scale
```

## COMMON MISTAKES

**1. Expecting `*` on a list to behave like element-wise multiplication**
```python
[1, 2, 3] * 2       # [1, 2, 3, 1, 2, 3] — list repetition, NOT multiplication!
np.array([1, 2, 3]) * 2   # array([2, 4, 6]) — this is what you probably wanted
```
This is one of the most common sources of confusion switching from lists to arrays — the same operator means something completely different for each type.

**2. Mixing types inside an array without realizing NumPy silently converts everything**
```python
np.array([1, 2, "3"])   # array(['1', '2', '3'], dtype='<U21') — ALL became strings!
```
Fix: check `.dtype` after creating an array from mixed or uncertain data, especially when it came from an external source like a CSV.

**3. Converting back and forth between lists and arrays unnecessarily**
```python
for i in range(len(arr)):
    result_list.append(arr[i] * 2)   # defeats the purpose — you're back to a slow loop
result_array = np.array(result_list)
```
Fix: if you're working with arrays, stay in vectorized operations (`arr * 2`) rather than looping and converting back — looping over a NumPy array element-by-element is actually *slower* than looping over a plain Python list, because of the overhead of accessing individual NumPy elements.

**4. Assuming an array can grow like a list**
```python
arr = np.array([1, 2, 3])
arr.append(4)   # AttributeError — arrays don't have .append()
```
Fix: NumPy arrays have a fixed size once created. Growing one means creating a new array (`np.append(arr, 4)`, which actually copies the whole array) — for data that changes size frequently, a Python list is often still the better tool, converting to an array only once the size is stable.

**Data Engineer Tip:** A pandas DataFrame column is a NumPy array with a label attached — this is literally what's happening under the hood. When you write `df["price"] * 1.1` in Module 4, you're triggering the exact same vectorized array multiplication shown here, just through pandas' interface. Understanding *why* this is fast (contiguous memory, single dtype, compiled operations) explains why pandas strongly discourages `.apply()` with a custom Python function for simple operations — it forces pandas back into slow, element-by-element Python execution.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create a NumPy array from the list `[10, 20, 30, 40]` and print its `dtype`.

<details>
<summary>Hint</summary>
`np.array([10, 20, 30, 40])`, then `.dtype`.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

arr = np.array([10, 20, 30, 40])
print(arr.dtype)   # int64 (or int32 on some systems)
```
</details>

---

**Exercise 2 (Easy):** Given a NumPy array `prices = np.array([9.99, 19.99, 29.99])`, add a flat $5 shipping fee to every price using a single vectorized operation, and print the result.

<details>
<summary>Hint</summary>
`prices + 5`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

prices = np.array([9.99, 19.99, 29.99])
with_shipping = prices + 5
print(with_shipping)   # [14.99 24.99 34.99]
```
</details>

---

**Exercise 3 (Medium):** Given `quantities = [1, 2, "3", 4]` (note the string `"3"`), create a NumPy array from it and print its `.dtype`. Explain in a comment what happened and why.

<details>
<summary>Hint</summary>
Mixed types force NumPy to upcast everything to a common type — with a string present, everything becomes a string.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

quantities = [1, 2, "3", 4]
arr = np.array(quantities)
print(arr)          # ['1' '2' '3' '4']
print(arr.dtype)     # <U21 (a string type)

# All elements were converted to strings because NumPy arrays require
# a single shared dtype — since one element was already a string, everything
# got upcast to match it rather than raising an error.
```
</details>

---

**Exercise 4 (Medium):** Given a Python list `discounts_pct = [10, 20, 0, 15]` representing percentage discounts, convert it to a NumPy array and compute the multiplier for each (e.g. 10% discount → 0.90 multiplier) in one vectorized expression. Print the result.

<details>
<summary>Hint</summary>
`1 - (arr / 100)`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

discounts_pct = np.array([10, 20, 0, 15])
multipliers = 1 - (discounts_pct / 100)
print(multipliers)   # [0.9  0.8  1.   0.85]
```
</details>

---

**Exercise 5 (Hard):** Given `unit_prices = [24.99, 12.50, 89.00, 34.00, 199.99]` and `quantities = [2, 5, 1, 3, 1]` as plain Python lists, write **both** a loop-based version and a NumPy vectorized version that compute the line total for each item (`price * quantity`) and then the grand total across all items. Confirm both approaches produce the same grand total (rounded to 2 decimals).

<details>
<summary>Hint</summary>
For the loop version, use `zip()` (from earlier modules) to pair prices with quantities. For the NumPy version, convert both lists to arrays first, multiply them element-wise directly (arrays of the same shape multiply position-by-position), then use `.sum()`.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

unit_prices = [24.99, 12.50, 89.00, 34.00, 199.99]
quantities = [2, 5, 1, 3, 1]

# --- Loop version ---
line_totals_loop = []
for price, qty in zip(unit_prices, quantities):
    line_totals_loop.append(price * qty)
grand_total_loop = round(sum(line_totals_loop), 2)

# --- NumPy version ---
prices_arr = np.array(unit_prices)
qty_arr = np.array(quantities)
line_totals_array = prices_arr * qty_arr   # element-wise multiplication
grand_total_array = round(line_totals_array.sum(), 2)

print("Loop grand total:", grand_total_loop)     # 484.47
print("NumPy grand total:", grand_total_array)     # 484.47
print("Match:", grand_total_loop == grand_total_array)   # True
```
</details>

---

**Next lesson:** [3.2 — Creating Arrays, Shapes & Dtypes](lesson-3.2-creating-arrays.md)
