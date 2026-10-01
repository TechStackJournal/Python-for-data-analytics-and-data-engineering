# Lesson 2.7 — File Paths with `pathlib`

## CONCEPT

`pathlib` is Python's modern, object-oriented module for working with file system paths — replacing older, error-prone string-based path handling (`os.path`) with a `Path` object that has useful methods and behaves consistently across Windows, macOS, and Linux.

```python
from pathlib import Path

data_dir = Path("data")
file_path = data_dir / "orders.csv"    # the / operator joins paths — works on every OS
```

A `Path` object represents a location in the file system and knows how to check if it exists, list what's inside it (if it's a folder), read/write its contents, and correctly build up paths without you worrying about `/` vs `\`.

## WHY IT MATTERS

Every pipeline reads from and writes to files — raw data drops, output reports, log files, config files. Hardcoded path strings like `"data/raw/orders.csv"` break the moment the script runs on a different operating system (Windows uses `\`, everything else uses `/`) or a different machine with a different folder structure. `pathlib` solves this at the source: paths built with `/` work correctly everywhere, and common operations (does this file exist? what's its extension? list every CSV in this folder?) become one-liners instead of fragile string manipulation.

## SYNTAX REFERENCE

```python
from pathlib import Path

# Creating paths
p = Path("data/orders.csv")
p = Path.cwd()                      # current working directory
p = Path.home()                     # user's home directory

# Joining paths (the / operator — works cross-platform)
data_dir = Path("data")
raw_dir = data_dir / "raw"
file_path = raw_dir / "orders.csv"  # data/raw/orders.csv (or data\raw\orders.csv on Windows)

# Inspecting a path
p.exists()          # True/False — does this exist on disk?
p.is_file()          # True/False
p.is_dir()           # True/False
p.name               # "orders.csv" — the final component
p.stem               # "orders" — name without extension
p.suffix             # ".csv" — the extension
p.parent             # the containing folder, as a Path

# Creating directories
Path("data/raw").mkdir(parents=True, exist_ok=True)
# parents=True: create intermediate folders too
# exist_ok=True: don't error if it already exists

# Listing contents of a directory
for item in Path("data").iterdir():
    print(item)

# Finding files matching a pattern
for csv_file in Path("data").glob("*.csv"):        # only this folder
    print(csv_file)

for csv_file in Path("data").rglob("*.csv"):       # this folder AND subfolders, recursively
    print(csv_file)

# Reading/writing text directly (shortcuts — no manual open()/close() needed)
content = Path("notes.txt").read_text()
Path("notes.txt").write_text("Hello, world!")

# Converting to a plain string, if a library specifically requires one
str(p)
```

## WORKED EXAMPLE

Setting up a small project's folder structure, then discovering and processing every CSV file in a `raw` folder:

```python
from pathlib import Path

# Build the folder structure for a project
project_root = Path("ecommerce_pipeline")
raw_dir = project_root / "data" / "raw"
processed_dir = project_root / "data" / "processed"

raw_dir.mkdir(parents=True, exist_ok=True)
processed_dir.mkdir(parents=True, exist_ok=True)

# Simulate creating a couple of raw data files
(raw_dir / "orders_2024_11.csv").write_text("id,total\n1,45.99\n2,89.00\n")
(raw_dir / "orders_2024_12.csv").write_text("id,total\n3,120.50\n")
(raw_dir / "readme.txt").write_text("This folder contains raw order exports.")

# Discover every CSV in the raw folder (ignoring the readme.txt)
csv_files = list(raw_dir.glob("*.csv"))
print(f"Found {len(csv_files)} CSV files:")
for f in csv_files:
    print(f"  {f.name} ({f.stat().st_size} bytes)")

# Process each file: read it, and write a "processed" marker file alongside it
for csv_file in csv_files:
    content = csv_file.read_text()
    line_count = len(content.strip().split("\n")) - 1   # minus the header row
    print(f"{csv_file.name}: {line_count} data rows")

    # Build the output path by combining the processed_dir with the same filename
    output_path = processed_dir / csv_file.name
    output_path.write_text(content)   # (a stand-in for real cleaning logic)

print("\nProcessed files now in:", processed_dir)
for f in processed_dir.iterdir():
    print(f"  {f}")
```

## COMMON MISTAKES

**1. Building paths with manual string concatenation**
```python
path = "data" + "/" + "raw" + "/" + "orders.csv"   # breaks on Windows (needs \)
```
Fix: always use `Path` objects and the `/` operator — `Path("data") / "raw" / "orders.csv"` — which `pathlib` translates to the correct separator for whatever OS the code runs on.

**2. Forgetting `mkdir()` fails if the parent folders don't exist yet**
```python
Path("data/raw/2024").mkdir()   # FileNotFoundError if "data/raw" doesn't already exist
```
Fix: use `parents=True` to create every missing intermediate folder in one call.

**3. Calling `mkdir()` on a folder that already exists, without `exist_ok=True`**
```python
Path("data").mkdir()   # FileExistsError if "data" already exists
```
Fix: `exist_ok=True` makes the call a no-op instead of erroring when the folder is already there — usually what you want in a script that might run more than once.

**4. Confusing `.glob("*.csv")` (this folder only) with `.rglob("*.csv")` (recursive, includes subfolders)**
If your files live in nested subfolders and you use `.glob()`, you'll silently get an incomplete list with no error — it simply won't look inside subfolders.

**5. Passing a `Path` object to older code that strictly expects a string**
Most modern libraries accept `Path` objects directly, but a few older ones don't. Fix: wrap it explicitly with `str(p)` if you hit a `TypeError` from a library that expected a plain string.

**Data Engineer Tip:** Scheduled pipelines commonly need to discover *which* files landed in a drop folder before processing them (`raw_dir.glob("*.csv")` from the worked example is exactly this pattern), and to build output paths that don't assume a specific operating system — since your pipeline might be written on a Mac, tested in a Linux Docker container, and deployed to an Azure environment, all in the same day. `pathlib`'s cross-platform behavior is precisely why it's preferred over manual string paths in production code.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create a `Path` object for `"data/orders.csv"` and print its `.name`, `.stem`, and `.suffix`.

<details>
<summary>Hint</summary>
`Path("data/orders.csv")`, then access each attribute.
</details>

<details>
<summary>Solution</summary>

```python
from pathlib import Path

p = Path("data/orders.csv")
print(p.name)    # "orders.csv"
print(p.stem)    # "orders"
print(p.suffix)  # ".csv"
```
</details>

---

**Exercise 2 (Easy):** Create a folder called `test_output` (with any missing parent folders, and without erroring if it already exists), then confirm it exists with `.exists()`.

<details>
<summary>Hint</summary>
`Path("test_output").mkdir(parents=True, exist_ok=True)`
</details>

<details>
<summary>Solution</summary>

```python
from pathlib import Path

output_dir = Path("test_output")
output_dir.mkdir(parents=True, exist_ok=True)
print(output_dir.exists())   # True
```
</details>

---

**Exercise 3 (Medium):** Inside a folder `sample_data` (create it if needed), create three files: `a.csv`, `b.csv`, and `c.txt` (any content). Then use `.glob("*.csv")` to list only the CSV files and print their names.

<details>
<summary>Hint</summary>
Use `.write_text()` to quickly create each file, then `.glob("*.csv")` to filter.
</details>

<details>
<summary>Solution</summary>

```python
from pathlib import Path

folder = Path("sample_data")
folder.mkdir(exist_ok=True)

(folder / "a.csv").write_text("id,val\n1,10\n")
(folder / "b.csv").write_text("id,val\n2,20\n")
(folder / "c.txt").write_text("just some notes")

csv_files = list(folder.glob("*.csv"))
for f in csv_files:
    print(f.name)
# a.csv
# b.csv
```
</details>

---

**Exercise 4 (Medium):** Build a path by joining three parts — `"project"`, `"data"`, `"raw"` — using the `/` operator, print the resulting path, and print its `.parent`.

<details>
<summary>Hint</summary>
`Path("project") / "data" / "raw"`
</details>

<details>
<summary>Solution</summary>

```python
from pathlib import Path

full_path = Path("project") / "data" / "raw"
print(full_path)          # project/data/raw
print(full_path.parent)   # project/data
```
</details>

---

**Exercise 5 (Hard):** Simulate a small data drop: create a folder `incoming` containing 4 files — `orders_jan.csv`, `orders_feb.csv`, `customers.json`, and `readme.md` (any placeholder content is fine). Write code that: (1) finds only the `.csv` files using `glob`, (2) for each one, reads its content, counts the number of lines, and (3) moves a "processed" copy of each into a new `archive` folder (using `.write_text()` to the new location — `pathlib` doesn't move files directly without the extra `shutil` module, so writing a copy and printing a message is fine here). Print a summary of files processed and their line counts.

<details>
<summary>Hint</summary>
Create `incoming` and `archive` folders with `mkdir(exist_ok=True)`. Use `.glob("*.csv")` to find only CSVs, ignoring the `.json` and `.md` files. For each match, `.read_text()`, count lines with `.split("\n")`, then `.write_text()` to a path built from `archive / csv_file.name`.
</details>

<details>
<summary>Solution</summary>

```python
from pathlib import Path

incoming = Path("incoming")
archive = Path("archive")
incoming.mkdir(exist_ok=True)
archive.mkdir(exist_ok=True)

# Simulate the data drop
(incoming / "orders_jan.csv").write_text("id,total\n1,45.99\n2,89.00\n")
(incoming / "orders_feb.csv").write_text("id,total\n3,120.50\n4,15.00\n5,60.00\n")
(incoming / "customers.json").write_text('{"id": 1, "name": "Maria"}')
(incoming / "readme.md").write_text("# Data drop notes")

csv_files = list(incoming.glob("*.csv"))
print(f"Found {len(csv_files)} CSV files to process.")

for csv_file in csv_files:
    content = csv_file.read_text()
    line_count = len(content.strip().split("\n")) - 1   # exclude header

    archive_path = archive / csv_file.name
    archive_path.write_text(content)

    print(f"{csv_file.name}: {line_count} rows -> archived to {archive_path}")

# Found 2 CSV files to process.
# orders_jan.csv: 2 rows -> archived to archive/orders_jan.csv
# orders_feb.csv: 3 rows -> archived to archive/orders_feb.csv
```
</details>

---

**Previous lesson:** [2.6 — Reading & Writing JSON](lesson-2.6-json.md)
**Next lesson:** [2.8 — Regular Expressions for Text Cleaning](lesson-2.8-regex.md)
