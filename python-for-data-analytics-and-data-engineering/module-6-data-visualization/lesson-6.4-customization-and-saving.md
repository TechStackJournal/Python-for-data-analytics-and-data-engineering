# Lesson 6.4 — Customization: Labels, Legends, Styling & Saving

## CONCEPT

A technically correct chart that's hard to read fails at its actual job. This lesson covers the finishing touches that turn a default matplotlib/seaborn chart into something genuinely presentation-ready: clear titles and axis labels, well-placed legends, sensible color choices, annotations calling out specific points, and exporting the result as a shareable file.

```python
ax.set_title("Monthly Revenue, 2024", fontsize=14, fontweight="bold")
ax.set_xlabel("Month")
ax.legend(loc="upper left")
fig.savefig("revenue_2024.png", dpi=300, bbox_inches="tight")
```

## WHY IT MATTERS

A chart is a communication tool, and most of what makes a chart genuinely useful to a non-technical stakeholder happens in this lesson, not the previous ones. An unlabeled axis, a missing title, or a chart cut off mid-save when exported all undermine trust in otherwise solid analysis — a portfolio project with polished charts reads as noticeably more professional than the same analysis with matplotlib's bare defaults.

## SYNTAX REFERENCE

```python
import matplotlib.pyplot as plt

fig, ax = plt.subplots(figsize=(8, 5))
ax.plot(months, revenue, marker="o", label="Revenue")

# Titles and labels with font control
ax.set_title("Monthly Revenue, 2024", fontsize=14, fontweight="bold")
ax.set_xlabel("Month", fontsize=11)
ax.set_ylabel("Revenue ($)", fontsize=11)

# Legend placement
ax.legend(loc="upper left")           # options: "upper right", "lower left", "best", etc.
ax.legend(loc="upper left", frameon=False)   # remove the legend's box border

# Rotating x-axis labels (helpful for long category names or many dates)
ax.set_xticklabels(months, rotation=45, ha="right")

# Formatting axis tick labels (e.g. as currency)
ax.set_yticklabels([f"${int(y):,}" for y in ax.get_yticks()])

# Adding gridlines
ax.grid(True, alpha=0.3)             # light, semi-transparent gridlines
ax.grid(axis="y")                      # gridlines on the y-axis only

# Annotating a specific point (e.g. calling out a peak)
ax.annotate(
    "Peak month",
    xy=(4, 21000),                 # the point being annotated
    xytext=(3, 23000),               # where the text label goes
    arrowprops=dict(arrowstyle="->"),
)

# Adding a horizontal/vertical reference line
ax.axhline(y=15000, color="gray", linestyle="--", label="Target")

# Color and style control
ax.plot(x, y, color="#3366cc", linewidth=2, linestyle="-")
ax.bar(categories, values, color=["steelblue", "coral", "seagreen"])   # different color per bar

# Applying a global style (affects every chart made afterward)
plt.style.use("seaborn-v0_8-whitegrid")   # a built-in matplotlib style theme
print(plt.style.available)                  # list every available built-in style

# Saving to a file
fig.savefig("chart.png")                                    # basic save
fig.savefig("chart.png", dpi=300)                              # higher resolution, print-quality
fig.savefig("chart.png", dpi=300, bbox_inches="tight")           # trims excess whitespace
fig.savefig("chart.svg")                                          # vector format — scales without blurring
fig.savefig("chart.pdf")                                            # good for embedding in a document
```

## WORKED EXAMPLE

Taking a plain default chart and progressively polishing it into a presentation-ready version:

```python
import matplotlib.pyplot as plt

months = ["Jan", "Feb", "Mar", "Apr", "May", "Jun"]
revenue = [12000, 15000, 11000, 18000, 21000, 19500]
target = 15000

# --- Version 1: the bare, unstyled default ---
fig1, ax1 = plt.subplots()
ax1.plot(months, revenue)
plt.show()
# Functional, but generic — no title, no units on the y-axis, default blue line

# --- Version 2: fully polished for a stakeholder-facing report ---
plt.style.use("seaborn-v0_8-whitegrid")

fig2, ax2 = plt.subplots(figsize=(9, 5.5))

ax2.plot(months, revenue, marker="o", color="#2c5f8a", linewidth=2.5, label="Actual Revenue")
ax2.axhline(y=target, color="gray", linestyle="--", linewidth=1.5, label=f"Target (${target:,})")

# Annotate the peak month specifically
peak_index = revenue.index(max(revenue))
ax2.annotate(
    f"Peak: ${max(revenue):,}",
    xy=(peak_index, max(revenue)),
    xytext=(peak_index - 1.3, max(revenue) + 1200),
    arrowprops=dict(arrowstyle="->", color="black"),
    fontsize=10,
)

ax2.set_title("Monthly Revenue vs. Target, H1 2024", fontsize=14, fontweight="bold")
ax2.set_xlabel("Month", fontsize=11)
ax2.set_ylabel("Revenue ($)", fontsize=11)
ax2.set_yticklabels([f"${int(y):,}" for y in ax2.get_yticks()])
ax2.legend(loc="lower right", frameon=True)
ax2.grid(True, alpha=0.3)

fig2.tight_layout()
fig2.savefig("h1_2024_revenue.png", dpi=300, bbox_inches="tight")
plt.show()
# A noticeably more polished chart: a styled background grid, currency-formatted
# y-axis labels, a dashed reference line for the target, an arrow calling out
# the peak month specifically, a bold title, and a legend distinguishing the
# actual line from the target line — genuinely presentable to a non-technical
# audience, and saved as a high-resolution file ready to share
```

## COMMON MISTAKES

**1. Saving a figure AFTER calling `plt.show()`**
```python
plt.show()                       # this can clear/close the figure in some environments
fig.savefig("chart.png")           # may save a blank or missing chart!
```
Fix: call `fig.savefig()` *before* `plt.show()` to be safe across different environments and matplotlib backends.

**2. Forgetting `bbox_inches="tight"` and getting a cropped or oddly-padded saved image**
Without it, titles, rotated tick labels, or legends placed outside the main plot area can get cut off in the saved file even though they looked fine in the interactive window. Fix: make `bbox_inches="tight"` close to a default habit for any chart you're saving to share.

**3. Using low `dpi` for a saved chart meant for a document or presentation**
```python
fig.savefig("chart.png")   # default dpi is often too low for print or a polished slide deck
```
Fix: `dpi=300` is a common professional standard for print-quality images; `dpi=150` is usually sufficient for on-screen use (a webpage, a chat message) and keeps the file size smaller.

**4. Choosing colors that don't work for colorblind viewers, or relying on color alone to convey meaning**
A red/green-only distinction (a very common default choice) is invisible to a meaningful fraction of viewers with red-green color blindness. Fix: for anything meant for a wide audience, prefer perceptually distinct palettes (seaborn's default palettes are generally colorblind-considerate), and where practical, reinforce a color distinction with a second cue too (different line styles, direct labels, or markers) rather than color alone.

**5. Over-annotating a chart until it becomes cluttered**
Adding an annotation to every single data point defeats the purpose — annotations are most effective when used sparingly, to call out one or two genuinely notable points (a peak, a specific anomaly), not as a substitute for a data table.

**Data Engineer Tip:** `dpi` and file format choices matter for automated reporting specifically: a PNG embedded in an automatically-generated email needs a moderate `dpi` (150-ish) to keep file/email size reasonable, while a chart destined for a printed or PDF report benefits from `dpi=300` and possibly a vector format (`.svg`/`.pdf`) that stays sharp at any zoom level. Building this judgment now avoids a common rough edge in early automated-reporting scripts, where charts either look blurry in print or bloat an email's size unnecessarily.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Given a simple line chart of `x = [1, 2, 3, 4]`, `y = [10, 15, 12, 20]`, add a bold, size-14 title `"Sample Chart"` and axis labels, then save it to `sample_chart.png` at 150 dpi.

<details>
<summary>Hint</summary>
`ax.set_title(..., fontsize=14, fontweight="bold")`, then `fig.savefig("sample_chart.png", dpi=150)`.
</details>

<details>
<summary>Solution</summary>

```python
import matplotlib.pyplot as plt

x = [1, 2, 3, 4]
y = [10, 15, 12, 20]

fig, ax = plt.subplots()
ax.plot(x, y, marker="o")
ax.set_title("Sample Chart", fontsize=14, fontweight="bold")
ax.set_xlabel("X")
ax.set_ylabel("Y")
fig.savefig("sample_chart.png", dpi=150)
plt.show()
```
</details>

---

**Exercise 2 (Easy):** Given two lines plotted with `label="Series A"` and `label="Series B"`, add a legend positioned in the `"upper left"` corner.

<details>
<summary>Hint</summary>
`ax.legend(loc="upper left")`
</details>

<details>
<summary>Solution</summary>

```python
import matplotlib.pyplot as plt

x = [1, 2, 3, 4]
a = [10, 15, 12, 20]
b = [8, 9, 14, 11]

fig, ax = plt.subplots()
ax.plot(x, a, label="Series A")
ax.plot(x, b, label="Series B")
ax.legend(loc="upper left")
plt.show()
```
</details>

---

**Exercise 3 (Medium):** Given `categories = ["North America", "South America", "Europe", "Asia Pacific"]` and `values = [4500, 1200, 3800, 5100]`, create a bar chart, rotate the x-axis labels 45 degrees so the longer names don't overlap, and add a light horizontal gridline on the y-axis only.

<details>
<summary>Hint</summary>
`ax.set_xticklabels(categories, rotation=45, ha="right")` and `ax.grid(axis="y", alpha=0.3)`.
</details>

<details>
<summary>Solution</summary>

```python
import matplotlib.pyplot as plt

categories = ["North America", "South America", "Europe", "Asia Pacific"]
values = [4500, 1200, 3800, 5100]

fig, ax = plt.subplots(figsize=(8, 5))
ax.bar(categories, values, color="steelblue")
ax.set_xticklabels(categories, rotation=45, ha="right")
ax.grid(axis="y", alpha=0.3)
ax.set_title("Revenue by Region")
fig.tight_layout()
plt.show()
```
</details>

---

**Exercise 4 (Medium):** Given `months = ["Jan", "Feb", "Mar", "Apr", "May"]` and `revenue = [8000, 9500, 7200, 11000, 9800]`, add a dashed horizontal reference line at `y=9000` labeled `"Target"`, and annotate the highest-revenue month with an arrow pointing to it, labeled with its exact value.

<details>
<summary>Hint</summary>
`ax.axhline(y=9000, linestyle="--", label="Target")` for the reference line; `ax.annotate()` with `xy=` at the peak point, using `months.index()`/`max()` to find it.
</details>

<details>
<summary>Solution</summary>

```python
import matplotlib.pyplot as plt

months = ["Jan", "Feb", "Mar", "Apr", "May"]
revenue = [8000, 9500, 7200, 11000, 9800]

fig, ax = plt.subplots(figsize=(8, 5))
ax.plot(months, revenue, marker="o", color="steelblue", label="Revenue")
ax.axhline(y=9000, color="gray", linestyle="--", label="Target")

peak_idx = revenue.index(max(revenue))
ax.annotate(
    f"${max(revenue):,}",
    xy=(peak_idx, max(revenue)),
    xytext=(peak_idx - 0.8, max(revenue) + 500),
    arrowprops=dict(arrowstyle="->"),
)

ax.set_title("Monthly Revenue vs. Target")
ax.legend()
plt.show()
```
</details>

---

**Exercise 5 (Hard):** Build a fully polished chart from this data:
```python
quarters = ["Q1", "Q2", "Q3", "Q4"]
this_year = [45000, 52000, 48000, 61000]
last_year = [38000, 44000, 41000, 50000]
```
Requirements: (1) apply the `"seaborn-v0_8-whitegrid"` style globally, (2) plot both years as lines with markers, distinct colors, and labels, (3) format the y-axis tick labels as currency (`$45,000` style), (4) add a bold title and axis labels, (5) place the legend in the best location automatically (`loc="best"`), (6) annotate the single highest value across BOTH years with an arrow, and (7) save the result as `quarterly_comparison.png` at 300 dpi with tight bounding box.

<details>
<summary>Hint</summary>
For currency formatting: `ax.set_yticklabels([f"${int(y):,}" for y in ax.get_yticks()])`. For the highest value across both series, compare `max(this_year)` and `max(last_year)` and figure out which quarter and series it came from before annotating.
</details>

<details>
<summary>Solution</summary>

```python
import matplotlib.pyplot as plt

plt.style.use("seaborn-v0_8-whitegrid")

quarters = ["Q1", "Q2", "Q3", "Q4"]
this_year = [45000, 52000, 48000, 61000]
last_year = [38000, 44000, 41000, 50000]

fig, ax = plt.subplots(figsize=(9, 5.5))

ax.plot(quarters, this_year, marker="o", color="#2c5f8a", linewidth=2.5, label="This Year")
ax.plot(quarters, last_year, marker="o", color="#c0743c", linewidth=2.5, label="Last Year")

ax.set_yticklabels([f"${int(y):,}" for y in ax.get_yticks()])
ax.set_title("Quarterly Revenue: This Year vs. Last Year", fontsize=14, fontweight="bold")
ax.set_xlabel("Quarter", fontsize=11)
ax.set_ylabel("Revenue ($)", fontsize=11)
ax.legend(loc="best")

# Highest value overall is in this_year, Q4 (61000)
peak_value = max(max(this_year), max(last_year))
peak_quarter_idx = this_year.index(peak_value) if peak_value in this_year else last_year.index(peak_value)

ax.annotate(
    f"Highest: ${peak_value:,}",
    xy=(peak_quarter_idx, peak_value),
    xytext=(peak_quarter_idx - 1.2, peak_value + 4000),
    arrowprops=dict(arrowstyle="->", color="black"),
    fontsize=10,
)

fig.tight_layout()
fig.savefig("quarterly_comparison.png", dpi=300, bbox_inches="tight")
plt.show()
```
</details>

---

**Previous lesson:** [6.3 — seaborn for Statistical Visualization](lesson-6.3-seaborn.md)
**Next lesson:** [6.5 — Intro to Interactive Visualization with Plotly](lesson-6.5-plotly-intro.md)
