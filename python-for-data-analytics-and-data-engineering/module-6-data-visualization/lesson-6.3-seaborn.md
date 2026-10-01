# Lesson 6.3 — seaborn for Statistical Visualization

## CONCEPT

**seaborn** is a visualization library built on top of matplotlib, designed specifically for *statistical* charts — distributions, categorical comparisons, and correlations — with far less code than the matplotlib equivalent, and with attractive default styling. Crucially, seaborn is built to work directly with pandas DataFrames: instead of passing raw arrays, you pass the DataFrame and the column *names*.

```python
import seaborn as sns

sns.histplot(data=orders, x="order_total")            # distribution
sns.boxplot(data=orders, x="category", y="order_total")  # distribution BY category
sns.heatmap(correlation_matrix, annot=True)               # correlations
```

seaborn charts still return a matplotlib `Axes` object underneath, so everything from Lessons 6.1–6.2 (titles, labels, `ax` methods) still applies on top of a seaborn chart.

## WHY IT MATTERS

Certain statistical questions — "how does order value differ across customer tiers?", "which numeric columns in this dataset are correlated with each other?", "what does the distribution of ratings look like, broken down by category?" — are painful to build in raw matplotlib but nearly one-liners in seaborn. Because seaborn reads column names directly from a DataFrame, it also fits naturally into the pandas-centric workflow you built in Modules 4–5, letting you go from a `groupby()` result straight to a chart with minimal glue code.

## SYNTAX REFERENCE

```python
import seaborn as sns
import matplotlib.pyplot as plt

# Distribution of a single numeric column
sns.histplot(data=df, x="order_total", bins=25)
sns.kdeplot(data=df, x="order_total")             # smooth density curve instead of bars

# Distribution of a numeric column, broken down BY a category
sns.boxplot(data=df, x="category", y="order_total")     # box-and-whisker per category
sns.violinplot(data=df, x="category", y="order_total")    # similar, shows full distribution shape

# Bar chart of an aggregate (seaborn computes the aggregation for you)
sns.barplot(data=df, x="category", y="order_total")    # defaults to showing the MEAN per category,
                                                           # with a confidence interval indicator

# Scatter plot with extra statistical features
sns.scatterplot(data=df, x="price", y="rating", hue="category")  # hue = color by a category
sns.regplot(data=df, x="price", y="rating")                        # scatter PLUS a fitted trend line

# Heatmap — typically used for a correlation matrix
correlation_matrix = df[["price", "rating", "quantity"]].corr()
sns.heatmap(correlation_matrix, annot=True, cmap="coolwarm")
# annot=True prints the actual numeric value inside each cell
# cmap sets the color scheme

# Pairwise relationships across every numeric column at once
sns.pairplot(df[["price", "rating", "quantity"]])

# Styling shortcuts
sns.set_style("whitegrid")     # apply a clean default style to ALL subsequent charts
sns.set_palette("Set2")          # apply a default color palette

# seaborn functions return an Axes — combine with matplotlib methods as usual
ax = sns.boxplot(data=df, x="category", y="order_total")
ax.set_title("Order Total by Category")
plt.show()
```

## WORKED EXAMPLE

Exploring order data statistically — distributions by category, a correlation heatmap, and a trend with a fitted line:

```python
import seaborn as sns
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd

np.random.seed(3)
n = 300
orders = pd.DataFrame({
    "order_total": np.round(np.random.normal(75, 30, n).clip(5), 2),
    "rating": np.clip(np.random.normal(3.8, 0.9, n), 1, 5).round(1),
    "category": np.random.choice(["Electronics", "Home", "Outdoor"], n, p=[0.5, 0.3, 0.2]),
    "quantity": np.random.randint(1, 6, n),
})

sns.set_style("whitegrid")   # apply once, affects every chart below

fig, axes = plt.subplots(nrows=2, ncols=2, figsize=(12, 9))

# 1. Distribution of order_total, split by category (box plot)
sns.boxplot(data=orders, x="category", y="order_total", ax=axes[0, 0])
axes[0, 0].set_title("Order Total by Category")

# 2. Distribution shape with a smooth density curve
sns.kdeplot(data=orders, x="order_total", ax=axes[0, 1], fill=True)
axes[0, 1].set_title("Order Total Density")

# 3. Correlation heatmap of the numeric columns
corr = orders[["order_total", "rating", "quantity"]].corr()
sns.heatmap(corr, annot=True, cmap="coolwarm", ax=axes[1, 0])
axes[1, 0].set_title("Correlation Matrix")

# 4. Scatter with a fitted trend line, colored by category
sns.scatterplot(data=orders, x="order_total", y="rating", hue="category", alpha=0.6, ax=axes[1, 1])
axes[1, 1].set_title("Order Total vs. Rating, by Category")

fig.tight_layout()
plt.show()
# Panel 1: three box plots (one per category) showing the spread of order
#   totals within each, useful for spotting whether one category tends
#   toward higher or more variable order values
# Panel 2: a single smooth bell-shaped curve for the overall order_total
#   distribution, filled underneath
# Panel 3: a 3x3 grid of correlation values between order_total, rating,
#   and quantity — likely close to 0 for all pairs here, since the
#   synthetic data was generated independently
# Panel 4: a scatter of order_total against rating, with points colored
#   by category, plus a legend
```

## COMMON MISTAKES

**1. Passing raw arrays instead of a DataFrame + column names**
```python
sns.histplot(x=orders["order_total"].values)   # works, but loses seaborn's real strength
sns.histplot(data=orders, x="order_total")        # idiomatic — pass the DataFrame and column name
```
Fix: prefer the `data=`/column-name style throughout — it's what makes features like `hue=` (automatic coloring by another column) and automatic axis labeling work seamlessly.

**2. Forgetting `hue=` needs a categorical (or low-cardinality) column to be useful**
```python
sns.scatterplot(data=orders, x="price", y="rating", hue="order_id")   # a unique ID per row —
                                                                          # produces a useless,
                                                                          # unreadable legend with
                                                                          # hundreds of entries
```
Fix: `hue=` works best with a genuinely categorical column (a handful of distinct values) — using a high-cardinality column like an ID defeats the purpose and clutters the chart.

**3. Not realizing `sns.barplot()` aggregates automatically, and being confused by the result**
```python
sns.barplot(data=orders, x="category", y="order_total")   # shows the MEAN per category by default,
                                                              # NOT a count or a sum!
```
Fix: if you want a *sum*, aggregate first with pandas (`orders.groupby("category")["order_total"].sum()`) and plot that result directly with a plain matplotlib `ax.bar()`, or use `sns.barplot(..., estimator="sum")` to change seaborn's aggregation explicitly.

**4. Passing a non-square or improperly-shaped table to `sns.heatmap()`**
```python
sns.heatmap(orders)   # tries to heatmap the WHOLE DataFrame, including non-numeric columns — error or nonsense
```
Fix: `heatmap()` expects a genuinely 2D numeric matrix — typically the output of `.corr()` or a `pivot_table()` (Lesson 5.3), not a raw DataFrame of mixed-type columns.

**5. Forgetting to pass `ax=` when placing a seaborn chart into a specific subplot**
```python
fig, axes = plt.subplots(nrows=1, ncols=2)
sns.boxplot(data=orders, x="category", y="order_total")   # plots to a NEW figure, ignoring axes[0]!
```
Fix: always pass `ax=axes[0]` (or whichever specific subplot) explicitly when building a multi-panel figure with seaborn — without it, seaborn creates its own default figure instead of using your subplot grid.

**Data Engineer Tip:** `sns.heatmap()` on a `.corr()` matrix is one of the fastest ways to do an initial "what's related to what" scan across every numeric column in a new dataset — genuinely useful during the exploratory phase before you decide which relationships are worth investigating further with a proper statistical test. It's a quick sanity check, not a substitute for rigorous analysis (correlation isn't causation, and a heatmap alone won't tell you whether a relationship is meaningful or just noise) — but as a first-look tool, it's hard to beat for speed.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `data = pd.DataFrame({"value": np.random.normal(50, 10, 200)})` (with `np.random.seed(5)` set first), create a `sns.histplot()` of the `"value"` column.

<details>
<summary>Hint</summary>
`sns.histplot(data=data, x="value")`
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

np.random.seed(5)
data = pd.DataFrame({"value": np.random.normal(50, 10, 200)})

sns.histplot(data=data, x="value")
plt.title("Value Distribution")
plt.show()
```
</details>

---

**Exercise 2 (Easy):** Given
```python
df = pd.DataFrame({
    "category": ["A", "A", "B", "B", "C", "C"],
    "total": [45, 55, 30, 35, 80, 70],
})
```
create a box plot of `total` grouped by `category`.

<details>
<summary>Hint</summary>
`sns.boxplot(data=df, x="category", y="total")`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

df = pd.DataFrame({
    "category": ["A", "A", "B", "B", "C", "C"],
    "total": [45, 55, 30, 35, 80, 70],
})

sns.boxplot(data=df, x="category", y="total")
plt.title("Total by Category")
plt.show()
```
</details>

---

**Exercise 3 (Medium):** Given a DataFrame with three numeric columns — `df = pd.DataFrame({"a": np.random.normal(0, 1, 100), "b": np.random.normal(0, 1, 100), "c": np.random.normal(0, 1, 100)})` (with `np.random.seed(6)`), compute the correlation matrix and visualize it with `sns.heatmap()`, including `annot=True`.

<details>
<summary>Hint</summary>
`df.corr()`, then `sns.heatmap(corr, annot=True)`.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

np.random.seed(6)
df = pd.DataFrame({
    "a": np.random.normal(0, 1, 100),
    "b": np.random.normal(0, 1, 100),
    "c": np.random.normal(0, 1, 100),
})

corr = df.corr()
sns.heatmap(corr, annot=True, cmap="coolwarm")
plt.title("Correlation Matrix")
plt.show()
# Expect values close to 0 off the diagonal, since these were generated
# independently — and exactly 1.0 on the diagonal (each column with itself)
```
</details>

---

**Exercise 4 (Medium):** Given
```python
df = pd.DataFrame({
    "price": [20, 45, 60, 15, 80, 35, 55, 25],
    "rating": [4.5, 3.2, 4.8, 2.1, 4.9, 3.5, 4.1, 3.8],
    "category": ["A", "B", "A", "B", "A", "B", "A", "B"],
})
```
create a `sns.scatterplot()` of `price` vs. `rating`, colored by `category` using `hue=`.

<details>
<summary>Hint</summary>
`sns.scatterplot(data=df, x="price", y="rating", hue="category")`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

df = pd.DataFrame({
    "price": [20, 45, 60, 15, 80, 35, 55, 25],
    "rating": [4.5, 3.2, 4.8, 2.1, 4.9, 3.5, 4.1, 3.8],
    "category": ["A", "B", "A", "B", "A", "B", "A", "B"],
})

sns.scatterplot(data=df, x="price", y="rating", hue="category")
plt.title("Price vs. Rating by Category")
plt.show()
```
</details>

---

**Exercise 5 (Hard):** Given synthetic order data:
```python
np.random.seed(10)
n = 250
orders = pd.DataFrame({
    "order_total": np.round(np.random.normal(80, 25, n).clip(5), 2),
    "rating": np.clip(np.random.normal(4.0, 0.8, n), 1, 5).round(1),
    "quantity": np.random.randint(1, 8, n),
    "customer_tier": np.random.choice(["bronze", "silver", "gold"], n, p=[0.4, 0.4, 0.2]),
})
```
Build a 2x2 figure using matplotlib subplots + seaborn (remembering to pass `ax=` to every seaborn call): (1) top-left: a box plot of `order_total` by `customer_tier`; (2) top-right: a `sns.barplot()` of the MEAN `order_total` per `customer_tier`; (3) bottom-left: a correlation heatmap of `order_total`, `rating`, and `quantity`, with `annot=True`; (4) bottom-right: a `sns.scatterplot()` of `order_total` vs. `rating`, colored by `customer_tier`. Give each panel a title and call `tight_layout()`.

<details>
<summary>Hint</summary>
Build the 2x2 grid first with `plt.subplots(nrows=2, ncols=2)`, then call each seaborn function with the matching `ax=axes[row, col]`.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np
import pandas as pd
import seaborn as sns
import matplotlib.pyplot as plt

np.random.seed(10)
n = 250
orders = pd.DataFrame({
    "order_total": np.round(np.random.normal(80, 25, n).clip(5), 2),
    "rating": np.clip(np.random.normal(4.0, 0.8, n), 1, 5).round(1),
    "quantity": np.random.randint(1, 8, n),
    "customer_tier": np.random.choice(["bronze", "silver", "gold"], n, p=[0.4, 0.4, 0.2]),
})

fig, axes = plt.subplots(nrows=2, ncols=2, figsize=(12, 9))

sns.boxplot(data=orders, x="customer_tier", y="order_total", ax=axes[0, 0])
axes[0, 0].set_title("Order Total by Tier")

sns.barplot(data=orders, x="customer_tier", y="order_total", ax=axes[0, 1])
axes[0, 1].set_title("Mean Order Total by Tier")

corr = orders[["order_total", "rating", "quantity"]].corr()
sns.heatmap(corr, annot=True, cmap="coolwarm", ax=axes[1, 0])
axes[1, 0].set_title("Correlation Matrix")

sns.scatterplot(data=orders, x="order_total", y="rating", hue="customer_tier", alpha=0.6, ax=axes[1, 1])
axes[1, 1].set_title("Order Total vs. Rating by Tier")

fig.tight_layout()
plt.show()
```
</details>

---

**Previous lesson:** [6.2 — Line, Bar, Histogram & Scatter Plots](lesson-6.2-core-chart-types.md)
**Next lesson:** [6.4 — Customization: Labels, Legends, Styling & Saving](lesson-6.4-customization-and-saving.md)
