# Lesson 6.2 — Line, Bar, Histogram & Scatter Plots

## CONCEPT

Four chart types cover the large majority of everyday data analytics needs, and each answers a fundamentally different kind of question:

- **Line chart** — how a value changes over a continuous sequence (usually time)
- **Bar chart** — comparing a value across discrete categories
- **Histogram** — the *distribution* (shape/spread) of a single numeric variable
- **Scatter plot** — the relationship between two numeric variables

```python
ax.plot(months, revenue)              # line — trend over time
ax.bar(categories, totals)             # bar — comparison across categories
ax.hist(order_totals, bins=20)          # histogram — distribution of one variable
ax.scatter(price, rating)                # scatter — relationship between two variables
```

## WHY IT MATTERS

Choosing the wrong chart type for a question is one of the most common — and most avoidable — visualization mistakes. A bar chart can't show a trend over time as clearly as a line chart; a line chart connecting unrelated categories (implying a false continuity) is actively misleading; a scatter plot answers "is there a relationship?" in a way no other chart type here can. Knowing these four cold, and matching each to the right question, is most of what "good chart choice" means in practice (formalized further in Lesson 6.6).

## SYNTAX REFERENCE

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots()

# Line chart — trend over a sequence
ax.plot(x_values, y_values)
ax.plot(x, y, marker="o", linestyle="--", color="green")   # customized

# Multiple lines on the same chart
ax.plot(months, revenue_2023, label="2023")
ax.plot(months, revenue_2024, label="2024")
ax.legend()    # needed to show the labels

# Bar chart — comparison across categories
ax.bar(categories, values)
ax.bar(categories, values, color="coral")
ax.barh(categories, values)          # horizontal bars — better for long category names

# Grouped bar chart (comparing two series side by side per category)
import numpy as np
x = np.arange(len(categories))
width = 0.35
ax.bar(x - width/2, values_a, width, label="A")
ax.bar(x + width/2, values_b, width, label="B")
ax.set_xticks(x)
ax.set_xticklabels(categories)
ax.legend()

# Histogram — distribution of one numeric variable
ax.hist(order_totals, bins=20)                # bins controls how many "buckets"
ax.hist(order_totals, bins=20, edgecolor="white")   # cleaner-looking bars

# Scatter plot — relationship between two numeric variables
ax.scatter(price, rating)
ax.scatter(price, rating, alpha=0.5)          # alpha = transparency, helps with overlapping points
ax.scatter(price, rating, c=category_codes)     # color points by a third (categorical) variable
```

## WORKED EXAMPLE

Building all four chart types against a small synthetic `ecommerce` dataset:

```python
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd

# Synthetic order data
np.random.seed(42)
n = 200
orders = pd.DataFrame({
    "order_total": np.round(np.random.normal(75, 30, n).clip(5), 2),
    "rating": np.random.randint(1, 6, n),
    "category": np.random.choice(["Electronics", "Home", "Outdoor"], n, p=[0.5, 0.3, 0.2]),
})
monthly_revenue = pd.Series(
    [12000, 15000, 11000, 18000, 21000, 19500],
    index=["Jan", "Feb", "Mar", "Apr", "May", "Jun"],
)
category_totals = orders.groupby("category")["order_total"].sum()

fig, axes = plt.subplots(nrows=2, ncols=2, figsize=(11, 9))

# 1. Line chart — revenue trend over months
axes[0, 0].plot(monthly_revenue.index, monthly_revenue.values, marker="o", color="steelblue")
axes[0, 0].set_title("Monthly Revenue (Line)")
axes[0, 0].set_ylabel("Revenue ($)")

# 2. Bar chart — total revenue by category
axes[0, 1].bar(category_totals.index, category_totals.values, color="coral")
axes[0, 1].set_title("Revenue by Category (Bar)")
axes[0, 1].set_ylabel("Total Revenue ($)")

# 3. Histogram — distribution of order totals
axes[1, 0].hist(orders["order_total"], bins=25, edgecolor="white", color="seagreen")
axes[1, 0].set_title("Order Total Distribution (Histogram)")
axes[1, 0].set_xlabel("Order Total ($)")
axes[1, 0].set_ylabel("Number of Orders")

# 4. Scatter plot — order total vs. rating (is there a relationship?)
axes[1, 1].scatter(orders["order_total"], orders["rating"], alpha=0.4, color="purple")
axes[1, 1].set_title("Order Total vs. Rating (Scatter)")
axes[1, 1].set_xlabel("Order Total ($)")
axes[1, 1].set_ylabel("Rating (1-5)")

fig.tight_layout()
plt.show()
# Four distinct panels: a smooth trend line, a 3-bar comparison, a bell-shaped
# distribution of order amounts centered around $75, and a scattered cloud
# of points showing no obvious strong relationship between price and rating
# — itself a useful (negative) finding
```

## COMMON MISTAKES

**1. Using a line chart for unordered, unrelated categories**
```python
ax.plot(["Electronics", "Home", "Outdoor"], [500, 300, 200])   # misleading!
```
A line implies continuity and order between points that categories like this don't actually have — it visually suggests a "trend" from Electronics to Outdoor that doesn't mean anything. Fix: use a bar chart for categorical comparisons; reserve line charts for genuinely sequential data (usually time).

**2. Choosing too few or too many histogram bins**
```python
ax.hist(order_totals, bins=3)     # too coarse — hides the actual shape of the distribution
ax.hist(order_totals, bins=200)     # too fine — mostly noise, hard to see the overall pattern
```
Fix: there's no single correct number, but 15–30 bins is a reasonable starting point for most datasets — adjust and look at the result; the right number often becomes visually obvious once you compare a couple of options.

**3. Overplotting a scatter plot with too many points, with no transparency**
```python
ax.scatter(x, y)   # with 10,000+ points, this becomes an unreadable solid blob
```
Fix: use `alpha=0.3` (or lower) to make overlapping points visible as denser regions rather than a solid mass — this single parameter dramatically improves readability on large scatter plots.

**4. Forgetting `.legend()` after adding `label=` to multiple series**
```python
ax.plot(months, revenue_2023, label="2023")
ax.plot(months, revenue_2024, label="2024")
# forgot ax.legend() — the labels exist but never actually display!
```
Fix: `label=` alone does nothing visible — you must call `ax.legend()` afterward to actually render the legend box.

**5. Not sorting categories in a bar chart when order matters**
```python
category_totals = orders.groupby("category")["total"].sum()   # alphabetical by default
ax.bar(category_totals.index, category_totals.values)            # bars in arbitrary/alphabetical order
```
Fix: if you want bars sorted by value (largest first, typically clearer for a ranking-style comparison), sort explicitly first: `category_totals.sort_values(ascending=False)`.

**Data Engineer Tip:** A histogram of a key numeric column (like `order_total` or `price`) is one of the fastest, most useful ad hoc data quality checks available — a distribution with an unexpected second peak, an odd spike at exactly zero, or values trailing off into implausible territory (negative prices, impossibly large totals) is often visible at a glance in a histogram well before it would show up in a written data quality report. Many experienced analysts build a habit of histogramming every new numeric column before doing anything else with it.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `x = [1, 2, 3, 4, 5]` and `y = [2, 4, 3, 5, 6]`, create a line chart with markers, titled `"Trend"`.

<details>
<summary>Hint</summary>
`ax.plot(x, y, marker="o")`
</details>

<details>
<summary>Solution</summary>

```python
import matplotlib.pyplot as plt

x = [1, 2, 3, 4, 5]
y = [2, 4, 3, 5, 6]

fig, ax = plt.subplots()
ax.plot(x, y, marker="o")
ax.set_title("Trend")
plt.show()
```
</details>

---

**Exercise 2 (Easy):** Given `categories = ["A", "B", "C", "D"]` and `values = [23, 45, 12, 38]`, create a bar chart, and separately, a horizontal bar chart (`barh`) of the same data. Compare the two.

<details>
<summary>Hint</summary>
`ax.bar()` vs. `ax.barh()` — note the arguments to `barh` are in the same order.
</details>

<details>
<summary>Solution</summary>

```python
import matplotlib.pyplot as plt

categories = ["A", "B", "C", "D"]
values = [23, 45, 12, 38]

fig, axes = plt.subplots(nrows=1, ncols=2, figsize=(10, 4))
axes[0].bar(categories, values)
axes[0].set_title("Vertical Bars")
axes[1].barh(categories, values)
axes[1].set_title("Horizontal Bars")
plt.show()
```
</details>

---

**Exercise 3 (Medium):** Given `np.random.seed(1)` followed by `values = np.random.normal(50, 15, 500)`, create a histogram with 25 bins, a title, and an x-axis label. Describe in a comment what shape you'd expect to see.

<details>
<summary>Hint</summary>
`ax.hist(values, bins=25)` — a normal distribution should produce a roughly symmetric, bell-shaped histogram.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np
import matplotlib.pyplot as plt

np.random.seed(1)
values = np.random.normal(50, 15, 500)

fig, ax = plt.subplots()
ax.hist(values, bins=25, edgecolor="white")
ax.set_title("Distribution of Simulated Values")
ax.set_xlabel("Value")
plt.show()
# Expected: a roughly bell-shaped, symmetric histogram centered around 50,
# since the data was drawn from a normal distribution with mean 50
```
</details>

---

**Exercise 4 (Medium):** Given `np.random.seed(2)`, `x = np.random.uniform(10, 100, 150)` (simulating prices) and `y = x * 0.3 + np.random.normal(0, 5, 150)` (simulating a related quantity, e.g. commission), create a scatter plot with `alpha=0.5`, and add axis labels. In a comment, describe the relationship you'd expect to see.

<details>
<summary>Hint</summary>
`ax.scatter(x, y, alpha=0.5)` — since `y` was built directly from `x` plus some noise, expect a clear upward/positive relationship.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np
import matplotlib.pyplot as plt

np.random.seed(2)
x = np.random.uniform(10, 100, 150)
y = x * 0.3 + np.random.normal(0, 5, 150)

fig, ax = plt.subplots()
ax.scatter(x, y, alpha=0.5)
ax.set_xlabel("Price")
ax.set_ylabel("Commission")
ax.set_title("Price vs. Commission")
plt.show()
# Expected: a clear upward-trending cloud of points — as price increases,
# commission tends to increase too, since y was constructed as a function of x
```
</details>

---

**Exercise 5 (Hard):** Given
```python
months = ["Jan", "Feb", "Mar", "Apr", "May", "Jun"]
revenue_2023 = [10000, 11500, 9800, 13000, 14200, 13800]
revenue_2024 = [12000, 15000, 11000, 18000, 21000, 19500]
```
build a single line chart comparing both years on the same axes (two lines, each with a `label`, different colors, with markers), a legend, a title, and axis labels. Then, in a **separate** figure, build a grouped bar chart comparing the same two years side by side per month (hint: this needs the `np.arange()` + `width` offset pattern from the Syntax Reference). Add a legend and title to the bar chart too.

<details>
<summary>Hint</summary>
For the line chart: two `ax.plot()` calls with `label=`, then `ax.legend()`. For the grouped bar chart: `x = np.arange(len(months))`, then two `ax.bar()` calls offset by `-width/2` and `+width/2`, followed by `ax.set_xticks(x)` and `ax.set_xticklabels(months)`.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np
import matplotlib.pyplot as plt

months = ["Jan", "Feb", "Mar", "Apr", "May", "Jun"]
revenue_2023 = [10000, 11500, 9800, 13000, 14200, 13800]
revenue_2024 = [12000, 15000, 11000, 18000, 21000, 19500]

# --- Line chart ---
fig1, ax1 = plt.subplots(figsize=(8, 5))
ax1.plot(months, revenue_2023, marker="o", label="2023", color="steelblue")
ax1.plot(months, revenue_2024, marker="o", label="2024", color="darkorange")
ax1.set_title("Revenue: 2023 vs. 2024")
ax1.set_xlabel("Month")
ax1.set_ylabel("Revenue ($)")
ax1.legend()
plt.show()

# --- Grouped bar chart ---
x = np.arange(len(months))
width = 0.35

fig2, ax2 = plt.subplots(figsize=(8, 5))
ax2.bar(x - width/2, revenue_2023, width, label="2023", color="steelblue")
ax2.bar(x + width/2, revenue_2024, width, label="2024", color="darkorange")
ax2.set_xticks(x)
ax2.set_xticklabels(months)
ax2.set_title("Revenue: 2023 vs. 2024 (Grouped Bars)")
ax2.set_xlabel("Month")
ax2.set_ylabel("Revenue ($)")
ax2.legend()
plt.show()
# The line chart shows both years as two overlapping trend lines; the grouped
# bar chart shows the same comparison as pairs of side-by-side bars per month —
# generally clearer for month-by-month comparison, while the line chart is
# clearer for seeing the overall shape of the trend across the year
```
</details>

---

**Previous lesson:** [6.1 — matplotlib Fundamentals](lesson-6.1-matplotlib-fundamentals.md)
**Next lesson:** [6.3 — seaborn for Statistical Visualization](lesson-6.3-seaborn.md)
