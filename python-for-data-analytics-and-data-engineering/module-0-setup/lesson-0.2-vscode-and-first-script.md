# Lesson 0.2 — VS Code Setup & Running Your First Script

## CONCEPT

**VS Code** (Visual Studio Code) is a free, widely-used code editor with strong Python support via extensions. In this lesson you'll install the official Python extension, point VS Code at the virtual environment you created in Lesson 0.1, and run your first `.py` script two different ways: from the integrated terminal, and using VS Code's built-in "Run" button.

You'll also be introduced to the **interactive REPL** (Read-Eval-Print Loop) — a way to run Python commands one at a time and see results immediately, useful for quick experiments before committing code to a file.

## WHY IT MATTERS

A good editor setup removes friction from the actual learning — proper syntax highlighting, error underlining, and auto-formatting let you focus on *what* the code does instead of fighting typos and indentation issues (which matter a lot in Python, as you'll see in Module 1). Getting VS Code correctly linked to your project's virtual environment also avoids a very common beginner confusion: writing code that VS Code appears to accept, but that fails when actually run, because the editor was "looking at" a different Python installation than the one in your terminal.

## SYNTAX REFERENCE

**Open your project folder in VS Code from the terminal:**
```bash
code .
```
(If `code` isn't recognized, open VS Code manually and use File → Open Folder.)

**Select the correct Python interpreter in VS Code:**
- Open the Command Palette: `Cmd+Shift+P` (macOS) or `Ctrl+Shift+P` (Windows/Linux)
- Type and select: `Python: Select Interpreter`
- Choose the one whose path includes `.venv` (e.g. `./.venv/bin/python`)

**Run a script from VS Code's integrated terminal:**
```bash
python3 my_script.py
```

**Run a script using VS Code's Run button:**
- Open the `.py` file
- Click the ▶️ "Run Python File" button in the top-right corner (or `Ctrl+F5`)

**Open the interactive REPL (in a terminal, with your venv active):**
```bash
python3
```
```python
>>> 2 + 2
4
>>> print("hello")
hello
>>> exit()
```

**Run a specific block interactively in VS Code (Python extension feature):**
- Highlight a few lines of code
- Right-click → "Run Selection/Line in Python Terminal" (or `Shift+Enter`)

## WORKED EXAMPLE

A complete first-script walkthrough, start to finish:

```bash
# 1. Make sure your virtual environment is active (from Lesson 0.1)
cd python-course
source .venv/bin/activate

# 2. Open the folder in VS Code
code .
```

Inside VS Code:
1. Open the Command Palette (`Cmd+Shift+P` / `Ctrl+Shift+P`)
2. Run `Python: Select Interpreter`
3. Pick the interpreter under `.venv` — VS Code will remember this per-project going forward

Create a new file called `hello_pipeline.py` with this content:

```python
# hello_pipeline.py
# A tiny "pipeline" simulating what's ahead in this course

products = ["Wireless Mouse", "USB-C Cable", "Laptop Stand"]
prices = [24.99, 12.50, 39.00]

total_value = 0
for name, price in zip(products, prices):
    print(f"{name}: ${price:.2f}")
    total_value += price

print(f"\nTotal inventory value: ${total_value:.2f}")
```

Run it either way:

```bash
# From the integrated terminal
python3 hello_pipeline.py
```
or click the ▶️ Run button in the editor. Expected output:
```
Wireless Mouse: $24.99
USB-C Cable: $12.50
Laptop Stand: $39.00

Total inventory value: $76.49
```

Then try the REPL for a quick experiment, without creating a file at all:
```bash
python3
```
```python
>>> prices = [24.99, 12.50, 39.00]
>>> sum(prices)
76.49
>>> exit()
```

## COMMON MISTAKES

**1. VS Code using the wrong Python interpreter**
Symptoms: code that imports a package you know you installed shows a red squiggly "module not found" warning, or the wrong Python version is reported at the bottom of the VS Code window. Fix: re-run `Python: Select Interpreter` and confirm the `.venv` path is selected — check the bottom-right status bar in VS Code, which displays the active interpreter.

**2. Running the script from the wrong terminal (outside the virtual environment)**
If you open a *new* terminal window separately from VS Code's integrated one, it won't automatically have `.venv` active. Fix: always confirm `(.venv)` appears in the prompt before running anything, or use VS Code's integrated terminal, which activates the selected interpreter's environment automatically.

**3. Saving a file with the wrong extension**
A Python file must end in `.py` (not `.py.txt`, not no extension) for VS Code and the `python3` command to treat it correctly. Fix: double-check the filename in VS Code's tab, and make sure "Show file extensions" is enabled in your OS's file explorer/finder if you're renaming files outside VS Code.

**4. Confusing the REPL with a script file**
Code typed directly into the REPL is not saved anywhere — closing the terminal loses it. Fix: use the REPL for quick, disposable experiments; use `.py` files for anything you want to keep, edit, or rerun.

**Data Engineer Tip:** As pipelines grow, you'll rarely run a single `.py` file in isolation — production code gets triggered by a scheduler (cron, Airflow, Azure Data Factory) with no human watching a "Run" button. Getting comfortable running scripts from the command line now, not just clicking Run in an editor, builds the habit that scheduled automation depends on later (Module 9).

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Install the official Python extension in VS Code (search "Python" by Microsoft in the Extensions panel) and confirm it's enabled.

<details>
<summary>Hint</summary>
Open the Extensions panel with `Cmd+Shift+X` / `Ctrl+Shift+X`, search "Python," and install the one published by Microsoft.
</details>

<details>
<summary>Solution</summary>
No code — verification step. After installing, opening any `.py` file should show syntax highlighting and a Python version indicator in the bottom status bar.
</details>

---

**Exercise 2 (Easy):** Open your `python-course` project folder in VS Code and select the `.venv` interpreter using the Command Palette.

<details>
<summary>Hint</summary>
`Cmd+Shift+P` / `Ctrl+Shift+P` → "Python: Select Interpreter" → choose the path containing `.venv`.
</details>

<details>
<summary>Solution</summary>
No code — verification step. Confirm the bottom-right status bar shows a Python version and the word `('.venv': venv)` or similar, indicating the project-local environment is active.
</details>

---

**Exercise 3 (Medium):** Create a file `setup_check.py` that prints your Python version and confirms a test package (e.g. `requests`, installed in Lesson 0.1) can be imported successfully. Run it from the integrated terminal.

<details>
<summary>Hint</summary>
Use the `sys` module for the version, and a plain `import requests` — if the import line doesn't error, it worked.
</details>

<details>
<summary>Solution</summary>

```python
# setup_check.py
import sys
import requests

print(f"Python version: {sys.version}")
print("requests imported successfully:", requests.__version__)
```

Run with:
```bash
python3 setup_check.py
```
Expected output (versions will vary):
```
Python version: 3.12.4 (...)
requests imported successfully: 2.32.3
```
</details>

---

**Exercise 4 (Medium):** Using the REPL (not a file), compute the average of `[24.99, 12.50, 39.00, 8.75]` interactively, without writing a script.

<details>
<summary>Hint</summary>
Open `python3` in the terminal, define the list, then divide `sum(...)` by `len(...)`.
</details>

<details>
<summary>Solution</summary>

```bash
python3
```
```python
>>> prices = [24.99, 12.50, 39.00, 8.75]
>>> sum(prices) / len(prices)
21.31
>>> exit()
```
</details>

---

**Exercise 5 (Hard):** Write a script `env_report.py` that prints: (1) the Python executable path currently in use (hint: `sys.executable`), (2) confirmation of whether that path is inside a `.venv` folder (a simple string check is fine), and (3) a list of all installed packages using the `importlib.metadata` module. Run it both from VS Code's Run button and from the terminal, and confirm the output is identical both ways.

<details>
<summary>Hint</summary>
`sys.executable` gives the interpreter path as a string — check with `".venv" in sys.executable`. For installed packages, use `importlib.metadata.distributions()` and pull each one's `.metadata["Name"]`.
</details>

<details>
<summary>Solution</summary>

```python
# env_report.py
import sys
from importlib.metadata import distributions

print("Python executable:", sys.executable)

in_venv = ".venv" in sys.executable
print("Running inside project .venv:", in_venv)

print("\nInstalled packages:")
for dist in distributions():
    print(f"  {dist.metadata['Name']} == {dist.version}")
```

Run both ways:
```bash
python3 env_report.py
```
and via the VS Code ▶️ Run button. If VS Code's selected interpreter matches the terminal's active environment (Lesson 0.2, Exercise 2), the output — including the executable path and `in_venv` result — should match exactly between the two run methods. If they don't match, it's a sign VS Code is pointing at a different Python than your terminal, worth fixing before Module 1.
</details>

---

## Module 0 Complete

Your environment is set up: Python 3.12+, an isolated virtual environment, and VS Code correctly linked to it. From here on, every lesson assumes you have a `.venv` active and a working way to run `.py` files.

**Previous lesson:** [0.1 — Installing Python & a Virtual Environment](lesson-0.1-installing-python.md)
**Next module:** [Module 1 — Python Fundamentals](../module-1-python-fundamentals/README.md)
