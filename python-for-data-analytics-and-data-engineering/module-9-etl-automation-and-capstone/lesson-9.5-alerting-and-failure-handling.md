# Lesson 9.5 — Logging, Alerting & Failure Handling

## CONCEPT

A scheduled pipeline runs with nobody watching — Module 8 already covered structured logging as the record of *what happened*. This lesson extends that into **alerting**: actively notifying a human when something goes wrong, rather than relying on someone to go looking through log files after the fact. It also covers **failure handling** at the pipeline-run level — deciding what the pipeline should do, and communicate, when a stage genuinely fails despite retries (Module 7's `tenacity`) having already been exhausted.

```python
try:
    run_pipeline()
except Exception as e:
    logger.exception("Pipeline failed")
    send_alert(f"Pipeline FAILED: {e}")   # a human finds out immediately, not next week
    raise
```

## WHY IT MATTERS

A pipeline that fails silently at 3 AM and nobody notices until a stakeholder complains about missing data two days later represents exactly the gap this lesson closes. Combined with everything else in this module — idempotency, data quality gates, retries — alerting is the final piece that makes a pipeline genuinely trustworthy to run unattended: it fails loudly, to a human, the moment something goes wrong, rather than failing silently into a log file nobody's reading in real time.

## SYNTAX REFERENCE

```python
import logging
import smtplib
from email.message import EmailMessage
import requests

logger = logging.getLogger(__name__)

# --- Alerting via email (using Python's built-in smtplib) ---
def send_email_alert(subject: str, body: str, to_address: str) -> None:
    msg = EmailMessage()
    msg["Subject"] = subject
    msg["From"] = "pipeline@yourcompany.com"
    msg["To"] = to_address
    msg.set_content(body)

    with smtplib.SMTP("smtp.yourprovider.com", 587) as server:
        server.starttls()
        server.login("username", "password")   # from .env, per Module 7, Lesson 7.5
        server.send_message(msg)

# --- Alerting via a Slack webhook (a common, simpler alternative) ---
def send_slack_alert(message: str, webhook_url: str) -> None:
    response = requests.post(webhook_url, json={"text": message}, timeout=10)
    response.raise_for_status()

# --- A reusable alert function the pipeline calls on failure ---
def alert_on_failure(pipeline_name: str, error: Exception, webhook_url: str) -> None:
    message = f":rotating_light: *{pipeline_name}* failed: `{error}`"
    try:
        send_slack_alert(message, webhook_url)
    except Exception:
        # If the ALERT itself fails, log it — but don't let that crash the
        # pipeline's own failure handling with a second, confusing exception
        logger.exception("Failed to send alert notification")

# --- Wrapping pipeline execution with full failure handling ---
def run_pipeline_with_alerting(pipeline_fn, pipeline_name: str, webhook_url: str) -> int:
    try:
        pipeline_fn()
        logger.info(f"{pipeline_name} completed successfully")
        return 0
    except Exception as e:
        logger.exception(f"{pipeline_name} failed")
        alert_on_failure(pipeline_name, e, webhook_url)
        return 1
```

## WORKED EXAMPLE

A complete pipeline runner that logs thoroughly, alerts on failure via Slack, and distinguishes between a few different kinds of failure with different responses:

```python
import logging
import requests
from datetime import datetime

logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s [%(levelname)s] %(message)s",
)
logger = logging.getLogger(__name__)


class DataQualityError(Exception):
    """Raised when the pipeline's data quality gate fails."""


def send_slack_alert(message: str, webhook_url: str) -> None:
    try:
        response = requests.post(webhook_url, json={"text": message}, timeout=10)
        response.raise_for_status()
    except Exception:
        logger.exception("Failed to send Slack alert — continuing anyway")


def extract():
    logger.info("Extracting...")
    return {"rows": 500}


def transform(raw):
    logger.info("Transforming...")
    return raw


def validate(clean):
    logger.info("Validating...")
    if clean["rows"] == 0:
        raise DataQualityError("Zero rows after transform — likely an upstream extraction problem")
    return clean


def load(validated):
    logger.info("Loading...")
    return True


def run_pipeline():
    raw = extract()
    clean = transform(raw)
    validated = validate(clean)
    load(validated)


def main(webhook_url: str) -> int:
    pipeline_name = "ecommerce_daily_etl"
    start_time = datetime.now()

    try:
        run_pipeline()
        duration = (datetime.now() - start_time).total_seconds()
        logger.info(f"{pipeline_name} succeeded in {duration:.1f}s")
        return 0

    except DataQualityError as e:
        # A data quality failure is a DIFFERENT kind of problem than a crash —
        # worth a distinct, more specific alert message
        logger.error(f"Data quality gate failed: {e}")
        send_slack_alert(
            f":warning: *{pipeline_name}*: data quality check failed — `{e}`. "
            f"No data was loaded. Investigate the source data before re-running.",
            webhook_url,
        )
        return 1

    except Exception as e:
        # Any OTHER unexpected failure — a genuine crash
        logger.exception(f"{pipeline_name} failed unexpectedly")
        send_slack_alert(
            f":rotating_light: *{pipeline_name}* crashed unexpectedly: `{e}`. "
            f"Check the logs immediately.",
            webhook_url,
        )
        return 1


if __name__ == "__main__":
    import sys
    exit_code = main(webhook_url="https://hooks.slack.com/services/XXX/YYY/ZZZ")
    sys.exit(exit_code)
```

Notice the two different `except` blocks produce genuinely different alert messages — a known, expected failure mode (bad data) gets a calmer, more specific message than a true unexpected crash, which helps whoever's on the receiving end triage faster.

## COMMON MISTAKES

**1. Alerting on every single log message, flooding the channel**
```python
logger.info("Processing row 1")   # if this ALSO triggers an alert, you'll get thousands per run
```
Fix: alerts should be reserved for genuine failures (or, at most, significant milestones like "pipeline completed") — routine progress belongs in logs only, not in an alert channel. Alert fatigue (too many notifications) causes people to tune out and miss the ones that actually matter.

**2. Letting a failure in the alerting code itself crash the pipeline's error handling**
```python
except Exception as e:
    send_slack_alert(...)   # if THIS line itself throws (e.g. webhook URL is down),
                               # the exception here replaces the original, more useful one!
```
Fix: wrap the alert-sending call in its own `try/except` (as shown in the worked example's `send_slack_alert`) — a failure to *send* an alert about a failure shouldn't itself cause a confusing, unrelated second exception to be what actually surfaces.

**3. Sending a generic, unhelpful alert message**
```python
send_slack_alert("Pipeline failed")   # failed HOW? WHERE? What should I do about it?
```
Fix: include the pipeline name, the specific error message, and ideally a hint about what to do next (as in the worked example) — a useful alert saves the person reading it a trip into the logs just to understand what happened.

**4. Not distinguishing between different categories of failure**
Treating a data quality issue (bad but recoverable — fix the source data and re-run) identically to a genuine code crash (a bug, a connection failure) in logging and alerting makes it harder for whoever's responding to triage quickly. Fix: custom exception types (Module 8, Lesson 8.4) combined with separate `except` blocks, as in the worked example, let you tailor both the log severity and the alert message to what kind of problem actually occurred.

**5. Having no alerting at all and relying purely on someone checking logs proactively**
In practice, nobody reliably checks log files every day "just in case" — Fix: even a minimal alert (a single Slack message on failure) is a meaningful upgrade over logs alone, since it's push-based (comes to you) rather than pull-based (you have to go looking).

**Data Engineer Tip:** In a real production environment, this kind of custom alerting code is often eventually replaced or supplemented by dedicated monitoring/observability tooling (Azure Monitor, Datadog, PagerDuty) that handles on-call rotations, alert routing, and escalation more robustly than a single Slack webhook call. But understanding the underlying principle — logged failures should actively reach a human, not just sit in a file — is what those tools are automating, and a simple webhook-based alert like this one is a completely legitimate, real approach for a smaller pipeline or a portfolio project.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Write a function `send_console_alert(message)` that simulates alerting by printing a clearly-formatted alert message (e.g., prefixed with `"🚨 ALERT: "`) to the console — standing in for a real Slack/email call for testing purposes. Call it from inside an `except` block after an intentionally failing operation.

<details>
<summary>Hint</summary>
A simple `print(f"🚨 ALERT: {message}")` function, called from a `try/except`.
</details>

<details>
<summary>Solution</summary>

```python
def send_console_alert(message: str) -> None:
    print(f"🚨 ALERT: {message}")

try:
    result = 10 / 0
except Exception as e:
    send_console_alert(f"Pipeline failed: {e}")
```
</details>

---

**Exercise 2 (Easy):** Write a `main()` function that runs a (possibly failing) pipeline function, logs success or failure appropriately using the `logging` module, and calls `send_console_alert()` (from Exercise 1) only on failure — never on success.

<details>
<summary>Hint</summary>
`try/except` around the pipeline call, `logger.info()` in the success path, `logger.exception()` + the alert call in the except path.
</details>

<details>
<summary>Solution</summary>

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def send_console_alert(message: str) -> None:
    print(f"🚨 ALERT: {message}")

def run_pipeline(should_fail: bool):
    if should_fail:
        raise RuntimeError("simulated pipeline failure")
    return "ok"

def main(should_fail: bool) -> int:
    try:
        run_pipeline(should_fail)
        logger.info("Pipeline succeeded")
        return 0
    except Exception as e:
        logger.exception("Pipeline failed")
        send_console_alert(f"Pipeline failed: {e}")
        return 1

main(False)   # logs success, no alert
main(True)     # logs failure, sends an alert
```
</details>

---

**Exercise 3 (Medium):** Write a custom exception class `DataQualityError(Exception)`. Modify Exercise 2's `main()` to handle `DataQualityError` and any other `Exception` with two DIFFERENT alert messages — the data quality case should mention "no data was loaded," the generic case should say "unexpected crash." Test both paths.

<details>
<summary>Hint</summary>
Two separate `except` blocks — the more specific `DataQualityError` one must come BEFORE the general `Exception` one (an ordering rule from Module 1's conditionals carries over to exception handling too).
</details>

<details>
<summary>Solution</summary>

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class DataQualityError(Exception):
    pass

def send_console_alert(message: str) -> None:
    print(f"🚨 ALERT: {message}")

def run_pipeline(mode: str):
    if mode == "bad_data":
        raise DataQualityError("found negative totals")
    elif mode == "crash":
        raise RuntimeError("database connection lost")
    return "ok"

def main(mode: str) -> int:
    try:
        run_pipeline(mode)
        logger.info("Pipeline succeeded")
        return 0
    except DataQualityError as e:
        logger.error(f"Data quality failure: {e}")
        send_console_alert(f"Data quality check failed, no data loaded: {e}")
        return 1
    except Exception as e:
        logger.exception("Unexpected crash")
        send_console_alert(f"Unexpected crash: {e}")
        return 1

main("bad_data")
main("crash")
main("ok")
```
</details>

---

**Exercise 4 (Medium):** Simulate a flaky alert-sending function `send_alert_flaky(message)` that raises an exception 50% of the time (use `random.random() < 0.5`). Write `safe_send_alert(message)` that wraps it in its own `try/except`, logging (but not re-raising) if the alert itself fails to send — then call it from a pipeline's failure-handling block, confirming the ORIGINAL pipeline failure is still what gets logged/returned, even when the alert itself also fails.

<details>
<summary>Hint</summary>
The outer pipeline's `except` block should call `safe_send_alert()`, and `safe_send_alert()`'s own internal `try/except` should swallow (just log) any failure from `send_alert_flaky()`, never letting it propagate up and replace the original pipeline error.
</details>

<details>
<summary>Solution</summary>

```python
import logging
import random

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def send_alert_flaky(message: str) -> None:
    if random.random() < 0.5:
        raise ConnectionError("could not reach alert service")
    print(f"🚨 ALERT SENT: {message}")

def safe_send_alert(message: str) -> None:
    try:
        send_alert_flaky(message)
    except Exception:
        logger.exception("Failed to send alert — continuing anyway")

def run_pipeline():
    raise RuntimeError("the real pipeline failure")

def main() -> int:
    try:
        run_pipeline()
        return 0
    except Exception as e:
        logger.exception("Pipeline failed")
        safe_send_alert(f"Pipeline failed: {e}")   # may itself fail, but won't crash main()
        return 1

exit_code = main()
print("Exit code:", exit_code)   # always 1, regardless of whether the alert itself succeeded
```
</details>

---

**Exercise 5 (Hard):** Build a complete failure-handling wrapper function `run_with_full_handling(pipeline_fn, pipeline_name, alert_fn)` that: (1) times the pipeline run using `datetime.now()` before and after; (2) on success, logs the duration; (3) on a `DataQualityError`, logs an error-level message, calls `alert_fn()` with a message that includes the pipeline name AND the specific error, and returns `1`; (4) on any OTHER exception, logs with `logger.exception()`, calls `alert_fn()` with a differently-worded message, and returns `1`; (5) critically, the alert call itself must be wrapped so a failure to alert never masks the original pipeline error. Test it against three scenarios: success, a `DataQualityError`, and a generic crash — printing the returned exit code each time.

<details>
<summary>Hint</summary>
Combine the patterns from Exercises 3 and 4: two `except` blocks (specific before general) for the pipeline's own failure, each calling a SEPARATE inner `try/except` around `alert_fn()` so an alert failure never overwrites or masks the original pipeline exception being handled.
</details>

<details>
<summary>Solution</summary>

```python
import logging
from datetime import datetime

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class DataQualityError(Exception):
    pass


def safe_alert(alert_fn, message: str) -> None:
    try:
        alert_fn(message)
    except Exception:
        logger.exception("Alert sending failed — continuing without crashing the handler")


def run_with_full_handling(pipeline_fn, pipeline_name: str, alert_fn) -> int:
    start = datetime.now()
    try:
        pipeline_fn()
        duration = (datetime.now() - start).total_seconds()
        logger.info(f"{pipeline_name} succeeded in {duration:.2f}s")
        return 0

    except DataQualityError as e:
        logger.error(f"{pipeline_name}: data quality check failed — {e}")
        safe_alert(alert_fn, f"[{pipeline_name}] Data quality failure: {e}. No data loaded.")
        return 1

    except Exception as e:
        logger.exception(f"{pipeline_name}: unexpected failure")
        safe_alert(alert_fn, f"[{pipeline_name}] Unexpected crash: {e}. Check logs.")
        return 1


def console_alert(message: str) -> None:
    print(f"🚨 {message}")


# Test scenarios
def good_pipeline():
    pass

def bad_data_pipeline():
    raise DataQualityError("negative totals found")

def crashing_pipeline():
    raise ConnectionError("database unreachable")


print("Success case exit code:", run_with_full_handling(good_pipeline, "test_pipeline", console_alert))
print("DQ failure exit code:", run_with_full_handling(bad_data_pipeline, "test_pipeline", console_alert))
print("Crash exit code:", run_with_full_handling(crashing_pipeline, "test_pipeline", console_alert))
```
</details>

---

**Previous lesson:** [9.4 — Data Quality Checks](lesson-9.4-data-quality-checks.md)
**Next lesson:** [9.6 — Packaging the Pipeline for Deployment](lesson-9.6-packaging-for-deployment.md)
