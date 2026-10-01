# Lesson 6.6 — Choosing the Right Chart

## CONCEPT

Every chart type from this module answers a different *kind* of question. Choosing the right one isn't about which chart looks the most impressive — it's about matching the chart's structure to the structure of the question being asked. A simple, well-chosen bar chart communicates more clearly than an elaborate chart that's technically correct but structurally mismatched to the data.

This lesson doesn't introduce new syntax — it's a decision framework tying together every chart type from Lessons 6.1–6.5, plus honest coverage of common chart *misuse*.

## WHY IT MATTERS

A hiring manager or stakeholder reviewing your work will notice chart choice before they notice code quality — it's the most visible signal of analytical judgment in a portfolio project. Conversely, a poorly chosen chart (a pie chart with 15 slices, a 3D bar chart, a line connecting unrelated categories) is one of the fastest ways to undermine otherwise solid analysis. This lesson is deliberately about *judgment*, the harder and more transferable skill underneath all the syntax from this module.

## SYNTAX REFERENCE

A decision framework, not new syntax — organized by the question you're trying to answer:

```
QUESTION TYPE                          → CHART TYPE
─────────────────────────────────────────────────────────────
"How did X change over time?"          → Line chart
"How does X compare across categories?" → Bar chart (sorted, if ranking matters)
"What's the shape/spread of X?"         → Histogram, box plot, or violin plot
"Is there a relationship between        → Scatter plot (add trend line / regplot
 X and Y?"                                 if the relationship itself matters)
"What's the composition of a whole?"    → Stacked bar chart (small number of parts)
                                            — AVOID pie charts beyond 3-4 slices
"How do two numeric variables relate,   → Heatmap (for a matrix of many pairs)
 across many pairs?"                       or a small-multiples grid of scatters
"What does a single number look like    → A "big number" callout, or a simple
 compared to a target?"                    bar/gauge — not a full chart at all,
                                            often
"Does the audience need to explore      → Interactive (Plotly) — hover, zoom,
 the data themselves?"                     filter by legend
"Is this for print, a PDF report, or    → Static (matplotlib/seaborn), higher dpi
 a fixed slide?"
```

Common misuse patterns, worth naming explicitly:
```
# Pie chart with too many slices — nearly impossible to compare slice sizes visually
px.pie(df, names="category", values="total")   # fine for 3-4 categories, poor for 10+

# 3D bar/pie charts — the added dimension distorts perceived size/proportion,
# almost never worth it even when a tool makes it easy to add

# Truncated y-axis on a bar chart — starting the y-axis above zero exaggerates
# differences between bars and can visually mislead
ax.set_ylim(bottom=40000)   # be very deliberate and TRANSPARENT if you ever do this

# A line chart connecting genuinely unordered categories (Lesson 6.2, Mistake #1)
```

## WORKED EXAMPLE

Walking through the same dataset with several different real business questions, and matching each to its right chart — demonstrating the decision process, not just the final code:

```python
import matplotlib.pyplot as plt
import seaborn as sns
import numpy as np
import pandas as pd

np.random.seed(20)
n = 300
orders = pd.DataFrame({
    "order_date": pd.date_range("2024-01-01", periods=n, freq="D") % pd.Timedelta(days=180) + pd.Timestamp("2024-01-01"),
    "order_total": np.round(np.random.normal(75, 30, n).clip(5), 2),
    "category": np.random.choice(["Electronics", "Home", "Outdoor"], n, p=[0.5, 0.3, 0.2]),
    "rating": np.clip(np.random.normal(4.0, 0.8, n), 1, 5).round(1),
})
orders["month"] = orders["order_date"].dt.to_period("M").astype(str)

fig, axes = plt.subplots(nrows=2, ncols=2, figsize=(12, 9))

# Question 1: "How has revenue trended over the year?" -> a TREND over time -> LINE CHART
monthly_revenue = orders.groupby("month")["order_total"].sum()
axes[0, 0].plot(monthly_revenue.index, monthly_revenue.values, marker="o")
axes[0, 0].set_title("Q1: Revenue Trend (Line — right choice for a time trend)")
axes[0, 0].tick_params(axis="x", rotation=45)

# Question 2: "Which category generates the most revenue?" -> COMPARISON across
# discrete categories -> BAR CHART, sorted for a clear ranking
category_totals = orders.groupby("category")["order_total"].sum().sort_values(ascending=False)
axes[0, 1].bar(category_totals.index, category_totals.values, color="coral")
axes[0, 1].set_title("Q2: Revenue by Category (Sorted Bar — clear ranking)")

# Question 3: "What does a typical order look like, and how much do they vary?"
# -> SHAPE/SPREAD of one numeric variable -> HISTOGRAM
axes[1, 0].hist(orders["order_total"], bins=25, edgecolor="white", color="seagreen")
axes[1, 0].set_title("Q3: Order Total Distribution (Histogram — shows shape/spread)")

# Question 4: "Do higher-value orders tend to get better ratings?" -> RELATIONSHIP
# between two numeric variables -> SCATTER PLOT
axes[1, 1].scatter(orders["order_total"], orders["rating"], alpha=0.4)
axes[1, 1].set_title("Q4: Order Total vs. Rating (Scatter — reveals relationship, or lack of one)")

fig.tight_layout()
plt.show()

# A counter-example: what if we'd used the WRONG chart for Question 2?
fig2, ax2 = plt.subplots(figsize=(6, 4))
ax2.plot(category_totals.index, category_totals.values, marker="o")   # a LINE for 3 categories
ax2.set_title("Anti-example: Line chart for unordered categories (misleading)")
plt.show()
# This visually implies a continuous trend or progression from Electronics to
# Outdoor that doesn't actually exist — the categories have no inherent order
# or continuity between them, so a line here creates a false impression of
# connectedness. The bar chart in Question 2 is the correct choice.
```

## COMMON MISTAKES

**1. Choosing a chart type because it "looks impressive" rather than because it fits the question**
A 3D exploded pie chart might look more sophisticated than a plain bar chart, but if it makes the actual comparison harder to read accurately, it's a worse chart — not a better one. Fix: always start from the question, not the chart type — "what am I actually trying to show?" comes first, "which chart shows that most clearly?" comes second.

**2. Defaulting to a pie chart out of habit for any "parts of a whole" question**
Pie charts are genuinely hard for the human eye to compare accurately once there are more than 3–4 slices, or when slices are close in size — a sorted bar chart almost always communicates the same "composition" information more clearly and precisely. Fix: reach for a bar chart by default for category composition; reserve pie charts for genuinely simple 2–4 category breakdowns where the "whole" framing specifically matters.

**3. Truncating a bar chart's y-axis without clearly flagging it**
Starting a bar chart's y-axis at, say, 40,000 instead of 0 visually exaggerates the *relative* difference between bars — two bars might look like one is "twice as tall" when the underlying values differ by only 10%. Fix: bar charts should almost always start at zero, since bar height is the visual cue being compared; if you have a specific, deliberate reason to zoom in (comparing very close values), say so explicitly in the chart or its caption.

**4. Cramming too many series or categories into one chart "to be thorough"**
A line chart with 15 overlapping series, or a bar chart with 40 categories crammed along the x-axis, usually communicates *less* than a more focused chart — Fix: prioritize the handful of series/categories that actually matter to the question, and either group the rest into an "Other" category or move the full detail into a table alongside the chart, rather than forcing everything into one visual.

**5. Forgetting the audience when choosing between static and interactive**
An interactive Plotly chart is wasted in a printed PDF report (a viewer can't hover on paper), while a static matplotlib chart in a live dashboard forces a stakeholder to ask you for every follow-up cut of the data instead of exploring it themselves. Fix: match the delivery format to how the chart will actually be consumed — this is a legitimate part of "choosing the right chart," not a separate concern.

**Data Engineer Tip:** In an automated reporting pipeline (Module 9), chart choice often needs to be decided *once*, in code, and then applied consistently every time the report regenerates — you won't be there to manually pick the best chart type each run. This makes the judgment in this lesson worth encoding explicitly as a rule ("always use a sorted bar chart for this specific category breakdown") rather than a one-off decision, so the automated output stays reliably readable without your ongoing intervention.

## PRACTICE EXERCISES

These exercises focus on judgment rather than new syntax — for each scenario, identify the best chart type and briefly justify the choice, then implement it.

**Exercise 1 (Easy):** You have `customer_signups = pd.Series([120, 145, 130, 160, 175, 190], index=["Jan", "Feb", "Mar", "Apr", "May", "Jun"])` and want to show how signups have changed over the first half of the year. Which chart type fits, and why? Implement it.

<details>
<summary>Hint</summary>
This is a trend over a continuous time sequence.
</details>

<details>
<summary>Solution</summary>

A **line chart** is correct — this is a trend over time, and a line makes the month-to-month direction and overall trajectory immediately clear.

```python
import matplotlib.pyplot as plt
import pandas as pd

customer_signups = pd.Series([120, 145, 130, 160, 175, 190], index=["Jan", "Feb", "Mar", "Apr", "May", "Jun"])

fig, ax = plt.subplots()
ax.plot(customer_signups.index, customer_signups.values, marker="o")
ax.set_title("Customer Signups, Jan–Jun")
plt.show()
```
</details>

---

**Exercise 2 (Easy):** You have total revenue for 4 product categories and want a stakeholder to be able to immediately see which category performs best, ranked from highest to lowest. Which chart type fits, and why? Implement it with `data = {"Electronics": 45000, "Home": 28000, "Outdoor": 15000, "Books": 9000}`.

<details>
<summary>Hint</summary>
This is a ranked comparison across discrete categories.
</details>

<details>
<summary>Solution</summary>

A **sorted bar chart** is correct — categories have no inherent order, and sorting by value makes the ranking immediately visible without the reader needing to compare bar heights manually.

```python
import matplotlib.pyplot as plt

data = {"Electronics": 45000, "Home": 28000, "Outdoor": 15000, "Books": 9000}
sorted_items = sorted(data.items(), key=lambda x: x[1], reverse=True)
categories = [item[0] for item in sorted_items]
values = [item[1] for item in sorted_items]

fig, ax = plt.subplots()
ax.bar(categories, values, color="steelblue")
ax.set_title("Revenue by Category (Ranked)")
plt.show()
```
</details>

---

**Exercise 3 (Medium):** You want to understand whether customers who spend more also tend to leave higher ratings, using a dataset of 200 orders with `order_total` and `rating` columns. Which chart type fits, and why? Additionally, name one thing you'd check for BEFORE trusting any pattern you see in this chart.

<details>
<summary>Hint</summary>
This is a relationship between two numeric variables. For the caveat, think about outliers, sample size, or confounding factors.
</details>

<details>
<summary>Solution</summary>

A **scatter plot** is correct — this asks about the relationship between two numeric variables, which a scatter plot shows directly, point by point. Optionally, a fitted trend line (`sns.regplot()`) can make a weak-but-real relationship easier to see.

Before trusting a pattern in this chart, worth checking: whether a few extreme outliers are driving an apparent trend that doesn't hold for the bulk of the data; whether the sample size is large enough for the pattern to be meaningful rather than noise; and whether a third variable (e.g., product category) might be a confounding factor behind an apparent relationship, rather than order total causally affecting rating.

```python
import seaborn as sns
import matplotlib.pyplot as plt
import numpy as np
import pandas as pd

np.random.seed(30)
orders = pd.DataFrame({
    "order_total": np.round(np.random.normal(75, 30, 200).clip(5), 2),
    "rating": np.clip(np.random.normal(4.0, 0.8, 200), 1, 5).round(1),
})

sns.regplot(data=orders, x="order_total", y="rating", scatter_kws={"alpha": 0.4})
plt.title("Order Total vs. Rating")
plt.show()
```
</details>

---

**Exercise 4 (Medium):** Someone on your team has built a 3D pie chart showing revenue split across 12 product subcategories, and asks for your feedback before they present it. What's your critique, and what would you suggest instead? (No code needed for this one — write your answer as a short paragraph, as if actually giving this feedback.)

<details>
<summary>Hint</summary>
Think about both the "12 slices" problem and the "3D" problem separately.
</details>

<details>
<summary>Solution</summary>

Two separate issues are stacking here. First, 12 slices is well past the point where a pie chart is readable — the human eye is genuinely bad at comparing the angular size of more than 3–4 wedges, especially when several are close in size, so most of the audience won't be able to accurately rank the subcategories just by looking at it. Second, the 3D effect distorts the *apparent* size of slices further, depending on their position and rotation — a slice nearer the "front" of the 3D pie visually reads as larger than an equal-sized slice further back, which is actively misleading rather than just unclear. A sorted horizontal bar chart (`barh`, sorted descending) would let the audience read the actual ranking and relative sizes accurately at a glance, and would also scale cleanly if a 13th subcategory gets added next quarter — something a pie chart increasingly struggles with as more slices are added.
</details>

---

**Exercise 5 (Hard):** You're preparing two deliverables from the same underlying order dataset: (1) a one-page PDF summary that will be printed and handed to executives in a meeting, and (2) an internal analytics page that other analysts will use to explore the data themselves, filtering by date range and category as needed. For each deliverable, decide: static or interactive, which specific library, and briefly justify both choices given the constraints of how each will actually be used.

<details>
<summary>Hint</summary>
Think about who's consuming each one, and whether the constraint (printed paper vs. an interactive web page) rules anything out entirely.
</details>

<details>
<summary>Solution</summary>

**Deliverable 1 (printed PDF for executives):** **Static**, using **matplotlib/seaborn**, saved at high `dpi` (300) via `fig.savefig(..., dpi=300, bbox_inches="tight")`. Interactivity is simply wasted on paper — nobody can hover or click a printed page — so a Plotly chart provides no benefit here and adds unnecessary complexity. A clean, well-labeled, high-resolution static chart, focused on the 2-3 numbers that matter most for the meeting, is the right fit. Since this is a one-time, fixed snapshot (not something the audience will interact with later), the chart should be self-explanatory without requiring anyone to zoom or hover to understand it.

**Deliverable 2 (internal exploration page):** **Interactive**, using **Plotly**, likely exported via `fig.write_html()` or embedded in a lightweight internal web page. Since other analysts specifically need to filter by date range and category themselves, static images would force them to come back to you for every new cut of the data — defeating the purpose of a self-serve analytics page. Plotly's built-in hover, zoom, and legend-toggle behavior handles a meaningful chunk of that exploration need out of the box; if filtering by date range/category needs to go beyond what Plotly's defaults offer, that's a sign this deliverable may eventually want a proper dashboarding tool built on top of Plotly, rather than a single static export.
</details>

---

## Module 6 Complete

You've now covered the full visualization toolkit: matplotlib fundamentals, the four core chart types, seaborn for statistical visualization, professional polish and export, interactive Plotly charts, and — most importantly — the judgment to choose the right one for a given question. Module 7 shifts from visualizing data you already have to *getting* data programmatically, from databases and APIs.

**Previous lesson:** [6.5 — Intro to Interactive Visualization with Plotly](lesson-6.5-plotly-intro.md)
**Next module:** [Module 7 — Python ↔ Databases & APIs](../module-7-databases-and-apis/README.md)
