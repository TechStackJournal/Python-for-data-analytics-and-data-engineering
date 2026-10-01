# Lesson 7.7 — Retry Logic & Resilience with `tenacity`

## CONCEPT

Network calls — to a database or an API — fail sometimes, even when your code is completely correct: a momentary network blip, a server briefly overloaded, a rate limit temporarily exceeded. **`tenacity`** is a library that adds automatic **retry logic** to a function via a simple decorator, so a transient failure gets retried (often successfully) instead of immediately crashing your whole pipeline.

```python
from tenacity import retry, stop_after_attempt, wait_fixed

@retry(stop=stop_after_attempt(3), wait=wait_fixed(2))
def call_flaky_api():
    return requests.get("https://api.example.com/data", timeout=10)
```

This function will now automatically retry up to 3 times, waiting 2 seconds between attempts, if it raises an exception — no manual `try`/`except`/loop code required.

## WHY IT MATTERS

A pipeline that fails completely on the very first transient network hiccup is fragile in a way that's genuinely costly in production — a job that runs nightly and fails because of one momentary blip means missing data, a page to whoever's on call, and often a fully manual rerun. Building in sensible, automatic retry behavior for the specific failures that are actually worth retrying (network timeouts, rate limits, temporary server errors) — while *not* retrying failures that will never succeed no matter how many times you try (a genuinely wrong password, a malformed query) — is one of the clearest, highest-leverage resilience patterns in real pipeline code.

## SYNTAX REFERENCE

```python
from tenacity import (
    retry,
    stop_after_attempt,
    stop_after_delay,
    wait_fixed,
    wait_exponential,
    retry_if_exception_type,
)

# Basic retry: try up to 3 times total, waiting 2 seconds between attempts
@retry(stop=stop_after_attempt(3), wait=wait_fixed(2))
def unreliable_call():
    ...

# Exponential backoff — wait longer after each successive failure (a common,
# API-friendly pattern: 1s, then 2s, then 4s, then 8s, etc.)
@retry(stop=stop_after_attempt(5), wait=wait_exponential(multiplier=1, min=1, max=30))
def call_with_backoff():
    ...

# Stop after a total time limit instead of a fixed attempt count
@retry(stop=stop_after_delay(30))    # keep retrying for up to 30 seconds total
def time_limited_retry():
    ...

# Only retry SPECIFIC exception types — don't retry errors that will never succeed
@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(2),
    retry=retry_if_exception_type(requests.exceptions.ConnectionError),
)
def only_retry_connection_errors():
    ...
    # a different exception type (e.g. a 401 Unauthorized) will NOT be retried —
    # it will raise immediately, since retrying a bad password can never succeed

# Combining multiple stop/wait conditions
@retry(
    stop=stop_after_attempt(5) | stop_after_delay(60),   # whichever limit hits first
    wait=wait_exponential(multiplier=1, min=2, max=20),
)
def robust_call():
    ...

# Logging each retry attempt (useful for visibility into what's happening)
import logging
from tenacity import before_sleep_log

logger = logging.getLogger(__name__)

@retry(
    stop=stop_after_attempt(3),
    wait=wait_fixed(2),
    before_sleep=before_sleep_log(logger, logging.WARNING),
)
def logged_retry_call():
    ...

# Accessing retry statistics after a call (e.g. how many attempts it took)
result = unreliable_call.retry.statistics   # a dict with attempt_number, etc.
```

## WORKED EXAMPLE

Adding resilient retry logic to an API call and a database query — the two connectivity patterns from earlier lessons in this module:

```python
import requests
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
)
import psycopg2

# --- Simulating a flaky API call, retried with exponential backoff ---
call_attempt_count = 0

@retry(
    stop=stop_after_attempt(4),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type(requests.exceptions.RequestException),
)
def fetch_data_with_retry(url):
    global call_attempt_count
    call_attempt_count += 1
    print(f"Attempt #{call_attempt_count}...")

    response = requests.get(url, timeout=5)
    response.raise_for_status()   # raises requests.HTTPError on a 4xx/5xx status —
                                     # this IS a RequestException subclass, so it gets retried
    return response.json()

# A real, working call — succeeds on the first attempt
data = fetch_data_with_retry("https://jsonplaceholder.typicode.com/users/1")
print("Success:", data["name"])
print(f"Total attempts needed: {call_attempt_count}")   # 1

# --- Simulating a genuinely broken call, showing retries exhaust and THEN raise ---
call_attempt_count = 0
try:
    # This endpoint doesn't exist — every attempt will fail, retries won't help,
    # but this demonstrates the RETRY BEHAVIOR itself (it tries 4 times, waiting
    # longer each time, before finally giving up and raising)
    fetch_data_with_retry("https://jsonplaceholder.typicode.com/this_will_always_404")
except requests.exceptions.RequestException as e:
    print(f"Gave up after {call_attempt_count} attempts. Final error: {e}")

# --- Retry logic for a database connection, only for genuinely transient errors ---
@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=2, max=15),
    retry=retry_if_exception_type(psycopg2.OperationalError),
    # NOTE: psycopg2.OperationalError covers connection issues (network,
    # server temporarily down) — genuinely worth retrying. A different error
    # like psycopg2.errors.UndefinedTable (a typo in a table name) would NOT
    # be caught by this specific retry_if_exception_type, and correctly so —
    # retrying a query against a table that doesn't exist will never succeed
)
def connect_with_retry(host, dbname, user, password):
    return psycopg2.connect(host=host, dbname=dbname, user=user, password=password)

try:
    conn = connect_with_retry("localhost", "ecommerce", "postgres", "your_password")
    print("Connected successfully")
    conn.close()
except psycopg2.OperationalError as e:
    print(f"Failed to connect after retries: {e}")
```

## COMMON MISTAKES

**1. Retrying every exception type indiscriminately**
```python
@retry(stop=stop_after_attempt(5))   # no `retry=` filter — retries EVERYTHING, including bugs!
def my_function():
    return 1 / 0   # a ZeroDivisionError from a genuine code bug — will NEVER succeed no matter how many retries
```
Fix: always specify `retry=retry_if_exception_type(...)` with the *specific* exception types that represent genuinely transient, retry-worthy failures (network timeouts, connection errors, rate limits) — retrying a logic bug or a permanent error (bad credentials, a malformed query, a 404) just wastes time before failing anyway, and can mask a real bug behind a slow, confusing failure.

**2. Using a fixed, short wait time against a rate-limited API**
```python
@retry(stop=stop_after_attempt(10), wait=wait_fixed(1))   # hammers the API every 1 second
```
If the failure is specifically a rate limit, retrying immediately and repeatedly can make the situation worse, not better. Fix: exponential backoff (`wait_exponential`) is the standard, API-friendly pattern — it gives the server progressively more breathing room on each retry, which is exactly what most APIs' own documentation recommends for handling their rate limits gracefully.

**3. Setting `stop_after_attempt()` too high for a user-facing or time-sensitive operation**
A pipeline that retries 20 times with exponential backoff before giving up might take several minutes just to fail — fine for an unattended nightly batch job, but potentially unacceptable if something downstream is waiting on a fast response. Fix: match your retry budget (attempt count, max wait) to how much delay is actually acceptable for that specific call's context.

**4. Not distinguishing between "should retry" and "should alert a human"**
Even with retries in place, a function that ultimately exhausts every retry attempt and still fails represents a real problem worth surfacing loudly (logging clearly, alerting, failing the pipeline run visibly) — retry logic reduces *transient* failures, it doesn't eliminate the need to handle a *persistent* one. Fix: always wrap a retried function call in its own `try`/`except` at the call site too, so a final, genuine failure (after retries are exhausted) is still caught, logged clearly, and handled deliberately — not left to crash the whole script with an unclear traceback.

**5. Forgetting retries add real latency, and not accounting for it elsewhere**
A function wrapped in `@retry(stop=stop_after_attempt(5), wait=wait_exponential(max=30))` could, in the worst case, take well over a minute to finally give up — if this call happens inside a loop processing thousands of records, and a meaningful fraction of them hit this worst case, the total pipeline runtime can balloon unexpectedly. Fix: be deliberate about retry budgets specifically for calls inside a loop over many records, where the *per-call* worst case multiplies across the whole batch.

**Data Engineer Tip:** The distinction in the worked example's comments — `psycopg2.OperationalError` (worth retrying) vs. a table-not-found error (never worth retrying) — is one of the more subtle but important judgment calls in production pipeline design. Getting this filter wrong in either direction has a real cost: too broad, and you waste time (and potentially worsen rate limiting) retrying failures that will never succeed; too narrow, and a genuinely transient failure crashes the whole pipeline unnecessarily. This judgment, more than the `tenacity` syntax itself, is the actual skill this lesson is building toward.

## PRACTICE EXERCISES

**Exercise 1 (Easy):** Write a function `flaky_function()` that raises a `ValueError` the first two times it's called (track this with a global counter) and succeeds on the third, returning `"success"`. Wrap it with `@retry(stop=stop_after_attempt(3), wait=wait_fixed(1))` and confirm it succeeds.

<details>
<summary>Hint</summary>
Use a global counter incremented each call; raise `ValueError` while the counter is below 3.
</details>

<details>
<summary>Solution</summary>

```python
from tenacity import retry, stop_after_attempt, wait_fixed

attempt_count = 0

@retry(stop=stop_after_attempt(3), wait=wait_fixed(1))
def flaky_function():
    global attempt_count
    attempt_count += 1
    if attempt_count < 3:
        raise ValueError(f"Simulated failure on attempt {attempt_count}")
    return "success"

result = flaky_function()
print(result)               # "success"
print(attempt_count)          # 3
```
</details>

---

**Exercise 2 (Easy):** Write a function that always fails (raises `ValueError` every time), wrapped with `@retry(stop=stop_after_attempt(3), wait=wait_fixed(1))`. Call it inside a `try`/`except` and confirm it eventually raises after exhausting all 3 attempts, printing how many attempts were made.

<details>
<summary>Hint</summary>
`tenacity` re-raises the original exception once `stop_after_attempt` is reached — catch it with a regular `try`/`except ValueError`.
</details>

<details>
<summary>Solution</summary>

```python
from tenacity import retry, stop_after_attempt, wait_fixed

always_fails_count = 0

@retry(stop=stop_after_attempt(3), wait=wait_fixed(1))
def always_fails():
    global always_fails_count
    always_fails_count += 1
    raise ValueError("This always fails")

try:
    always_fails()
except ValueError as e:
    print(f"Gave up after {always_fails_count} attempts: {e}")
```
</details>

---

**Exercise 3 (Medium):** Use `wait_exponential(multiplier=1, min=1, max=10)` instead of a fixed wait, on a function that always fails, with `stop_after_attempt(4)`. Print the elapsed time before and after the call (using `time.time()`) to observe that the retries take progressively longer, rather than a fixed interval.

<details>
<summary>Hint</summary>
Wrap the call in `try/except`, and measure `time.time()` immediately before and after.
</details>

<details>
<summary>Solution</summary>

```python
import time
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(stop=stop_after_attempt(4), wait=wait_exponential(multiplier=1, min=1, max=10))
def always_fails_backoff():
    raise ValueError("Simulated persistent failure")

start = time.time()
try:
    always_fails_backoff()
except ValueError:
    pass
elapsed = time.time() - start
print(f"Total elapsed time across all retries: {elapsed:.1f} seconds")
# Should be noticeably more than a few seconds, since each wait grows —
# roughly 1s + 2s + 4s between the 4 attempts (capped at max=10 each)
```
</details>

---

**Exercise 4 (Medium):** Write two custom exception classes, `TransientError` and `PermanentError` (both inheriting from `Exception`). Write a function that raises `PermanentError` (not `TransientError`) every time it's called. Wrap it with `@retry(retry=retry_if_exception_type(TransientError), stop=stop_after_attempt(3), wait=wait_fixed(1))`. Call it and confirm the function raises IMMEDIATELY (only 1 attempt, not 3), since `PermanentError` doesn't match the `retry_if_exception_type` filter.

<details>
<summary>Hint</summary>
Since `retry_if_exception_type(TransientError)` is specified, an exception of a DIFFERENT type (`PermanentError`) won't trigger a retry at all — it propagates immediately.
</details>

<details>
<summary>Solution</summary>

```python
from tenacity import retry, stop_after_attempt, wait_fixed, retry_if_exception_type

class TransientError(Exception):
    pass

class PermanentError(Exception):
    pass

permanent_attempt_count = 0

@retry(
    retry=retry_if_exception_type(TransientError),
    stop=stop_after_attempt(3),
    wait=wait_fixed(1),
)
def always_permanent_failure():
    global permanent_attempt_count
    permanent_attempt_count += 1
    raise PermanentError("This will never succeed, no matter how many retries")

try:
    always_permanent_failure()
except PermanentError as e:
    print(f"Failed immediately after {permanent_attempt_count} attempt(s): {e}")
    # permanent_attempt_count is 1 — NOT 3, because PermanentError doesn't
    # match the retry_if_exception_type(TransientError) filter, so tenacity
    # doesn't retry it at all and lets it propagate right away
```
</details>

---

**Exercise 5 (Hard):** Write a function `fetch_with_resilience(url)` that: (1) makes a GET request with a 5-second timeout, (2) calls `response.raise_for_status()` to turn a bad HTTP status into an exception, (3) is wrapped with retry logic using exponential backoff, retrying only `requests.exceptions.RequestException` (which covers both connection errors and HTTP errors from `raise_for_status()`), stopping after 3 attempts, and (4) is called from a `try`/`except` block at the call site that prints a clear, final failure message if all retries are exhausted (don't let the exception crash the whole script). Test it against both a working URL (`https://jsonplaceholder.typicode.com/users/1`) and a broken one (`https://jsonplaceholder.typicode.com/this_endpoint_does_not_exist`), confirming the working one succeeds immediately and the broken one retries 3 times before failing gracefully.

<details>
<summary>Hint</summary>
Combine the `@retry` decorator pattern from the worked example with a `try`/`except requests.exceptions.RequestException` at the point where you actually call the decorated function — the decorator handles the retries, but the exception still needs to be caught once retries are exhausted.
</details>

<details>
<summary>Solution</summary>

```python
import requests
from tenacity import retry, stop_after_attempt, wait_exponential, retry_if_exception_type

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10),
    retry=retry_if_exception_type(requests.exceptions.RequestException),
)
def fetch_with_resilience(url):
    response = requests.get(url, timeout=5)
    response.raise_for_status()
    return response.json()

def safe_fetch(url):
    try:
        return fetch_with_resilience(url)
    except requests.exceptions.RequestException as e:
        print(f"Failed to fetch {url} after all retry attempts: {e}")
        return None

# Working URL — should succeed on the first attempt
good_result = safe_fetch("https://jsonplaceholder.typicode.com/users/1")
print("Good result:", good_result["name"] if good_result else None)

# Broken URL — should retry 3 times (with growing delays), then fail gracefully
bad_result = safe_fetch("https://jsonplaceholder.typicode.com/this_endpoint_does_not_exist")
print("Bad result:", bad_result)   # None — script continues running normally, doesn't crash
```
</details>

---

## Module 7 Complete

You've now covered connecting Python to both databases and APIs — the two primary sources of real-world data — along with the security and reliability practices (parameterized queries, credential management, retry logic) that turn a working script into something genuinely trustworthy to run unattended. Module 8 formalizes this further: structuring code into proper packages, adding tests, logging, and type hints — the habits that separate a script from production-grade pipeline code.

**Previous lesson:** [7.6 — Calling REST APIs with requests](lesson-7.6-calling-rest-apis.md)
**Next module:** [Module 8 — Production-Grade Python for Data Engineering](../module-8-production-grade-python/README.md)
