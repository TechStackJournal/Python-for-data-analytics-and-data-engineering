# Lesson 0.1 — Installing Python & a Virtual Environment

## CONCEPT

**Python** is the language interpreter you'll be running code through. A **virtual environment** is an isolated, self-contained copy of Python and its installed packages, scoped to a single project — so the packages one project needs (say, pandas version 2.x) never conflict with what another project needs (pandas version 1.x), and neither touches your computer's system-wide Python installation.

Every serious Python project — from a one-off script to a production pipeline — starts inside its own virtual environment.

## WHY IT MATTERS

Without a virtual environment, every package you install goes into one shared, global pool. Two projects with conflicting version requirements will break each other. Worse, many operating systems rely on their own system Python for internal tools — installing or upgrading packages globally can silently break unrelated system functionality. In data engineering specifically, reproducibility is critical: a pipeline that "works on my machine" but fails in production because of a mismatched package version is a real and common failure mode. Isolating environments per project is the first line of defense against that.

## INSTALLING PYTHON (STEP BY STEP)

First, check whether Python is already installed and current enough:

```bash
# Windows (PowerShell or Command Prompt)
python --version

# macOS / Linux
python3 --version
```

If you see `Python 3.12.x` or higher, skip ahead to **SYNTAX REFERENCE** below. If you see an older version, an error like `command not found`, or nothing at all, follow the steps for your operating system.

### Windows

1. Go to [python.org/downloads](https://www.python.org/downloads/) in your browser. The site auto-detects Windows and shows a "Download Python 3.x.x" button — click it.
2. Run the downloaded `.exe` installer.
3. **Important:** on the first installer screen, check the box at the bottom that says **"Add python.exe to PATH"** before clicking anything else. This is the single most common step beginners miss — skipping it means typing `python` in a terminal won't work anywhere outside the installer folder.
4. Click **"Install Now"** and wait for it to finish.
5. Close and reopen any terminal windows (PowerShell or Command Prompt) so they pick up the updated PATH.
6. Verify the install:
   ```powershell
   python --version
   ```
   You should see `Python 3.12.x` (or newer).
7. Verify `pip` (Python's package installer) came along with it:
   ```powershell
   pip --version
   ```

**If `python --version` still isn't recognized after reopening the terminal:** the PATH checkbox in step 3 was likely missed. Re-run the installer, choose "Modify," and ensure "Add python.exe to PATH" is checked, or add it manually via *Settings → System → About → Advanced system settings → Environment Variables*.

### macOS

macOS ships with an old system Python that you should **not** rely on for this course. The cleanest install method uses [Homebrew](https://brew.sh), a package manager for macOS.

1. Open the **Terminal** app (Spotlight search → "Terminal").
2. If you don't already have Homebrew installed, install it:
   ```bash
   /bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
   ```
   Follow any on-screen instructions it prints at the end (it sometimes asks you to run one or two extra commands to add Homebrew to your PATH — copy and run exactly what it shows).
3. Install Python with Homebrew:
   ```bash
   brew install python@3.12
   ```
4. Verify the install:
   ```bash
   python3 --version
   ```
   You should see `Python 3.12.x`.
5. Verify `pip`:
   ```bash
   pip3 --version
   ```

**Alternative (no Homebrew):** download the macOS installer directly from [python.org/downloads](https://www.python.org/downloads/), run the `.pkg` file, and follow the prompts. This works fine too — Homebrew is just easier to keep updated later.

### Linux (Debian/Ubuntu-based)

Most Linux distributions come with Python 3 pre-installed, but often an older version than this course needs.

1. Open a terminal.
2. Update your package list:
   ```bash
   sudo apt update
   ```
3. Install Python 3.12 (Ubuntu 22.04 and similar may need the `deadsnakes` PPA for the newest version):
   ```bash
   sudo apt install software-properties-common -y
   sudo add-apt-repository ppa:deadsnakes/ppa -y
   sudo apt update
   sudo apt install python3.12 python3.12-venv python3-pip -y
   ```
4. Verify the install:
   ```bash
   python3.12 --version
   ```
   You should see `Python 3.12.x`.

**Fedora/RHEL-based distributions:**
```bash
sudo dnf install python3.12 -y
```

**Note:** on Linux, once multiple Python versions exist side by side, you may need to explicitly type `python3.12` instead of `python3` to guarantee you're using the right one — especially when creating your virtual environment in the next section.

---

**Data Engineer Tip:** Notice that all three operating systems eventually converge on the exact same commands (`python3 --version`, `pip --version`, and soon `venv`). This is deliberate — Python is designed to behave identically across platforms once installed, which is a big part of why it's such a common choice for data pipelines that might run on a developer's laptop, a Linux server, and a cloud container over their lifetime.

## SYNTAX REFERENCE

**Check your Python version** (you need 3.12 or newer for this course):
```bash
python3 --version
```

**Create a project folder and a virtual environment inside it:**
```bash
mkdir python-course
cd python-course
python3 -m venv .venv
```

**Activate the virtual environment:**
```bash
# macOS / Linux
source .venv/bin/activate

# Windows (PowerShell)
.venv\Scripts\Activate.ps1

# Windows (Command Prompt)
.venv\Scripts\activate.bat
```

Once activated, your terminal prompt will show `(.venv)` at the start of the line — that's your confirmation it's active.

**Deactivate when you're done working:**
```bash
deactivate
```

**Install a package into the active environment:**
```bash
pip install pandas
```

**List installed packages:**
```bash
pip list
```

**Save your project's dependencies to a file (so others — or future you — can recreate the environment):**
```bash
pip freeze > requirements.txt
```

**Recreate an environment from that file later:**
```bash
pip install -r requirements.txt
```

## WORKED EXAMPLE

A full setup walkthrough, from nothing to a working isolated environment with one package installed:

```bash
# 1. Confirm Python is installed and modern enough
python3 --version
# Python 3.12.4   <- good, 3.12+

# 2. Create a dedicated folder for this course
mkdir python-de-course
cd python-de-course

# 3. Create the virtual environment (creates a .venv folder here)
python3 -m venv .venv

# 4. Activate it
source .venv/bin/activate
# prompt now shows: (.venv) yourname@computer python-de-course %

# 5. Confirm you're using the environment's Python, not the system one
which python3
# .../python-de-course/.venv/bin/python3   <- points inside .venv, correct

# 6. Install a package to test everything works
pip install requests

# 7. Freeze the current dependencies to a file
pip freeze > requirements.txt
cat requirements.txt
# requests==2.32.3
# ...(a few dependencies requests itself needs)

# 8. When finished for the day
deactivate
```

The next time you sit down to work on this project, you only need steps 4 and onward — `cd` into the folder and `source .venv/bin/activate` — the environment persists on disk between sessions.

## COMMON MISTAKES

**1. Installing packages without activating the environment first**
```bash
pip install pandas   # forgot to activate — installs globally, not into .venv!
```
Fix: always check your prompt shows `(.venv)` before running `pip install`. If unsure, run `which python3` (macOS/Linux) or `where python` (Windows) and confirm the path points inside your project's `.venv` folder.

**2. Committing `.venv` to Git**
The virtual environment folder is large, machine-specific, and shouldn't be tracked in version control. Fix: create a `.gitignore` file in your project root containing at least:
```
.venv/
__pycache__/
*.pyc
```
Share `requirements.txt` instead — that's what lets someone else (or future you, on a different machine) recreate the exact same environment.

**3. Having multiple Python versions installed and activating the wrong one**
If `python3 --version` shows an older version than expected, you may have multiple Python installations. Fix: be explicit about which interpreter creates the environment, e.g. `python3.12 -m venv .venv` if multiple versions are on your system.

**4. Forgetting to activate the environment in a new terminal window/tab**
Activation is per-terminal-session — opening a new terminal tab means you need to `activate` again, even in the same project folder.

**Data Engineer Tip:** In Module 8 you'll move from raw `venv`/`pip` to `poetry`, which manages both the virtual environment *and* a lockfile of exact dependency versions in one tool — a step up in reproducibility that most production data engineering teams use. Understanding the manual `venv`/`pip freeze` workflow first makes it much easier to understand *what problem* `poetry` is actually solving.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Run `python3 --version` (or `python --version` on Windows) in your terminal and confirm it reports 3.12 or higher. If it doesn't — or the command isn't found at all — follow the **INSTALLING PYTHON (STEP BY STEP)** section above for your operating system, then re-check.

<details>
<summary>Hint</summary>
If your system defaults to an older Python (common on macOS, which ships an old system Python, or on Linux distros with an older default package), you may need to install a fresh version and reference it explicitly, e.g. `python3.12`.
</details>

<details>
<summary>Solution</summary>
There's no code solution here — this is a verification step. The expected terminal output looks like:

```
Python 3.12.4
```

Any `3.12.x` or higher satisfies this course's requirements. If you had to install Python fresh, this confirms the install and PATH configuration both worked correctly.
</details>

---

**Exercise 2 (Easy):** Create a folder called `python-course`, and inside it, create a virtual environment named `.venv`. Activate it and confirm the `(.venv)` prefix appears in your terminal prompt.

<details>
<summary>Hint</summary>
`python3 -m venv .venv`, then the appropriate `activate` command for your OS.
</details>

<details>
<summary>Solution</summary>

```bash
mkdir python-course
cd python-course
python3 -m venv .venv
source .venv/bin/activate      # macOS/Linux
# or .venv\Scripts\Activate.ps1   on Windows PowerShell
```
Your prompt should now show `(.venv)` at the start.
</details>

---

**Exercise 3 (Medium):** With your virtual environment active, install the `requests` package, then run `pip list` and confirm it appears in the output along with its dependencies.

<details>
<summary>Hint</summary>
`pip install requests`, then `pip list`.
</details>

<details>
<summary>Solution</summary>

```bash
pip install requests
pip list
```
Expected output includes a line like:
```
requests           2.32.3
```
along with a few packages `requests` itself depends on (e.g. `certifi`, `charset-normalizer`, `idna`, `urllib3`).
</details>

---

**Exercise 4 (Medium):** Generate a `requirements.txt` file capturing your currently installed packages. Open it and confirm it lists `requests` with a pinned version number.

<details>
<summary>Hint</summary>
`pip freeze > requirements.txt`
</details>

<details>
<summary>Solution</summary>

```bash
pip freeze > requirements.txt
cat requirements.txt   # macOS/Linux
# or: type requirements.txt   on Windows
```
You should see exact pinned versions, e.g. `requests==2.32.3`.
</details>

---

**Exercise 5 (Hard):** Deactivate your environment, delete the `.venv` folder entirely (simulating a fresh clone of a project from GitHub), then recreate the environment from scratch and reinstall everything using only your `requirements.txt` file — without manually typing `pip install requests` again. Confirm `requests` is installed at the end with `pip list`.

<details>
<summary>Hint</summary>
`deactivate`, then delete `.venv` (`rm -rf .venv` on macOS/Linux, or delete the folder manually on Windows), then recreate it with `python3 -m venv .venv`, reactivate, and use `pip install -r requirements.txt`.
</details>

<details>
<summary>Solution</summary>

```bash
deactivate
rm -rf .venv                        # macOS/Linux
# or manually delete the .venv folder on Windows

python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt

pip list   # confirm requests is present again
```
This is exactly the workflow you (or a teammate) would use after cloning a project from GitHub — `requirements.txt` is what makes the environment reproducible without needing to remember every package you installed along the way.
</details>

---

**Next lesson:** [0.2 — VS Code Setup & Running Your First Script](lesson-0.2-vscode-and-first-script.md)
