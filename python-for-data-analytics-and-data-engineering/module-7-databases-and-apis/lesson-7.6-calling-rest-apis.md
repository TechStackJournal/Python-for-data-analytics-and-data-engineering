# Lesson 7.6 — Calling REST APIs with `requests`

## CONCEPT

**`requests`** is Python's standard library for making HTTP calls — the way you interact with REST APIs, which are the most common way data is exchanged over the web between different systems. A typical API call sends a request (specifying a URL, a method like `GET` or `POST`, and often authentication) and receives a **response**, usually containing JSON data (Module 2, Lesson 2.6).

```python
import requests

response = requests.get("https://api.example.com/products")
data = response.json()      # parses the JSON response body into a Python dict/list
```

Real-world APIs add three complications this lesson covers directly: **authentication** (proving who you are), **pagination** (large result sets split across multiple requests), and **rate limits** (a cap on how many requests you can make in a given time window).

## WHY IT MATTERS

APIs are, alongside databases (Lessons 7.1–7.3), one of the two primary sources of real data in professional work — pulling from a payment processor, a marketing platform, a shipping provider, or any third-party service almost always means calling their REST API. Understanding authentication, pagination, and rate limits isn't optional extra knowledge — nearly every real API requires handling at least one of these, and a script that ignores them will either fail outright or silently return incomplete data.

## SYNTAX REFERENCE

```python
import requests

# Basic GET request
response = requests.get("https://api.example.com/products")
response.status_code          # 200 = success, 404 = not found, 401 = unauthorized, etc.
response.json()                 # parse JSON body into Python dict/list
response.text                    # raw response body as a string
response.ok                       # True if status_code is in the 200-399 range

# Passing query parameters (added to the URL automatically, properly encoded)
response = requests.get(
    "https://api.example.com/products",
    params={"category": "electronics", "limit": 50},
)
# equivalent to: https://api.example.com/products?category=electronics&limit=50

# Authentication — API key in a header (a very common pattern)
headers = {"Authorization": "Bearer YOUR_API_KEY_HERE"}
response = requests.get("https://api.example.com/orders", headers=headers)

# Authentication — API key as a query parameter (another common pattern)
response = requests.get("https://api.example.com/orders", params={"api_key": "YOUR_KEY"})

# POST request with a JSON body
response = requests.post(
    "https://api.example.com/orders",
    json={"customer_id": 42, "total": 89.99},
    headers=headers,
)

# Checking for errors explicitly
if response.status_code == 200:
    data = response.json()
else:
    print(f"Request failed: {response.status_code}")

# Or, raising an exception automatically on a bad status code
response.raise_for_status()   # raises requests.HTTPError if status is 4xx or 5xx

# Pagination — a common "page-based" pattern (the exact structure varies by API)
all_results = []
page = 1
while True:
    response = requests.get("https://api.example.com/orders", params={"page": page})
    data = response.json()
    if not data["results"]:      # an empty page means we've reached the end
        break
    all_results.extend(data["results"])
    page += 1

# Setting a timeout (IMPORTANT — without one, a hung request can block forever)
response = requests.get("https://api.example.com/products", timeout=10)   # seconds
```

## WORKED EXAMPLE

Calling a real, free public API (no key required) to demonstrate the full pattern — fetching and paginating through data:

```python
import requests
import pandas as pd

# Using JSONPlaceholder, a free public test API — no authentication needed
BASE_URL = "https://jsonplaceholder.typicode.com"

# 1. A basic GET request
response = requests.get(f"{BASE_URL}/users", timeout=10)
print("Status code:", response.status_code)   # 200

users = response.json()
print(f"Fetched {len(users)} users")
print(users[0])   # a dict with keys like 'id', 'name', 'email', etc.

# 2. Converting the response directly into a DataFrame (a very common pattern —
# ties directly back to Module 4's pd.json_normalize concepts)
users_df = pd.json_normalize(users)
print(users_df[["id", "name", "email"]].head())

# 3. Using query parameters to filter server-side
posts_by_user = requests.get(f"{BASE_URL}/posts", params={"userId": 1}, timeout=10)
print(f"Posts by user 1: {len(posts_by_user.json())}")

# 4. Handling a request that FAILS — this endpoint deliberately doesn't exist
bad_response = requests.get(f"{BASE_URL}/nonexistent_endpoint", timeout=10)
print("Status code:", bad_response.status_code)   # 404

if not bad_response.ok:
    print(f"Request failed with status {bad_response.status_code} — handling gracefully")
else:
    data = bad_response.json()

# 5. A simulated pagination pattern (JSONPlaceholder doesn't paginate, but this
# is the standard shape for an API that DOES — e.g. GitHub's API, Stripe's API)
def fetch_all_posts_paginated(base_url, per_page=10):
    all_posts = []
    page = 1
    while True:
        response = requests.get(
            f"{base_url}/posts",
            params={"_page": page, "_limit": per_page},
            timeout=10,
        )
        page_data = response.json()
        if not page_data:            # an empty list means there are no more pages
            break
        all_posts.extend(page_data)
        page += 1
        if page > 20:                  # a safety cap, to avoid an accidental infinite loop
            break
    return all_posts

all_posts = fetch_all_posts_paginated(BASE_URL)
print(f"Total posts fetched across all pages: {len(all_posts)}")
```

## COMMON MISTAKES

**1. Not checking the response status before assuming `.json()` will work**
```python
response = requests.get(url)
data = response.json()   # if the request actually failed, this might raise an error
                            # or return an unexpected error payload instead of real data
```
Fix: check `response.ok` or `response.status_code`, or use `response.raise_for_status()`, before trusting the response body contains what you expect.

**2. Never setting a `timeout`**
```python
response = requests.get(url)   # no timeout — can hang INDEFINITELY if the server never responds
```
Fix: always pass an explicit `timeout=` (commonly 10-30 seconds) — an unattended pipeline script with no timeout can hang forever on a single unresponsive request, silently blocking everything downstream of it.

**3. Putting an API key directly in code instead of loading it from `.env`**
```python
headers = {"Authorization": "Bearer sk_live_abc123..."}   # same risk as Lesson 7.5!
```
Fix: this is the exact same principle from Lesson 7.5 — API keys are credentials, and belong in environment variables loaded via `python-dotenv`, never hardcoded.

**4. Building an infinite pagination loop with no termination safeguard**
```python
while True:
    page += 1
    # if the "is this the last page?" check has a bug, this NEVER stops
```
Fix: as shown in the worked example, always include a reasonable safety cap (a maximum page count) alongside the "proper" termination condition, so a bug in your pagination-end detection fails loudly (hits the cap) rather than running forever.

**5. Ignoring rate limits until a script starts getting blocked**
Most real APIs specify a maximum request rate (e.g., "100 requests per minute") and will start returning `429 Too Many Requests` responses, or temporarily block your API key entirely, if you exceed it. Fix: check the specific API's documentation for its rate limit, and either add a deliberate delay between requests (`time.sleep()`) or — better — read the API's response headers, which often directly report your remaining quota and when it resets, and pace accordingly. Lesson 7.7 covers the closely related topic of retrying gracefully when a rate limit (or any transient failure) is hit.

**Data Engineer Tip:** The pagination pattern in the worked example — loop through pages, accumulate results, stop when a page comes back empty — is the standard shape of an API-based "extract" step in a pipeline, exactly parallel to how Lesson 7.2's `pd.read_sql()` extracts from a database. Nearly every real API you'll integrate with in a production pipeline uses some variation of this page-based or cursor-based pagination, so this exact loop structure is worth having genuinely memorized rather than looked up each time.

## PRACTICE EXERCISES

*(These exercises use `https://jsonplaceholder.typicode.com`, a free public test API requiring no authentication — safe and appropriate for practice.)*

**Exercise 1 (Easy):** Make a GET request to `https://jsonplaceholder.typicode.com/users/1`, print the status code, and print the response's `.json()` result.

<details>
<summary>Hint</summary>
`requests.get(url, timeout=10)`
</details>

<details>
<summary>Solution</summary>

```python
import requests

response = requests.get("https://jsonplaceholder.typicode.com/users/1", timeout=10)
print(response.status_code)
print(response.json())
```
</details>

---

**Exercise 2 (Easy):** Request a URL that doesn't exist (`https://jsonplaceholder.typicode.com/nope`), check `response.ok`, and print an appropriate message depending on whether the request succeeded or failed.

<details>
<summary>Hint</summary>
`if response.ok: ... else: ...`
</details>

<details>
<summary>Solution</summary>

```python
import requests

response = requests.get("https://jsonplaceholder.typicode.com/nope", timeout=10)

if response.ok:
    print("Success:", response.json())
else:
    print(f"Request failed with status {response.status_code}")
```
</details>

---

**Exercise 3 (Medium):** Fetch all posts for `userId=3` using query parameters (`https://jsonplaceholder.typicode.com/posts`), convert the result into a DataFrame with `pd.json_normalize()`, and print just the `title` column.

<details>
<summary>Hint</summary>
`requests.get(url, params={"userId": 3})`, then `pd.json_normalize(response.json())`.
</details>

<details>
<summary>Solution</summary>

```python
import requests
import pandas as pd

response = requests.get(
    "https://jsonplaceholder.typicode.com/posts",
    params={"userId": 3},
    timeout=10,
)
posts_df = pd.json_normalize(response.json())
print(posts_df["title"])
```
</details>

---

**Exercise 4 (Medium):** Write a function `safe_get(url, params=None)` that makes a GET request with a 10-second timeout, checks `response.ok`, and returns the parsed JSON on success or `None` on failure (printing a warning message in the failure case). Test it against both a working endpoint and a broken one.

<details>
<summary>Hint</summary>
Wrap the check in an `if/else`, returning `response.json()` or `None` accordingly.
</details>

<details>
<summary>Solution</summary>

```python
import requests

def safe_get(url, params=None):
    response = requests.get(url, params=params, timeout=10)
    if response.ok:
        return response.json()
    else:
        print(f"Warning: request to {url} failed with status {response.status_code}")
        return None

good_data = safe_get("https://jsonplaceholder.typicode.com/users/1")
print(good_data)

bad_data = safe_get("https://jsonplaceholder.typicode.com/nonexistent")
print(bad_data)   # None
```
</details>

---

**Exercise 5 (Hard):** Write a function `fetch_all_comments_paginated(per_page=20, max_pages=10)` that paginates through `https://jsonplaceholder.typicode.com/comments` using the `_page`/`_limit` query parameter pattern (as in the worked example), accumulating all results into a single list, and stopping either when a page comes back empty OR when `max_pages` is reached (whichever happens first — this API actually has ~500 comments total, so with `per_page=20` you'd expect it to run out of real data well before hitting a reasonable `max_pages`). Print the total number of comments fetched, and convert the result into a DataFrame, printing its shape.

<details>
<summary>Hint</summary>
Same loop structure as the worked example's `fetch_all_posts_paginated`, adapted to the `/comments` endpoint, with the `max_pages` safety check as an explicit `if page > max_pages: break`.
</details>

<details>
<summary>Solution</summary>

```python
import requests
import pandas as pd

def fetch_all_comments_paginated(per_page=20, max_pages=10):
    all_comments = []
    page = 1
    while True:
        response = requests.get(
            "https://jsonplaceholder.typicode.com/comments",
            params={"_page": page, "_limit": per_page},
            timeout=10,
        )
        page_data = response.json()
        if not page_data:
            print(f"Reached the end of the data at page {page}")
            break
        all_comments.extend(page_data)
        page += 1
        if page > max_pages:
            print(f"Reached max_pages safety cap ({max_pages})")
            break
    return all_comments

comments = fetch_all_comments_paginated(per_page=20, max_pages=10)
print(f"Total comments fetched: {len(comments)}")   # up to 200 (20 per page * 10 pages cap)

comments_df = pd.json_normalize(comments)
print(comments_df.shape)
```
</details>

---

**Previous lesson:** [7.5 — Managing Credentials with python-dotenv](lesson-7.5-credentials-and-dotenv.md)
**Next lesson:** [7.7 — Retry Logic & Resilience with tenacity](lesson-7.7-retry-logic-and-resilience.md)
