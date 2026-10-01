# Lesson 7.5 — Managing Credentials with `python-dotenv`

## CONCEPT

**`python-dotenv`** loads configuration values — database passwords, API keys, connection strings — from a plain-text `.env` file into **environment variables**, which your Python code then reads at runtime. Critically, the `.env` file itself is excluded from version control (via `.gitignore`), so secrets never end up committed to a Git repository.

```python
from dotenv import load_dotenv
import os

load_dotenv()                                    # reads .env and loads its contents
password = os.getenv("DB_PASSWORD")                # read a value by name
```

This finally resolves the "hardcoded password" warning raised in every lesson so far this module.

## WHY IT MATTERS

A credential committed to Git — even briefly, even in a private repository — should be treated as permanently compromised, since it remains in the repository's history even after being removed from the current version of a file, and can be exposed if the repo is ever made public, forked, or breached. This is one of the most common real security incidents in software, entirely preventable with a five-minute setup. It's also directly relevant to your portfolio work: a public GitHub repo with a real database password baked into a `.py` file is a genuine, embarrassing, and avoidable mistake — this lesson is how you avoid it for good.

## SYNTAX REFERENCE

```python
# 1. Create a file named exactly ".env" in your project root (NOT .env.txt)
# Contents (plain text, KEY=value, no quotes needed for simple values):
#   DB_HOST=localhost
#   DB_PORT=5432
#   DB_NAME=ecommerce
#   DB_USER=postgres
#   DB_PASSWORD=your_actual_password

# 2. Create a .gitignore file (or add to an existing one) containing:
#   .env

# 3. In your Python code:
from dotenv import load_dotenv
import os

load_dotenv()   # looks for a .env file in the current directory by default

db_host = os.getenv("DB_HOST")
db_password = os.getenv("DB_PASSWORD")

# Providing a default value if a variable isn't set (useful for optional config)
db_port = os.getenv("DB_PORT", "5432")   # falls back to "5432" if DB_PORT isn't in .env

# Checking a required variable actually loaded, and failing clearly if not
db_password = os.getenv("DB_PASSWORD")
if db_password is None:
    raise ValueError("DB_PASSWORD not found — check your .env file exists and is loaded")

# Building a full connection string/engine from environment variables
from sqlalchemy import create_engine

connection_string = (
    f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}"
    f"@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
)
engine = create_engine(connection_string)

# Loading from a specific, non-default file path
load_dotenv(dotenv_path="config/production.env")

# A useful companion file: .env.example — a TEMPLATE with no real secrets,
# committed to Git so teammates know what variables they need to set themselves:
#   DB_HOST=localhost
#   DB_PORT=5432
#   DB_NAME=your_database_name
#   DB_USER=your_username
#   DB_PASSWORD=your_password_here
```

## WORKED EXAMPLE

Setting up a project properly, from nothing, following this lesson's pattern end to end:

```bash
# In your project folder, create the .env file (this file is NEVER committed)
cat > .env << 'EOF'
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce
DB_USER=postgres
DB_PASSWORD=my_real_password_123
EOF

# Create or update .gitignore to exclude it
echo ".env" >> .gitignore

# Create a template file that IS safe to commit, showing teammates the shape
cat > .env.example << 'EOF'
DB_HOST=localhost
DB_PORT=5432
DB_NAME=your_database_name
DB_USER=your_username
DB_PASSWORD=your_password_here
EOF
```

```python
# pipeline.py — this file IS safe to commit; it contains no real secrets
from dotenv import load_dotenv
from sqlalchemy import create_engine
import pandas as pd
import os

load_dotenv()   # reads the local .env file (not committed) into environment variables

# Fail fast and clearly if required configuration is missing
required_vars = ["DB_HOST", "DB_PORT", "DB_NAME", "DB_USER", "DB_PASSWORD"]
missing = [var for var in required_vars if os.getenv(var) is None]
if missing:
    raise ValueError(f"Missing required environment variables: {missing}")

connection_string = (
    f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}"
    f"@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
)
engine = create_engine(connection_string)

# From here on, the rest of the script looks exactly like Lessons 7.1-7.3 —
# but with zero real credentials visible anywhere in this file
orders = pd.read_sql("SELECT * FROM orders LIMIT 5;", engine)
print(orders)

print("Connected successfully — no hardcoded secrets in this file")
```

Anyone cloning this repository sees `pipeline.py` (safe) and `.env.example` (safe, shows the required shape) but never sees `.env` itself (excluded by `.gitignore`) — each person creates their own local `.env` with their own real credentials, which never leaves their machine.

## COMMON MISTAKES

**1. Forgetting to add `.env` to `.gitignore` — or adding it AFTER already committing the file once**
```bash
git add .env    # oops — now it's in Git history, even if you delete it in a later commit
```
Fix: add `.env` to `.gitignore` **before** ever running `git add .` in a project, ideally as one of the very first things you do when setting up a new repository. If a secret was already committed, the safest fix is to treat that credential as compromised and rotate it (change the actual password) — simply deleting the file in a later commit does NOT remove it from the repository's history.

**2. Naming the file `.env.txt` instead of `.env`**
Many operating systems hide file extensions by default, so a file created as "`.env`" through a text editor's "Save As" dialog can silently become "`.env.txt`" — and `load_dotenv()` won't find it, since it looks for the exact name `.env` by default. Fix: verify the exact filename from a terminal (`ls -la` on macOS/Linux, `dir /a` on Windows) rather than trusting a graphical file browser, which may hide the true extension.

**3. Calling `os.getenv()` before `load_dotenv()`**
```python
password = os.getenv("DB_PASSWORD")   # None — .env hasn't been loaded into the environment yet!
load_dotenv()
```
Fix: `load_dotenv()` must run first, before any `os.getenv()` calls that depend on values from the `.env` file — put it at the very top of your script.

**4. Assuming `.env` values are automatically typed correctly**
```python
port = os.getenv("DB_PORT")   # "5432" — a STRING, not an int!
```
Every value from `os.getenv()` is a plain string, exactly like every value read from a CSV (Module 4) or the command line — cast explicitly (`int(os.getenv("DB_PORT"))`) if you need a specific type.

**5. Committing a REAL `.env.example` instead of a template with placeholder values**
```
# .env.example — WRONG, this has a real password!
DB_PASSWORD=my_real_password_123
```
Fix: `.env.example` should contain only placeholder text (`your_password_here`) showing the *shape* of what's needed — it exists specifically to be safely committed, so it must never contain actual working credentials.

**Data Engineer Tip:** This `.env` pattern is the local-development equivalent of what a proper cloud secrets manager (Azure Key Vault, in your specific roadmap) does in production — both exist to keep credentials out of source code and configuration files, injected into the running application only at the moment it actually needs them. Getting comfortable with `.env`/`python-dotenv` now, on your own machine, builds the exact mental model you'll extend to Key Vault later: secrets live somewhere separate from your code, are fetched at runtime, and are never hardcoded or committed — same principle, different mechanism depending on the environment.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create a `.env` file with a single variable `GREETING=Hello from dotenv`, add `.env` to a `.gitignore` file, then write a Python script that loads it and prints the value.

<details>
<summary>Hint</summary>
`load_dotenv()`, then `os.getenv("GREETING")`.
</details>

<details>
<summary>Solution</summary>

```bash
echo "GREETING=Hello from dotenv" > .env
echo ".env" >> .gitignore
```
```python
from dotenv import load_dotenv
import os

load_dotenv()
message = os.getenv("GREETING")
print(message)   # "Hello from dotenv"
```
</details>

---

**Exercise 2 (Easy):** Add a second variable `MAX_RETRIES=3` to your `.env` file, load it, convert it to an integer, and print its type to confirm it needed the explicit conversion.

<details>
<summary>Hint</summary>
`os.getenv()` always returns a string — use `int()` to convert.
</details>

<details>
<summary>Solution</summary>

```bash
echo "MAX_RETRIES=3" >> .env
```
```python
from dotenv import load_dotenv
import os

load_dotenv()
raw_value = os.getenv("MAX_RETRIES")
print(type(raw_value), raw_value)       # <class 'str'> 3

max_retries = int(raw_value)
print(type(max_retries), max_retries)    # <class 'int'> 3
```
</details>

---

**Exercise 3 (Medium):** Write a script that attempts to read a variable `API_KEY` that does NOT exist in your `.env` file, using `os.getenv()` with a default value of `"not_set"`. Print the result to confirm the default was used.

<details>
<summary>Hint</summary>
`os.getenv("API_KEY", "not_set")`
</details>

<details>
<summary>Solution</summary>

```python
from dotenv import load_dotenv
import os

load_dotenv()
api_key = os.getenv("API_KEY", "not_set")
print(api_key)   # "not_set" — since API_KEY isn't actually in the .env file
```
</details>

---

**Exercise 4 (Medium):** Write a function `require_env(var_name)` that reads an environment variable and raises a clear `ValueError` (with a message naming the missing variable) if it's not set, rather than silently returning `None`. Test it with a variable that exists and one that doesn't.

<details>
<summary>Hint</summary>
Check `if value is None:` and raise `ValueError(f"...")` inside the function.
</details>

<details>
<summary>Solution</summary>

```python
from dotenv import load_dotenv
import os

load_dotenv()

def require_env(var_name):
    value = os.getenv(var_name)
    if value is None:
        raise ValueError(f"Required environment variable '{var_name}' is not set")
    return value

# Assuming GREETING exists from Exercise 1
print(require_env("GREETING"))   # works fine

try:
    require_env("DEFINITELY_NOT_SET")
except ValueError as e:
    print(f"Caught expected error: {e}")
```
</details>

---

**Exercise 5 (Hard):** Set up a full `.env` with `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` (use placeholder/fake values if you don't have a real database handy). Write a function `build_connection_string()` that: (1) loads the `.env` file, (2) checks that ALL five required variables are present, raising a `ValueError` listing every missing one (not just the first) if any are absent, and (3) if all are present, returns a properly formatted PostgreSQL connection string built from them. Test it once with all variables present, and once by temporarily renaming/commenting out one variable in your `.env` to confirm the error message correctly identifies it.

<details>
<summary>Hint</summary>
Build a list of required variable names, use a list comprehension to find which are `None`, and only raise if that list is non-empty — this collects every missing one rather than stopping at the first.
</details>

<details>
<summary>Solution</summary>

```python
from dotenv import load_dotenv
import os

def build_connection_string():
    load_dotenv()

    required_vars = ["DB_HOST", "DB_PORT", "DB_NAME", "DB_USER", "DB_PASSWORD"]
    missing = [var for var in required_vars if os.getenv(var) is None]

    if missing:
        raise ValueError(f"Missing required environment variables: {missing}")

    return (
        f"postgresql://{os.getenv('DB_USER')}:{os.getenv('DB_PASSWORD')}"
        f"@{os.getenv('DB_HOST')}:{os.getenv('DB_PORT')}/{os.getenv('DB_NAME')}"
    )

# With all variables present
conn_str = build_connection_string()
print("Connection string built successfully (password hidden in this print):",
      conn_str.split("@")[1])   # print only the non-secret part, as good practice

# To test the failure case, comment out DB_PASSWORD in your .env file, then rerun:
# try:
#     build_connection_string()
# except ValueError as e:
#     print(f"Caught expected error: {e}")
#     # Missing required environment variables: ['DB_PASSWORD']
```
</details>

---

**Previous lesson:** [7.4 — Parameterized Queries & SQL Injection](lesson-7.4-parameterized-queries.md)
**Next lesson:** [7.6 — Calling REST APIs with requests](lesson-7.6-calling-rest-apis.md)
