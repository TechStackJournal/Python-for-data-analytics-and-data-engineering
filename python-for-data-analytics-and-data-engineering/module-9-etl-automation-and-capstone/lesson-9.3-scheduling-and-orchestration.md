# Lesson 9.3 — Scheduling & an Intro to Orchestration

## CONCEPT

**Scheduling** is running a pipeline automatically at a defined interval, with no human triggering it manually. **`cron`** is the standard Unix/Linux scheduler — a simple, time-based tool for running a command on a recurring schedule. **Orchestration** is a broader concept for more complex pipelines: managing multiple interdependent tasks (a DAG — Directed Acyclic Graph — of steps with dependencies), handling retries, tracking run history, and alerting on failure. **Apache Airflow** is the most widely used open-source orchestration tool in the Python data ecosystem; **Azure Data Factory (ADF)** is the equivalent managed service in the Azure ecosystem this course's roadmap is building toward.

```cron
# crontab syntax: minute hour day month weekday command
0 6 * * * /path/to/venv/bin/python /path/to/pipeline.py
# runs the pipeline every day at 6:00 AM
```

## WHY IT MATTERS

A pipeline that only runs when you remember to run it manually isn't really a pipeline — it's a script. Understanding `cron` gives you the simplest possible path to genuine automation, and it's still widely used for straightforward, single-task scheduling in real production environments. Understanding orchestration concepts — even without deploying a full Airflow instance for this course — prepares you for nearly every data engineering job posting, since DAG-based thinking (task dependencies, retries, backfills) is the dominant mental model across Airflow, ADF, Dagster, and Prefect alike.

## SYNTAX REFERENCE

```bash
# --- cron basics ---
crontab -e                 # open your user's crontab for editing
crontab -l                  # list current scheduled jobs

# crontab line format: minute hour day-of-month month day-of-week command
# *  *  *  *  *
# |  |  |  |  |
# |  |  |  |  +-- day of week (0-6, Sunday=0)
# |  |  |  +----- month (1-12)
# |  |  +-------- day of month (1-31)
# |  +----------- hour (0-23)
# +-------------- minute (0-59)

0 6 * * *        /path/to/venv/bin/python /path/to/pipeline.py        # daily at 6:00 AM
*/15 * * * *     /path/to/venv/bin/python /path/to/pipeline.py         # every 15 minutes
0 0 * * 0        /path/to/venv/bin/python /path/to/weekly_report.py     # weekly, Sunday midnight
0 9 1 * *        /path/to/venv/bin/python /path/to/monthly_pipeline.py   # 1st of every month, 9 AM

# Redirecting output to a log file (cron jobs run with no visible terminal)
0 6 * * * /path/to/venv/bin/python /path/to/pipeline.py >> /var/log/pipeline.log 2>&1
```

```python
# --- A minimal, conceptual DAG, expressed in plain Python ---
# (This is the MENTAL MODEL behind Airflow/ADF — not real Airflow syntax)

tasks = {
    "extract_orders": {"depends_on": []},
    "extract_customers": {"depends_on": []},
    "transform": {"depends_on": ["extract_orders", "extract_customers"]},
    "data_quality_check": {"depends_on": ["transform"]},
    "load": {"depends_on": ["data_quality_check"]},
    "send_success_notification": {"depends_on": ["load"]},
}
# Reading this as a graph: extract_orders and extract_customers can run in
# PARALLEL (neither depends on the other); transform must wait for BOTH to
# finish; everything downstream runs only after its dependencies succeed.
# This dependency structure — not the specific syntax — is what "DAG" means.
```

```python
# --- Real (simplified) Airflow DAG syntax, for recognition — not required to run this course ---
from airflow import DAG
from airflow.operators.python import PythonOperator
from datetime import datetime

with DAG("ecommerce_etl", schedule="0 6 * * *", start_date=datetime(2024, 1, 1)) as dag:
    extract_task = PythonOperator(task_id="extract", python_callable=extract)
    transform_task = PythonOperator(task_id="transform", python_callable=transform)
    load_task = PythonOperator(task_id="load", python_callable=load)

    extract_task >> transform_task >> load_task   # the >> operator defines the dependency order
```

## WORKED EXAMPLE

Preparing a pipeline script to be scheduled with cron, including proper logging to a file (since cron jobs have no visible terminal) and a clear exit code so failures are detectable:

```python
# run_pipeline.py — designed to be run unattended by cron
import sys
import logging
from pathlib import Path
from datetime import datetime

# Configure logging to a FILE, not just the console — cron output isn't watched live
log_dir = Path("logs")
log_dir.mkdir(exist_ok=True)
log_file = log_dir / f"pipeline_{datetime.now().strftime('%Y%m%d')}.log"

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
    handlers=[
        logging.FileHandler(log_file),
        logging.StreamHandler(),   # ALSO prints to console, useful when testing manually
    ],
)
logger = logging.getLogger(__name__)


def extract():
    logger.info("Starting extraction")
    # ... real extraction logic ...
    return {"rows": 150}


def transform(data):
    logger.info("Starting transform")
    # ... real transform logic ...
    return data


def load(data):
    logger.info("Starting load")
    # ... real load logic ...
    return True


def main() -> int:
    """Returns an exit code: 0 for success, 1 for failure — this is what
    makes a cron job's failure actually DETECTABLE by monitoring tools."""
    try:
        raw = extract()
        clean = transform(raw)
        success = load(clean)
        if not success:
            logger.error("Load reported failure")
            return 1
        logger.info("Pipeline completed successfully")
        return 0
    except Exception:
        logger.exception("Pipeline failed with an unhandled exception")
        return 1


if __name__ == "__main__":
    exit_code = main()
    sys.exit(exit_code)
```

The corresponding crontab entry:
```bash
# Run daily at 6:00 AM, append both stdout and stderr to a log,
# and rely on the script's own exit code for success/failure detection
0 6 * * * /home/user/ecommerce_pipeline/.venv/bin/python /home/user/ecommerce_pipeline/run_pipeline.py >> /home/user/ecommerce_pipeline/logs/cron.log 2>&1
```

Note the **absolute path** to the virtual environment's Python interpreter — cron runs with a minimal environment that does NOT automatically know about your activated venv, so you must point directly at `.venv/bin/python`, not just `python3`.

## COMMON MISTAKES

**1. Using a relative path, or assuming `python3` resolves correctly inside cron**
```bash
0 6 * * * python3 pipeline.py   # FAILS — cron's PATH is minimal, pipeline.py's location is ambiguous
```
Fix: always use absolute paths for both the Python interpreter (specifically your project's `.venv/bin/python`, not the system `python3`) and the script itself — this is the single most common reason a cron job "works when I test it manually but fails when cron runs it."

**2. Not redirecting output anywhere, then having no way to debug a silent failure**
```bash
0 6 * * * /path/.venv/bin/python /path/pipeline.py   # no logging, no output redirect
```
If this fails, there's no record of why — cron by default emails output only if your system's mail setup is configured (rare on modern systems), and otherwise the output simply vanishes. Fix: always redirect to a log file (`>> logfile.log 2>&1`) AND have the script itself log to a file via Python's `logging` module, as in the worked example — belt and suspenders.

**3. Forgetting the script needs to exit with a non-zero code to signal failure**
```python
def main():
    try:
        run_pipeline()
    except Exception as e:
        print(f"Failed: {e}")
        # no sys.exit(1) here — the process exits with code 0 (success) anyway!
```
Fix: any monitoring tool watching for job failures typically checks the process's exit code — `sys.exit(1)` (or returning a non-zero int from `main()` and passing it to `sys.exit()`, as in the worked example) is what makes a failure actually *detectable* externally, not just loggable.

**4. Scheduling a job more frequently than it can reliably complete**
```bash
*/5 * * * * /path/.venv/bin/python /path/slow_pipeline.py   # runs every 5 min, but takes 8 minutes!
```
This causes overlapping runs — a new instance starts before the previous one finishes, which can cause race conditions, duplicate processing, or resource contention. Fix: ensure the schedule interval is comfortably longer than the pipeline's typical run time, or add explicit overlap protection (a lock file, or rely on orchestration tooling's built-in concurrency controls).

**5. Treating cron as sufficient for anything with real task dependencies**
`cron` has no concept of "task B should only run if task A succeeded" — it just runs a command at a time, with zero awareness of other jobs. Fix: for anything beyond a single linear script (multiple interdependent extract/transform/load steps, cross-pipeline dependencies, a need for retry policies or a visual run history), that's exactly the gap orchestration tools like Airflow or Azure Data Factory are built to fill — recognizing when you've outgrown cron is itself a useful skill.

**Data Engineer Tip:** Your Azure roadmap's next phase (dbt + Synapse) will very likely use Azure Data Factory or a similar managed orchestrator rather than raw cron for anything beyond local development and testing — but the underlying DAG concept (tasks, dependencies, retries, a schedule) you've learned here transfers directly. When you eventually look at an ADF pipeline or an Airflow DAG for the first time, the `tasks = {...}` dependency dictionary in this lesson's Syntax Reference is conceptually exactly what you're looking at, just with a visual designer or a different syntax on top.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Write the crontab line that would run a script located at `/home/data/pipeline.py`, using the Python interpreter at `/home/data/.venv/bin/python`, every day at 2:30 AM.

<details>
<summary>Hint</summary>
Minute 30, hour 2, then `* * *` for the rest.
</details>

<details>
<summary>Solution</summary>

```
30 2 * * * /home/data/.venv/bin/python /home/data/pipeline.py
```
</details>

---

**Exercise 2 (Easy):** Write the crontab line that runs a script every Monday at 9:00 AM only (not every day).

<details>
<summary>Hint</summary>
The day-of-week field — Monday is `1` (Sunday is `0`).
</details>

<details>
<summary>Solution</summary>

```
0 9 * * 1 /home/data/.venv/bin/python /home/data/weekly_report.py
```
</details>

---

**Exercise 3 (Medium):** Write a `main()` function for a simple script that calls a (possibly failing) `do_work()` function, catches any exception, logs it with `logger.exception()`, and returns the correct exit code in each case (0 for success, 1 for failure). Test it twice: once where `do_work()` succeeds, once where it raises an exception, confirming the correct exit code each time (print it rather than actually calling `sys.exit()`, so you can observe the result inline).

<details>
<summary>Hint</summary>
Wrap the call in `try/except`, return `0` on success and `1` on any caught exception.
</details>

<details>
<summary>Solution</summary>

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def do_work(should_fail: bool):
    if should_fail:
        raise ValueError("simulated failure")
    return "done"

def main(should_fail: bool) -> int:
    try:
        do_work(should_fail)
        logger.info("Success")
        return 0
    except Exception:
        logger.exception("Pipeline failed")
        return 1

print("Exit code (success case):", main(False))   # 0
print("Exit code (failure case):", main(True))      # 1
```
</details>

---

**Exercise 4 (Medium):** Given this set of pipeline tasks and their dependencies:
```python
tasks = {
    "extract_a": {"depends_on": []},
    "extract_b": {"depends_on": []},
    "merge": {"depends_on": ["extract_a", "extract_b"]},
    "validate": {"depends_on": ["merge"]},
    "load": {"depends_on": ["validate"]},
}
```
write a function `can_run(task_name, completed_tasks, tasks)` that returns `True` if every dependency of `task_name` is already in `completed_tasks` (a set), `False` otherwise. Test it: confirm `"merge"` cannot run before both extracts are complete, but can once they are.

<details>
<summary>Hint</summary>
Check `all(dep in completed_tasks for dep in tasks[task_name]["depends_on"])`.
</details>

<details>
<summary>Solution</summary>

```python
tasks = {
    "extract_a": {"depends_on": []},
    "extract_b": {"depends_on": []},
    "merge": {"depends_on": ["extract_a", "extract_b"]},
    "validate": {"depends_on": ["merge"]},
    "load": {"depends_on": ["validate"]},
}

def can_run(task_name, completed_tasks, tasks):
    deps = tasks[task_name]["depends_on"]
    return all(dep in completed_tasks for dep in deps)

completed = {"extract_a"}
print(can_run("merge", completed, tasks))   # False — extract_b not done yet

completed = {"extract_a", "extract_b"}
print(can_run("merge", completed, tasks))   # True — both dependencies satisfied
```
</details>

---

**Exercise 5 (Hard):** Using the `tasks` dependency dictionary from Exercise 4, write a simple scheduler simulation: a function `run_all(tasks)` that repeatedly scans for any task whose dependencies are all in a `completed` set, "runs" it (just print its name and add it to `completed`), and keeps looping until every task has run. Handle the case of tasks with no dependencies (they can run immediately) and confirm the final printed order respects every dependency (e.g., `"merge"` never prints before both `"extract_a"` and `"extract_b"`).

<details>
<summary>Hint</summary>
Loop until `len(completed) == len(tasks)`; each pass through the loop, check every not-yet-completed task with `can_run()` from Exercise 4, and run (mark complete) any that are ready. Guard against an infinite loop in case of a cycle (not expected here, but good practice) with a maximum iteration count.
</details>

<details>
<summary>Solution</summary>

```python
tasks = {
    "extract_a": {"depends_on": []},
    "extract_b": {"depends_on": []},
    "merge": {"depends_on": ["extract_a", "extract_b"]},
    "validate": {"depends_on": ["merge"]},
    "load": {"depends_on": ["validate"]},
}

def can_run(task_name, completed_tasks, tasks):
    deps = tasks[task_name]["depends_on"]
    return all(dep in completed_tasks for dep in deps)

def run_all(tasks):
    completed = set()
    max_iterations = len(tasks) * 2   # safety guard against an unexpected cycle
    iterations = 0

    while len(completed) < len(tasks) and iterations < max_iterations:
        ran_something = False
        for task_name in tasks:
            if task_name in completed:
                continue
            if can_run(task_name, completed, tasks):
                print(f"Running: {task_name}")
                completed.add(task_name)
                ran_something = True
        iterations += 1
        if not ran_something:
            break   # nothing could run this pass — would indicate a real dependency problem

    if len(completed) < len(tasks):
        print("Warning: not all tasks completed — possible dependency issue")

run_all(tasks)
# Expected order: extract_a and extract_b print first (in some order, since
# neither depends on the other), THEN merge, THEN validate, THEN load —
# never merge before both extracts, never validate before merge, etc.
```
</details>

---

**Previous lesson:** [9.2 — Idempotency & Incremental Loads](lesson-9.2-idempotency-and-incremental-loads.md)
**Next lesson:** [9.4 — Data Quality Checks](lesson-9.4-data-quality-checks.md)
