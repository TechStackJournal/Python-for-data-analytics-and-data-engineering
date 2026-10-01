# Lesson 6.1 — matplotlib Fundamentals

## CONCEPT

**matplotlib** is Python's foundational plotting library — nearly every other visualization tool in the Python ecosystem (including pandas' built-in `.plot()` and seaborn) is built on top of it or interoperates closely with it. Every matplotlib chart is built from two core objects: a **Figure** (the overall canvas/window) and one or more **Axes** (an individual plot area within that figure — confusingly, "Axes" is the *plot itself*, not to be confused with a single axis line).

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots()     # create one Figure containing one Axes
ax.plot([1, 2, 3], [10, 20, 15])
plt.show()
```

There are two ways to write matplotlib code — the **object-oriented interface** (`fig, ax = plt.subplots()`, then calling methods on `ax`) and the older **pyplot interface** (`plt.plot(...)` directly). This course uses the object-oriented style throughout, since it scales cleanly to multiple subplots and is considered the more maintainable modern convention.

## WHY IT MATTERS

Understanding Figure vs. Axes now prevents a lot of confusion later, especially once you're building multi-panel dashboards (several charts side by side) — a extremely common real deliverable. Every customization you'll do in this module (titles, labels, legends, colors) is a method call on an `Axes` object, so getting comfortable with this object model early makes everything downstream easier.

## SYNTAX REFERENCE

```python
import matplotlib.pyplot as plt

# Creating a figure with a single plot
fig, ax = plt.subplots()

# Creating a figure with multiple subplots (a grid)
fig, axes = plt.subplots(nrows=1, ncols=2)      # 1 row, 2 columns — axes is an array
fig, axes = plt.subplots(nrows=2, ncols=2)       # a 2x2 grid — axes is a 2D array

# Controlling the overall figure size (width, height in inches)
fig, ax = plt.subplots(figsize=(8, 5))

# Basic plotting on an Axes object
ax.plot(x_values, y_values)
ax.bar(categories, values)
ax.scatter(x_values, y_values)
ax.hist(values)

# Titles and labels
ax.set_title("Monthly Revenue")
ax.set_xlabel("Month")
ax.set_ylabel("Revenue ($)")

# Displaying or saving
plt.show()                          # opens an interactive window (script context)
fig.savefig("chart.png", dpi=150)     # saves to a file — covered fully in Lesson 6.4

# Accessing a specific subplot in a grid
fig, axes = plt.subplots(nrows=1, ncols=2)
axes[0].plot(x, y)          # left subplot
axes[1].bar(cats, vals)      # right subplot

# For a 2D grid of subplots
fig, axes = plt.subplots(nrows=2, ncols=2)
axes[0, 0].plot(x, y)         # top-left
axes[1, 1].hist(values)        # bottom-right

# Adjusting spacing so subplots don't overlap
fig.tight_layout()
```

## WORKED EXAMPLE

Building a simple single chart, then a 1x2 grid comparing two views of the same order data:

```python
import matplotlib.pyplot as plt

months = ["Jan", "Feb", "Mar", "Apr", "May", "Jun"]
revenue = [12000, 15000, 11000, 18000, 21000, 19500]

# --- A single chart ---
fig, ax = plt.subplots(figsize=(8, 5))
ax.plot(months, revenue, marker="o")
ax.set_title("Monthly Revenue")
ax.set_xlabel("Month")
ax.set_ylabel("Revenue ($)")
plt.show()
# Result: a line chart with 6 points, one per month, connected by a line,
# with small circular markers at each data point

# --- A 1x2 grid: line chart next to a bar chart of the same data ---
fig, axes = plt.subplots(nrows=1, ncols=2, figsize=(12, 5))

axes[0].plot(months, revenue, marker="o", color="steelblue")
axes[0].set_title("Revenue Trend")
axes[0].set_xlabel("Month")
axes[0].set_ylabel("Revenue ($)")

axes[1].bar(months, revenue, color="coral")
axes[1].set_title("Revenue by Month")
axes[1].set_xlabel("Month")
axes[1].set_ylabel("Revenue ($)")

fig.suptitle("Revenue: Two Views")   # a title for the WHOLE figure, above both subplots
fig.tight_layout()
plt.show()
# Result: one window containing two side-by-side charts — a line chart on the
# left showing the trend, a bar chart on the right showing the same values
# as discrete bars, with a shared overall title above both
```

## COMMON MISTAKES

**1. Confusing the pyplot interface (`plt.plot()`) with the object-oriented interface (`ax.plot()`)**
```python
plt.plot(x, y)        # pyplot style — works, but implicitly tracks "the current axes"
ax.plot(x, y)           # object-oriented style — explicit, preferred, especially with subplots
```
Mixing both styles in the same script (common when copying snippets from different tutorials online) can cause confusing bugs, especially with multiple subplots, since `plt.title()` (pyplot style) sets the title of whatever the "current" axes happens to be — which may not be the one you expect. Fix: pick the object-oriented style (`fig, ax = plt.subplots()`, then `ax.method()`) and use it consistently, as this course does throughout.

**2. Forgetting `plt.show()` in a plain script**
```python
fig, ax = plt.subplots()
ax.plot(x, y)
# script ends here — no window ever appears!
```
Fix: always end with `plt.show()` when running matplotlib from a `.py` file. (Jupyter-style notebooks display the chart automatically without needing this, which is a common source of confusion when code is copied between the two environments.)

**3. Indexing a subplot grid incorrectly**
```python
fig, axes = plt.subplots(nrows=2, ncols=2)
axes[2]   # IndexError — a 2x2 grid needs TWO indices: axes[row, col], not axes[2]
```
Fix: for a grid with more than one row AND more than one column, `axes` is a 2D array — use `axes[row, col]`. For a single row or single column (`nrows=1` or `ncols=1`), `axes` is a flat 1D array and a single index (`axes[0]`) works fine.

**4. Overlapping subplot titles/labels because of missing `tight_layout()`**
Without `fig.tight_layout()`, a multi-subplot figure often has titles, labels, or tick marks overlapping between adjacent subplots — especially with longer text. Fix: call `fig.tight_layout()` right before `plt.show()`/`savefig()` as a near-automatic habit for any multi-subplot figure.

**5. Reusing the same `fig`/`ax` across multiple unrelated charts without creating a new one**
```python
fig, ax = plt.subplots()
ax.plot(x1, y1)
plt.show()

ax.plot(x2, y2)   # this ADDS to the same Axes as before, doesn't start fresh!
plt.show()
```
Fix: call `plt.subplots()` again to get a genuinely new Figure/Axes for each new, unrelated chart — reusing `ax` is only intentional when you deliberately want multiple series layered on the same plot.

**Data Engineer Tip:** In a pipeline context, charts are frequently generated as part of an automated report (Module 9) — a script that runs on a schedule and saves a PNG or embeds a chart in an email/Slack message. Getting comfortable with the object-oriented `fig`/`ax` pattern now, rather than the simpler-looking-but-more-fragile pyplot shortcuts, pays off directly once you're generating charts unattended as part of a larger automated script, where predictable, explicit behavior matters more than convenience.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create a simple line chart of `x = [1, 2, 3, 4, 5]` against `y = [10, 15, 13, 18, 22]`, with a title `"Simple Trend"` and axis labels `"X"` and `"Y"`.

<details>
<summary>Hint</summary>
`fig, ax = plt.subplots()`, then `ax.plot(x, y)`, `ax.set_title()`, `ax.set_xlabel()`, `ax.set_ylabel()`.
</details>

<details>
<summary>Solution</summary>

```python
import matplotlib.pyplot as plt

x = [1, 2, 3, 4, 5]
y = [10, 15, 13, 18, 22]

fig, ax = plt.subplots()
ax.plot(x, y)
ax.set_title("Simple Trend")
ax.set_xlabel("X")
ax.set_ylabel("Y")
plt.show()
# A line connecting 5 points, rising overall with a small dip at the 3rd point
```
</details>

---

**Exercise 2 (Easy):** Create a figure with `figsize=(10, 4)` and a single bar chart of `categories = ["A", "B", "C"]` against `values = [30, 45, 20]`.

<details>
<summary>Hint</summary>
`plt.subplots(figsize=(10, 4))`, then `ax.bar(categories, values)`.
</details>

<details>
<summary>Solution</summary>

```python
import matplotlib.pyplot as plt

categories = ["A", "B", "C"]
values = [30, 45, 20]

fig, ax = plt.subplots(figsize=(10, 4))
ax.bar(categories, values)
ax.set_title("Values by Category")
plt.show()
# A wide, short chart with 3 vertical bars of heights 30, 45, and 20
```
</details>

---

**Exercise 3 (Medium):** Create a 1x2 subplot grid. In the left subplot, plot a line chart of `[1, 2, 3, 4]` vs. `[5, 3, 6, 2]`. In the right subplot, plot a bar chart of `["Q1", "Q2", "Q3", "Q4"]` vs. `[100, 150, 120, 180]`. Give each subplot its own title, and call `tight_layout()` before showing.

<details>
<summary>Hint</summary>
`fig, axes = plt.subplots(nrows=1, ncols=2)`, then `axes[0]` for the line, `axes[1]` for the bar.
</details>

<details>
<summary>Solution</summary>

```python
import matplotlib.pyplot as plt

fig, axes = plt.subplots(nrows=1, ncols=2, figsize=(10, 4))

axes[0].plot([1, 2, 3, 4], [5, 3, 6, 2])
axes[0].set_title("Line Chart")

axes[1].bar(["Q1", "Q2", "Q3", "Q4"], [100, 150, 120, 180])
axes[1].set_title("Quarterly Values")

fig.tight_layout()
plt.show()
```
</details>

---

**Exercise 4 (Medium):** Create a 2x2 subplot grid (4 panels total). Fill only the top-left (`[0,0]`) and bottom-right (`[1,1]`) panels with any simple line or bar chart of your choosing, each with a title. Leave the other two panels empty. Add a figure-level title using `fig.suptitle()`.

<details>
<summary>Hint</summary>
`plt.subplots(nrows=2, ncols=2)`, then index with `axes[0, 0]` and `axes[1, 1]`.
</details>

<details>
<summary>Solution</summary>

```python
import matplotlib.pyplot as plt

fig, axes = plt.subplots(nrows=2, ncols=2, figsize=(8, 6))

axes[0, 0].plot([1, 2, 3], [10, 20, 15])
axes[0, 0].set_title("Top Left")

axes[1, 1].bar(["A", "B"], [40, 25])
axes[1, 1].set_title("Bottom Right")

fig.suptitle("A Partially Filled Grid")
fig.tight_layout()
plt.show()
# A 2x2 grid where only the top-left and bottom-right panels contain
# visible charts; the other two are blank plot areas
```
</details>

---

**Exercise 5 (Hard):** Given synthetic monthly data:
```python
months = ["Jan", "Feb", "Mar", "Apr", "May", "Jun"]
revenue = [12000, 15000, 11000, 18000, 21000, 19500]
order_count = [120, 145, 110, 175, 200, 190]
```
Build a 1x2 subplot figure comparing revenue and order count side by side — left panel: a line chart of `revenue` over `months`; right panel: a line chart of `order_count` over `months`, in a different color. Give each subplot its own title and y-axis label (noting the units differ — dollars vs. a plain count), give the whole figure a shared title with `suptitle()`, and save the result to a file called `monthly_comparison.png` using `fig.savefig()` in addition to `plt.show()`.

<details>
<summary>Hint</summary>
Build the two-panel structure as in Exercise 3, using `color=` on each `.plot()` call to differentiate them, and call `fig.savefig("monthly_comparison.png")` before or after `plt.show()`.
</details>

<details>
<summary>Solution</summary>

```python
import matplotlib.pyplot as plt

months = ["Jan", "Feb", "Mar", "Apr", "May", "Jun"]
revenue = [12000, 15000, 11000, 18000, 21000, 19500]
order_count = [120, 145, 110, 175, 200, 190]

fig, axes = plt.subplots(nrows=1, ncols=2, figsize=(12, 5))

axes[0].plot(months, revenue, marker="o", color="steelblue")
axes[0].set_title("Monthly Revenue")
axes[0].set_xlabel("Month")
axes[0].set_ylabel("Revenue ($)")

axes[1].plot(months, order_count, marker="o", color="darkorange")
axes[1].set_title("Monthly Order Count")
axes[1].set_xlabel("Month")
axes[1].set_ylabel("Orders")

fig.suptitle("Revenue vs. Order Count, Jan–Jun")
fig.tight_layout()
fig.savefig("monthly_comparison.png", dpi=150)
plt.show()
# Two side-by-side line charts sharing the same x-axis months, one in blue
# tracking dollars, one in orange tracking a plain count — and a PNG file
# saved to disk with the same content
```
</details>

---

**Next lesson:** [6.2 — Line, Bar, Histogram & Scatter Plots](lesson-6.2-core-chart-types.md)
