# Lesson 3.7 — Random Sampling

## CONCEPT

NumPy's `random` module generates random numbers and samples — essential for creating realistic synthetic test data, simulating scenarios, and randomly splitting datasets (a technique you'll use constantly once you reach machine learning topics). Modern NumPy code uses a **Generator** object rather than older, global-state random functions, because it produces more reliable, reproducible results.

```python
import numpy as np

rng = np.random.default_rng(seed=42)   # create a random generator, with a fixed seed
rng.random(5)                            # 5 random floats between 0 and 1
```

The **seed** is the key to reproducibility: using the same seed always produces the exact same "random" sequence — essential for debugging and for anyone else being able to reproduce your results exactly.

## WHY IT MATTERS

You'll use random sampling constantly in this course and beyond: generating realistic fake datasets to practice on before you have real data access, simulating "what if" scenarios (Monte Carlo-style thinking), and — once you reach machine learning — randomly splitting data into training and test sets. Setting a seed is a genuine best practice in professional data work: a reviewer (or future you) rerunning your analysis should get identical results, not a different random sample each time.

## SYNTAX REFERENCE

```python
import numpy as np

# Create a reproducible random number generator
rng = np.random.default_rng(seed=42)

# Random floats between 0 and 1
rng.random(5)                    # array of 5 floats, each in [0, 1)

# Random integers
rng.integers(1, 100, size=10)     # 10 random ints, 1 to 99 (upper bound exclusive by default)
rng.integers(1, 100, size=10, endpoint=True)   # includes 100 as a possible value

# Random floats within a custom range
rng.uniform(10, 50, size=5)        # 5 random floats between 10 and 50

# Sampling from a normal (bell curve) distribution
rng.normal(loc=100, scale=15, size=1000)    # mean=100, std=15, 1000 samples

# Random choice from an existing array (with or without replacement)
options = np.array(["gold", "silver", "bronze"])
rng.choice(options, size=10)                    # random picks, WITH replacement (repeats allowed)
rng.choice(options, size=3, replace=False)        # WITHOUT replacement (no repeats)

# Weighted random choice (some options more likely than others)
rng.choice(options, size=10, p=[0.5, 0.3, 0.2])    # 50% gold, 30% silver, 20% bronze

# Shuffling an array in place
arr = np.arange(10)
rng.shuffle(arr)               # arr is now shuffled, in place

# Random sample of indices (useful for train/test-style splits)
rng.permutation(10)             # a shuffled array of 0-9, doesn't modify anything in place
```

## WORKED EXAMPLE

Generating a realistic synthetic dataset of orders for practice — exactly the kind of dataset you might build to test a pipeline before real data is available:

```python
import numpy as np

rng = np.random.default_rng(seed=42)   # fixed seed -> reproducible results every run

n_orders = 10

# Random order totals from a normal distribution (mean $75, std $30), floored at $5
order_totals = rng.normal(loc=75, scale=30, size=n_orders)
order_totals = np.round(np.maximum(order_totals, 5), 2)   # no negative/tiny orders
print("Order totals:", order_totals)

# Random customer tiers, weighted (most customers are "silver")
tiers = rng.choice(
    ["bronze", "silver", "gold"],
    size=n_orders,
    p=[0.3, 0.5, 0.2],
)
print("Tiers:", tiers)

# Random item counts (integers from 1 to 10)
item_counts = rng.integers(1, 11, size=n_orders)
print("Item counts:", item_counts)

# Randomly select 3 of these 10 orders for a manual quality-check sample
sample_indices = rng.choice(n_orders, size=3, replace=False)
print("Orders selected for QA review:", sorted(sample_indices))

# Combine everything into a small synthetic dataset — a list of dicts,
# exactly the shape you'd build for testing a pipeline before real data
synthetic_orders = []
for i in range(n_orders):
    synthetic_orders.append({
        "order_id": i + 1,
        "total": order_totals[i],
        "tier": tiers[i],
        "item_count": int(item_counts[i]),
    })

for order in synthetic_orders[:3]:   # print just the first 3
    print(order)

# Because we used seed=42, running this exact script again will produce
# IDENTICAL results every time — critical for reproducible testing
```

## COMMON MISTAKES

**1. Not setting a seed, and being unable to reproduce a result later**
```python
rng = np.random.default_rng()   # no seed — different results every run
```
Fix: always set an explicit `seed` value during development and testing. Only omit it deliberately in true production randomness scenarios (like a live A/B test assignment) where you genuinely want unpredictability.

**2. Using the older, global-state `np.random.rand()`/`np.random.seed()` style instead of a Generator**
```python
np.random.seed(42)      # older style — sets GLOBAL state, can cause hard-to-debug
np.random.rand(5)         # interactions if other code also touches global random state
```
Fix: prefer `np.random.default_rng(seed=42)` and call methods on that specific generator object — it's self-contained and doesn't risk interference from unrelated code elsewhere in a larger program.

**3. Confusing `rng.integers(1, 100)` bounds**
```python
rng.integers(1, 100, size=5)   # values from 1 to 99 — 100 is EXCLUDED by default!
```
Fix: remember the upper bound is exclusive by default, matching Python's `range()` convention from Module 1 — use `endpoint=True` if you specifically want the upper bound included.

**4. Sampling with replacement when you meant without (or vice versa)**
```python
rng.choice(customer_ids, size=5)                  # WITH replacement — same ID could appear twice
rng.choice(customer_ids, size=5, replace=False)     # WITHOUT replacement — each ID appears at most once
```
Fix: think about whether repeats make sense for your scenario — picking 5 random *customers* for a survey should usually be `replace=False` (you don't want to survey the same person twice), while simulating 5 random dice rolls should be `replace=True` (the default).

**5. Forgetting `rng.shuffle()` modifies the array in place and returns `None`**
```python
shuffled = rng.shuffle(arr)   # shuffled is None! arr itself was modified instead
print(shuffled)                  # None
```
Fix: after calling `.shuffle()`, use the original array variable (`arr`), not the return value. If you want a shuffled *copy* without modifying the original, use `rng.permutation(arr)` instead, which returns a new shuffled array and leaves the original untouched.

**Data Engineer Tip:** Generating realistic synthetic data (as in the worked example) is a genuinely useful production skill — it's how you build and test a pipeline *before* you have access to real (often sensitive) production data, and how you create safe, non-identifying sample datasets for demos, documentation, or a portfolio project. Setting a seed is what makes a synthetic dataset "the same" every time you regenerate it — useful for a reproducible portfolio project others can run and verify themselves.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create a random number generator with `seed=1`, and generate 5 random integers between 1 and 10 (10 exclusive). Print the result.

<details>
<summary>Hint</summary>
`rng.integers(1, 10, size=5)`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

rng = np.random.default_rng(seed=1)
values = rng.integers(1, 10, size=5)
print(values)
```
</details>

---

**Exercise 2 (Easy):** Using the same generator, randomly choose 3 items (with replacement) from `["red", "blue", "green"]`, and print the result.

<details>
<summary>Hint</summary>
`rng.choice(options, size=3)`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

rng = np.random.default_rng(seed=1)
options = np.array(["red", "blue", "green"])
choices = rng.choice(options, size=3)
print(choices)
```
</details>

---

**Exercise 3 (Medium):** Generate 1,000 samples from a normal distribution with mean 50 and standard deviation 10 (`seed=7`). Print the computed mean and standard deviation of your generated sample, and confirm they're reasonably close to 50 and 10 respectively.

<details>
<summary>Hint</summary>
`rng.normal(loc=50, scale=10, size=1000)`, then `.mean()` and `.std()` on the result.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

rng = np.random.default_rng(seed=7)
sample = rng.normal(loc=50, scale=10, size=1000)

print(f"Sample mean: {sample.mean():.2f}")   # close to 50
print(f"Sample std: {sample.std():.2f}")      # close to 10
```
</details>

---

**Exercise 4 (Medium):** Using `seed=42`, randomly select 5 customer IDs *without replacement* from a pool of `customer_ids = np.arange(1, 101)` (IDs 1 through 100), simulating picking 5 unique customers for a survey. Print the selected IDs, and confirm there are no duplicates using `len(set(...))`.

<details>
<summary>Hint</summary>
`rng.choice(customer_ids, size=5, replace=False)`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

rng = np.random.default_rng(seed=42)
customer_ids = np.arange(1, 101)

selected = rng.choice(customer_ids, size=5, replace=False)
print("Selected:", selected)
print("Unique count:", len(set(selected)))   # 5 — confirms no duplicates
```
</details>

---

**Exercise 5 (Hard):** Using `seed=99`, generate a synthetic dataset of 20 orders with: `order_total` (normal distribution, mean=60, std=25, floored at a minimum of $5 using `np.maximum`), `category` (randomly chosen from `["Electronics", "Home", "Outdoor"]` with weights `[0.5, 0.3, 0.2]`), and `is_returned` (randomly `True` 10% of the time — hint: use `rng.random(20) < 0.10` to generate this boolean array directly). Then, without any explicit loop: (1) compute the total revenue from non-returned orders only (boolean mask), (2) compute the percentage of orders that were returned, and (3) find the average order total specifically within the `"Electronics"` category (another boolean mask). Print all three results.

<details>
<summary>Hint</summary>
Build all three arrays first with the generator. For "non-returned total," mask `order_totals` with `~is_returned`. For "electronics average," mask `order_totals` with `category == "Electronics"`, then take `.mean()` of the result.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np

rng = np.random.default_rng(seed=99)
n = 20

order_totals = rng.normal(loc=60, scale=25, size=n)
order_totals = np.round(np.maximum(order_totals, 5), 2)

category = rng.choice(["Electronics", "Home", "Outdoor"], size=n, p=[0.5, 0.3, 0.2])
is_returned = rng.random(n) < 0.10

# 1. Revenue from non-returned orders
non_returned_revenue = order_totals[~is_returned].sum()
print(f"Non-returned revenue: ${non_returned_revenue:.2f}")

# 2. Percentage returned
pct_returned = (is_returned.sum() / n) * 100
print(f"Percent returned: {pct_returned:.1f}%")

# 3. Average order total for Electronics
electronics_avg = order_totals[category == "Electronics"].mean()
print(f"Electronics average order: ${electronics_avg:.2f}")
```
Results will vary slightly depending on NumPy version, but will be identical every time this exact code runs with `seed=99` on the same NumPy version — that reproducibility is the entire point of seeding.
</details>

---

## Module 3 Complete

You've now covered the core of NumPy: arrays vs. lists, creation and shape, indexing/slicing/masking, broadcasting, aggregations, missing data, and random sampling. Every one of these concepts reappears — with a friendlier interface, but the exact same underlying logic — throughout Module 4 and 5, where you'll work with pandas DataFrames instead of raw arrays.

**Previous lesson:** [3.6 — Handling Missing Data (NaN)](lesson-3.6-nan-handling.md)
**Next module:** [Module 4 — Pandas I: Data Wrangling](../module-4-pandas-1-data-wrangling/README.md)
