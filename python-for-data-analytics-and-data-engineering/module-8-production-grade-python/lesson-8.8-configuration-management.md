# Lesson 8.8 — Configuration Management

## CONCEPT

**Configuration** is everything about how a pipeline runs that might change between environments or over time, without the underlying code itself changing — a file path, a batch size, a table name, which environment (dev/staging/production) it's running in. Rather than hardcoding these values directly into your functions, **configuration management** centralizes them in one place — typically a YAML file, environment variables (Lesson 7.5's `.env` pattern), or both together — that the code reads from at runtime.

```yaml
# config.yaml
database:
  host: localhost
  port: 5432
  name: ecommerce
batch_size: 500
output_dir: data/processed
```

```python
import yaml

with open("config.yaml") as f:
    config = yaml.safe_load(f)

print(config["batch_size"])   # 500
```

## WHY IT MATTERS

Hardcoded values scattered throughout a codebase are one of the most common reasons a pipeline that "worked in development" breaks when deployed elsewhere — a hardcoded file path (`/Users/yourname/data/orders.csv`) simply doesn't exist on a production server, and a hardcoded batch size means changing it requires editing and redeploying code rather than just updating a config value. Centralizing configuration means the exact same code can run correctly in development, staging, and production, differing only in which config values are loaded — genuinely essential once a pipeline needs to run somewhere other than your own laptop.

## SYNTAX REFERENCE

```python
import yaml
import os
from pathlib import Path
from dotenv import load_dotenv

# Reading a YAML config file
with open("config.yaml") as f:
    config = yaml.safe_load(f)

config["database"]["host"]      # nested access, like any dict
config.get("batch_size", 100)     # with a sensible default if the key is missing

# Combining YAML (non-secret config) with .env (secrets) — the standard split
load_dotenv()   # loads environment variables from .env (Lesson 7.5)

db_password = os.getenv("DB_PASSWORD")       # secret — from .env, NEVER committed to Git
db_host = config["database"]["host"]            # non-secret — from config.yaml, SAFE to commit

# A common pattern: a dedicated config module that loads everything once
# config.py
class Settings:
    def __init__(self):
        load_dotenv()
        with open("config.yaml") as f:
            yaml_config = yaml.safe_load(f)

        self.db_host = yaml_config["database"]["host"]
        self.db_port = yaml_config["database"]["port"]
        self.db_password = os.getenv("DB_PASSWORD")
        self.batch_size = yaml_config.get("batch_size", 100)

settings = Settings()      # created ONCE, imported wherever needed

# NEVER use a hardcoded absolute path — build paths relative to the project,
# using pathlib (Module 2, Lesson 2.7), so the code works on any machine
PROJECT_ROOT = Path(__file__).parent          # the folder THIS file lives in
DATA_DIR = PROJECT_ROOT / "data"
OUTPUT_DIR = DATA_DIR / "processed"

# Supporting multiple environments (dev/staging/prod) via a single env variable
ENVIRONMENT = os.getenv("ENVIRONMENT", "development")

if ENVIRONMENT == "production":
    config_file = "config.prod.yaml"
else:
    config_file = "config.dev.yaml"
```

## WORKED EXAMPLE

Building a small, reusable configuration setup for a pipeline, combining YAML config with `.env` secrets — replacing hardcoded values scattered throughout earlier lessons' worked examples:

```yaml
# config.yaml — safe to commit to Git, contains no secrets
database:
  host: localhost
  port: 5432
  name: ecommerce

pipeline:
  batch_size: 500
  output_dir: data/processed
  log_file: logs/pipeline.log
```

```bash
# .env — NEVER committed to Git (Lesson 7.5), contains only secrets
DB_USER=pipeline_user
DB_PASSWORD=a_real_secret_password_here
```

```python
# config.py
import os
import yaml
from pathlib import Path
from dotenv import load_dotenv

PROJECT_ROOT = Path(__file__).parent


class Settings:
    def __init__(self, config_path: str = "config.yaml"):
        load_dotenv()

        config_file = PROJECT_ROOT / config_path
        with open(config_file) as f:
            raw_config = yaml.safe_load(f)

        # Non-secret settings — from YAML
        self.db_host = raw_config["database"]["host"]
        self.db_port = raw_config["database"]["port"]
        self.db_name = raw_config["database"]["name"]
        self.batch_size = raw_config["pipeline"]["batch_size"]
        self.output_dir = PROJECT_ROOT / raw_config["pipeline"]["output_dir"]
        self.log_file = PROJECT_ROOT / raw_config["pipeline"]["log_file"]

        # Secret settings — from .env, NEVER from the committed YAML file
        self.db_user = os.getenv("DB_USER")
        self.db_password = os.getenv("DB_PASSWORD")

        if not self.db_password:
            raise ValueError("DB_PASSWORD not set — check your .env file")

    def database_url(self) -> str:
        return f"postgresql://{self.db_user}:{self.db_password}@{self.db_host}:{self.db_port}/{self.db_name}"


# --- Using it elsewhere in the pipeline ---
settings = Settings()

print(f"Connecting to {settings.db_host}:{settings.db_port}/{settings.db_name}")
print(f"Batch size: {settings.batch_size}")
print(f"Output directory: {settings.output_dir}")
# Notice: NOT a single hardcoded value anywhere in this usage code — changing
# the batch size, output location, or even switching to an entirely different
# database host requires editing ONLY config.yaml/.env, never this Python code

# Ensure the output directory actually exists before the pipeline tries to write to it
settings.output_dir.mkdir(parents=True, exist_ok=True)
```

## COMMON MISTAKES

**1. Hardcoding a value "just this once" instead of adding it to config**
```python
BATCH_SIZE = 500   # hardcoded directly in a random function, buried deep in the codebase
```
Fix: the moment you write a literal value that conceivably differs between environments or might need tuning later, it belongs in config — a batch size, a timeout, a file path, a table name are all classic candidates; genuinely fixed values (like a mathematical constant) don't need this treatment.

**2. Putting secrets in the YAML config file instead of `.env`**
```yaml
# config.yaml — accidentally committed to Git with the password included!
database:
  password: a_real_secret_password_here
```
Fix: the split matters — YAML config is meant to be committed to version control (it's genuinely useful for a teammate or your future self to see what settings exist), while `.env` (Lesson 7.5) is specifically for secrets that must never be committed; mixing them up risks leaking a real credential into Git history.

**3. Using absolute, machine-specific paths**
```python
DATA_DIR = "/Users/yourname/projects/ecommerce_pipeline/data"   # ONLY works on your exact machine!
```
Fix: build paths relative to the project itself using `pathlib` and `Path(__file__).parent` (as in the worked example) — this makes the exact same code work correctly regardless of which machine or which absolute folder the project happens to be cloned into.

**4. Loading configuration repeatedly throughout a codebase, instead of once**
```python
def clean_orders():
    with open("config.yaml") as f:      # re-reading and re-parsing the file
        config = yaml.safe_load(f)        # EVERY time this function runs!
    ...
```
Fix: load configuration once, typically into a single `Settings` object (as in the worked example) created near your `main()` entry point, then pass it (or import it) wherever it's needed — repeatedly re-reading the same file on every function call is wasteful and makes it harder to reason about configuration as a single, consistent source of truth for a given run.

**5. Not validating that required configuration actually exists before using it**
```python
db_password = os.getenv("DB_PASSWORD")    # if .env is missing this, db_password is silently None!
connect(password=db_password)                # fails much later, with a confusing connection error
```
Fix: validate required settings immediately after loading them (as the worked example's `Settings.__init__` does, raising a clear `ValueError` if `DB_PASSWORD` is missing) — failing fast, with a clear message, at startup is far more useful than a confusing failure much later, deep inside an unrelated function.

**Data Engineer Tip:** The YAML-plus-`.env` split shown here is exactly the pattern that supports running the identical pipeline code across development, staging, and production environments (a specific, direct application of your Azure roadmap) — each environment gets its own `config.{env}.yaml` and its own `.env` (or, in a real cloud deployment, secrets pulled from Azure Key Vault instead of a local `.env` file), while the Python code itself never changes at all between environments. This separation of "code" from "configuration" is a foundational software engineering principle, not just a data engineering one, but it's especially visible and important in pipeline work specifically because the same transformation logic so often needs to run against genuinely different data sources and destinations.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Create a `config.yaml` file with a top-level key `app_name: "My Pipeline"` and a nested `settings: {debug: true, max_retries: 3}`. Write a script that loads it with `yaml.safe_load()` and prints all three values.

<details>
<summary>Hint</summary>
`yaml.safe_load(f)` returns a plain nested dict — access it exactly like the nested dictionaries from Module 2.
</details>

<details>
<summary>Solution</summary>

```yaml
# config.yaml
app_name: "My Pipeline"
settings:
  debug: true
  max_retries: 3
```
```python
import yaml

with open("config.yaml") as f:
    config = yaml.safe_load(f)

print(config["app_name"])
print(config["settings"]["debug"])
print(config["settings"]["max_retries"])
```
</details>

---

**Exercise 2 (Easy):** Using `pathlib` (Module 2, Lesson 2.7) and `Path(__file__).parent`, build a path to a `data/` subfolder relative to your script's own location, and print it — confirming it works regardless of which directory you run the script FROM.

<details>
<summary>Hint</summary>
`Path(__file__).parent / "data"`
</details>

<details>
<summary>Solution</summary>

```python
from pathlib import Path

PROJECT_ROOT = Path(__file__).parent
data_dir = PROJECT_ROOT / "data"
print(data_dir)
```
Try running this script from a different working directory (`cd ..` then `python3 subfolder/script.py`) and confirm the printed path still correctly points to `data/` relative to the SCRIPT's location, not wherever you happened to run the command from.
</details>

---

**Exercise 3 (Medium):** Build a `config.yaml` with `pipeline: {batch_size: 200, output_dir: "output"}`, and a `.env` file with `API_KEY=a_fake_test_key_12345`. Write a `Settings` class (similar to the worked example, simplified) that loads both, exposing `.batch_size`, `.output_dir` (as a `Path`, relative to the script), and `.api_key`. Print all three.

<details>
<summary>Hint</summary>
Combine `yaml.safe_load()` for the YAML file and `load_dotenv()` + `os.getenv()` for the `.env` file, inside one `__init__`.
</details>

<details>
<summary>Solution</summary>

```yaml
# config.yaml
pipeline:
  batch_size: 200
  output_dir: "output"
```
```bash
# .env
API_KEY=a_fake_test_key_12345
```
```python
# settings.py
import os
import yaml
from pathlib import Path
from dotenv import load_dotenv

PROJECT_ROOT = Path(__file__).parent

class Settings:
    def __init__(self):
        load_dotenv()
        with open(PROJECT_ROOT / "config.yaml") as f:
            config = yaml.safe_load(f)

        self.batch_size = config["pipeline"]["batch_size"]
        self.output_dir = PROJECT_ROOT / config["pipeline"]["output_dir"]
        self.api_key = os.getenv("API_KEY")

settings = Settings()
print(settings.batch_size)
print(settings.output_dir)
print(settings.api_key)
```
</details>

---

**Exercise 4 (Medium):** Extend the `Settings` class from Exercise 3 to raise a clear `ValueError` if `API_KEY` is missing from the environment (simulate this by temporarily removing/renaming the `.env` file and confirming the error message is genuinely informative).

<details>
<summary>Hint</summary>
`if not self.api_key: raise ValueError("API_KEY not set — check your .env file")`
</details>

<details>
<summary>Solution</summary>

```python
# settings.py
import os
import yaml
from pathlib import Path
from dotenv import load_dotenv

PROJECT_ROOT = Path(__file__).parent

class Settings:
    def __init__(self):
        load_dotenv()
        with open(PROJECT_ROOT / "config.yaml") as f:
            config = yaml.safe_load(f)

        self.batch_size = config["pipeline"]["batch_size"]
        self.output_dir = PROJECT_ROOT / config["pipeline"]["output_dir"]
        self.api_key = os.getenv("API_KEY")

        if not self.api_key:
            raise ValueError("API_KEY not set — check your .env file")

settings = Settings()
```
Rename `.env` to `.env.bak` and rerun — you should see:
```
ValueError: API_KEY not set — check your .env file
```
instead of a confusing failure later when `settings.api_key` gets used somewhere deep in the code with a silent `None` value.
</details>

---

**Exercise 5 (Hard):** Build a configuration setup supporting TWO environments: create `config.dev.yaml` (with `output_dir: "output/dev"`) and `config.prod.yaml` (with `output_dir: "output/prod"`), and a `Settings` class whose `__init__` reads an `ENVIRONMENT` variable (from `.env`, defaulting to `"development"` if unset) and loads the correspondingly-named config file (`config.dev.yaml` for `"development"`, `config.prod.yaml` for `"production"`). Test it by setting `ENVIRONMENT=production` in `.env` and confirming the correct `output_dir` loads, then removing that line (or setting it to blank) and confirming it falls back to the development config.

<details>
<summary>Hint</summary>
`environment = os.getenv("ENVIRONMENT", "development")`, then build the config filename conditionally: `f"config.{'prod' if environment == 'production' else 'dev'}.yaml"`.
</details>

<details>
<summary>Solution</summary>

```yaml
# config.dev.yaml
pipeline:
  output_dir: "output/dev"
```
```yaml
# config.prod.yaml
pipeline:
  output_dir: "output/prod"
```
```python
# settings.py
import os
import yaml
from pathlib import Path
from dotenv import load_dotenv

PROJECT_ROOT = Path(__file__).parent

class Settings:
    def __init__(self):
        load_dotenv()
        environment = os.getenv("ENVIRONMENT", "development")

        config_filename = "config.prod.yaml" if environment == "production" else "config.dev.yaml"
        with open(PROJECT_ROOT / config_filename) as f:
            config = yaml.safe_load(f)

        self.environment = environment
        self.output_dir = PROJECT_ROOT / config["pipeline"]["output_dir"]

settings = Settings()
print(f"Environment: {settings.environment}")
print(f"Output dir: {settings.output_dir}")
```
With `.env` containing `ENVIRONMENT=production`:
```
Environment: production
Output dir: .../output/prod
```
With `.env` missing that line entirely (or `ENVIRONMENT` unset):
```
Environment: development
Output dir: .../output/dev
```
This is exactly the mechanism that lets identical code deploy safely to different environments — only the environment variable (set differently per deployment) changes which config file, and therefore which settings, actually get used.
</details>

---

**Previous lesson:** [8.7 — Type Hints & Static Checking with mypy](lesson-8.7-type-hints-and-mypy.md)
**Next lesson:** [8.9 — Code Style, Linting & Pre-commit Hooks](lesson-8.9-linting-and-pre-commit.md)
