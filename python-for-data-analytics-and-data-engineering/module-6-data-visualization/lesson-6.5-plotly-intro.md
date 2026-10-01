# Lesson 6.5 — Intro to Interactive Visualization with Plotly

## CONCEPT

Everything so far in this module produces **static** images — fixed pictures. **Plotly** builds **interactive** charts instead: the viewer can hover over a point to see its exact value, zoom into a region, toggle a series on/off by clicking the legend, and pan around — all without writing any extra code, since Plotly builds this into every chart by default. Plotly's `plotly.express` module (usually imported as `px`) offers a concise, pandas-friendly API, similar in spirit to seaborn's `data=`/column-name style.

```python
import plotly.express as px

fig = px.line(df, x="month", y="revenue", title="Monthly Revenue")
fig.show()      # opens an interactive chart in a browser tab or notebook cell
```

## WHY IT MATTERS

A static PNG is fine for a printed report or an email, but for a dashboard, a shared notebook, or any web-based deliverable, interactivity meaningfully improves how someone can actually explore your data — letting a stakeholder hover to check an exact number, or zoom into the one time period they care about, without you needing to pre-build a version of the chart for every possible question they might have. Plotly is also the visualization library behind several popular dashboarding tools in the Python ecosystem, so this lesson is a genuine on-ramp to building shareable, interactive analytics deliverables beyond a single static chart.

## SYNTAX REFERENCE

```python
import plotly.express as px

# Core chart types — syntax closely mirrors seaborn's data=/column-name style
px.line(df, x="month", y="revenue")
px.bar(df, x="category", y="total")
px.histogram(df, x="order_total", nbins=25)
px.scatter(df, x="price", y="rating", color="category")     # color = seaborn's hue=
px.box(df, x="category", y="order_total")
px.pie(df, names="category", values="total")

# Multiple lines at once by specifying a "color" grouping column
px.line(df, x="month", y="revenue", color="year")

# Adding hover detail — extra columns shown on hover, beyond x/y
px.scatter(df, x="price", y="rating", color="category", hover_data=["product_name", "quantity"])

# Titles and labels
fig = px.line(df, x="month", y="revenue", title="Monthly Revenue")
fig.update_layout(xaxis_title="Month", yaxis_title="Revenue ($)")

# Displaying the chart
fig.show()                       # opens in a browser tab (script) or renders inline (notebook)

# Saving
fig.write_html("chart.html")       # a fully interactive standalone HTML file — shareable, no server needed
fig.write_image("chart.png")        # a static image (requires the `kaleido` package: pip install kaleido)

# A quick way to build a chart directly from a Series/simple data without a full DataFrame
import pandas as pd
s = pd.Series([10, 20, 15], index=["A", "B", "C"])
px.bar(s, title="Quick Bar Chart")
```

## WORKED EXAMPLE

Building an interactive dashboard-style chart set from synthetic order data:

```python
import plotly.express as px
import numpy as np
import pandas as pd

np.random.seed(11)
n = 200
orders = pd.DataFrame({
    "order_total": np.round(np.random.normal(75, 30, n).clip(5), 2),
    "rating": np.clip(np.random.normal(4.0, 0.8, n), 1, 5).round(1),
    "category": np.random.choice(["Electronics", "Home", "Outdoor"], n, p=[0.5, 0.3, 0.2]),
    "month": np.random.choice(["Jan", "Feb", "Mar", "Apr"], n),
})

# 1. An interactive scatter plot — hover over any point to see its exact values
fig1 = px.scatter(
    orders,
    x="order_total",
    y="rating",
    color="category",
    hover_data=["month"],
    title="Order Total vs. Rating, by Category",
)
fig1.update_layout(xaxis_title="Order Total ($)", yaxis_title="Rating")
fig1.show()
# An interactive chart opens: hovering over any point reveals its exact
# order_total, rating, category, and month; clicking a category name in the
# legend toggles that category's points on/off; the viewer can drag to zoom
# into any region and double-click to reset the view

# 2. An interactive bar chart of average order value per category
category_avg = orders.groupby("category", as_index=False)["order_total"].mean()
fig2 = px.bar(
    category_avg,
    x="category",
    y="order_total",
    title="Average Order Value by Category",
    text_auto=".2s",    # displays the value directly above each bar
)
fig2.show()

# 3. Save an interactive chart as a standalone, shareable HTML file
fig1.write_html("order_scatter.html")
print("Saved interactive chart to order_scatter.html — open it directly in any browser")
```

## COMMON MISTAKES

**1. Expecting `fig.show()` to work identically in every environment**
In a plain terminal/script context, `fig.show()` typically opens a new browser tab; in a Jupyter-style notebook, it usually renders inline in the cell output. If nothing appears, this is often an environment/renderer configuration issue rather than a code bug. Fix: `fig.write_html("chart.html")` is a reliable fallback that works everywhere — open the resulting file directly in any browser.

**2. Forgetting `write_image()` requires an extra package**
```python
fig.write_image("chart.png")   # ModuleNotFoundError if kaleido isn't installed
```
Fix: `pip install kaleido` specifically enables exporting Plotly charts as static images — it's a separate dependency from Plotly itself, easy to miss.

**3. Passing already-aggregated data into a chart type expecting raw rows (or vice versa)**
```python
px.bar(orders, x="category", y="order_total")   # plots ONE bar segment PER ROW, stacked — not what you want!
```
Fix: for a "one bar per category showing a summary value," aggregate first with pandas (`.groupby().mean()`, as in the worked example) and plot the aggregated result — `px.bar()` doesn't automatically summarize the way `sns.barplot()` does.

**4. Building a huge, cluttered chart by including too many hover fields or too many categories in `color=`**
A `color=` column with 30 distinct values produces a legend and color scheme that's no longer useful — Fix: reserve `color=` (like seaborn's `hue=`) for genuinely low-cardinality categorical columns; for high-cardinality grouping, consider faceting (`facet_col=`) into small multiples instead.

**5. Assuming an interactive HTML file is small and lightweight**
A single `write_html()` output embeds the full Plotly JavaScript library by default, which can make even a simple chart's HTML file several megabytes. Fix: if file size matters (e.g., emailing many charts), `fig.write_html("chart.html", include_plotlyjs="cdn")` loads the library from an external source instead of embedding it, dramatically shrinking the file — at the cost of requiring an internet connection to view it.

**Data Engineer Tip:** `fig.write_html()` producing a single, self-contained, shareable file with no server or special software needed to view is genuinely useful for portfolio work — you can commit an interactive chart directly into a GitHub repo (GitHub doesn't render the interactivity in its file preview, but anyone can download and open the file locally) or attach it to a report, letting a reviewer explore your analysis interactively rather than just looking at a fixed picture.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given `df = pd.DataFrame({"month": ["Jan", "Feb", "Mar"], "revenue": [1000, 1500, 1200]})`, create an interactive line chart with `px.line()`, with a title, and call `.show()`.

<details>
<summary>Hint</summary>
`px.line(df, x="month", y="revenue", title="...")`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import plotly.express as px

df = pd.DataFrame({"month": ["Jan", "Feb", "Mar"], "revenue": [1000, 1500, 1200]})

fig = px.line(df, x="month", y="revenue", title="Monthly Revenue")
fig.show()
```
</details>

---

**Exercise 2 (Easy):** Given
```python
df = pd.DataFrame({
    "category": ["A", "A", "B", "B"],
    "price": [10, 20, 15, 25],
    "rating": [3.5, 4.0, 3.8, 4.5],
})
```
create an interactive scatter plot of `price` vs. `rating`, colored by `category`.

<details>
<summary>Hint</summary>
`px.scatter(df, x="price", y="rating", color="category")`
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import plotly.express as px

df = pd.DataFrame({
    "category": ["A", "A", "B", "B"],
    "price": [10, 20, 15, 25],
    "rating": [3.5, 4.0, 3.8, 4.5],
})

fig = px.scatter(df, x="price", y="rating", color="category")
fig.show()
```
</details>

---

**Exercise 3 (Medium):** Given a DataFrame of raw order rows (not pre-aggregated):
```python
orders = pd.DataFrame({
    "category": ["Electronics", "Electronics", "Home", "Home", "Outdoor"],
    "total": [45, 89, 34, 56, 22],
})
```
first aggregate to get the SUM of `total` per `category` using pandas, then create an interactive bar chart of the aggregated result.

<details>
<summary>Hint</summary>
`orders.groupby("category", as_index=False)["total"].sum()`, then `px.bar()` on the result.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import plotly.express as px

orders = pd.DataFrame({
    "category": ["Electronics", "Electronics", "Home", "Home", "Outdoor"],
    "total": [45, 89, 34, 56, 22],
})

category_totals = orders.groupby("category", as_index=False)["total"].sum()
fig = px.bar(category_totals, x="category", y="total", title="Total by Category")
fig.show()
```
</details>

---

**Exercise 4 (Medium):** Given a DataFrame with a `product_name` column in addition to `price` and `rating`, build a scatter plot as in Exercise 2, but add `hover_data=["product_name"]` so hovering over each point also shows the product name. Also save the chart to `product_scatter.html`.

<details>
<summary>Hint</summary>
`px.scatter(df, x="price", y="rating", hover_data=["product_name"])`, then `fig.write_html(...)`.
</details>

<details>
<summary>Solution</summary>

```python
import pandas as pd
import plotly.express as px

df = pd.DataFrame({
    "product_name": ["Mouse", "Keyboard", "Lamp", "Yoga Mat"],
    "price": [24.99, 59.99, 34.00, 19.99],
    "rating": [4.2, 4.5, 3.9, 4.7],
})

fig = px.scatter(df, x="price", y="rating", hover_data=["product_name"], title="Price vs. Rating")
fig.write_html("product_scatter.html")
fig.show()
```
</details>

---

**Exercise 5 (Hard):** Given synthetic order data:
```python
np.random.seed(12)
n = 150
orders = pd.DataFrame({
    "order_total": np.round(np.random.normal(70, 25, n).clip(5), 2),
    "rating": np.clip(np.random.normal(4.0, 0.8, n), 1, 5).round(1),
    "category": np.random.choice(["Electronics", "Home", "Outdoor"], n, p=[0.5, 0.3, 0.2]),
    "customer_tier": np.random.choice(["bronze", "silver", "gold"], n, p=[0.4, 0.4, 0.2]),
})
```
Build two interactive charts: (1) a scatter plot of `order_total` vs `rating`, colored by `category`, with `hover_data=["customer_tier"]` and a descriptive title; (2) a bar chart of the mean `order_total` grouped by `customer_tier` (aggregate first with pandas), with `text_auto=".2s"` to show values on the bars. Save both as separate standalone HTML files using `include_plotlyjs="cdn"` to keep file size small.

<details>
<summary>Hint</summary>
For the bar chart, remember to aggregate with `.groupby("customer_tier", as_index=False)["order_total"].mean()` first — passing raw rows produces a stacked bar chart, not a clean per-tier summary.
</details>

<details>
<summary>Solution</summary>

```python
import numpy as np
import pandas as pd
import plotly.express as px

np.random.seed(12)
n = 150
orders = pd.DataFrame({
    "order_total": np.round(np.random.normal(70, 25, n).clip(5), 2),
    "rating": np.clip(np.random.normal(4.0, 0.8, n), 1, 5).round(1),
    "category": np.random.choice(["Electronics", "Home", "Outdoor"], n, p=[0.5, 0.3, 0.2]),
    "customer_tier": np.random.choice(["bronze", "silver", "gold"], n, p=[0.4, 0.4, 0.2]),
})

# Chart 1: scatter
fig1 = px.scatter(
    orders,
    x="order_total",
    y="rating",
    color="category",
    hover_data=["customer_tier"],
    title="Order Total vs. Rating, Colored by Category",
)
fig1.write_html("order_scatter.html", include_plotlyjs="cdn")
fig1.show()

# Chart 2: bar chart of mean order_total by tier
tier_avg = orders.groupby("customer_tier", as_index=False)["order_total"].mean()
fig2 = px.bar(
    tier_avg,
    x="customer_tier",
    y="order_total",
    title="Average Order Total by Customer Tier",
    text_auto=".2s",
)
fig2.write_html("tier_avg_bar.html", include_plotlyjs="cdn")
fig2.show()
```
</details>

---

**Previous lesson:** [6.4 — Customization: Labels, Legends, Styling & Saving](lesson-6.4-customization-and-saving.md)
**Next lesson:** [6.6 — Choosing the Right Chart](lesson-6.6-choosing-the-right-chart.md)
