---
name: resilience-patterns
description: Build fault-tolerant backend services using circuit breakers, retries, timeouts, bulkheads, and fallbacks. Covers implementation patterns and when to apply each.
origin: local
---

# Resilience Patterns

Use this skill when designing services that call external dependencies (APIs, DBs, queues) and must tolerate failures gracefully.

## When to Use

- Service calls a downstream API or database that may be slow or unavailable
- A single dependency failure should not cascade to take down the whole system
- Reviewing a service for failure mode coverage

---

## Core Patterns

### 1. Timeout
Never wait indefinitely. Set a maximum time for every external call.

```python
import httpx

async with httpx.AsyncClient(timeout=3.0) as client:
    response = await client.get("https://api.example.com/data")
```

**Rule of thumb:**
- Internal service calls: 100–500ms
- External APIs: 2–5s
- DB queries: 1–3s (flag slow queries, don't just increase timeout)

---

### 2. Retry with Exponential Backoff
Retry transient failures, but back off to avoid hammering a struggling service.

```python
import time, random

def call_with_retry(fn, max_attempts=3):
    for attempt in range(max_attempts):
        try:
            return fn()
        except TransientError as e:
            if attempt == max_attempts - 1:
                raise
            wait = (2 ** attempt) + random.uniform(0, 1)  # jitter
            time.sleep(wait)
```

**Only retry on:**
- Network timeouts
- 429 (rate limited), 503 (service unavailable)
- Idempotent operations (GET, PUT, DELETE)

**Never retry on:**
- 4xx client errors (bad request, unauthorized)
- Non-idempotent writes without idempotency keys

---

### 3. Circuit Breaker
Stop calling a failing service to give it time to recover.

**States:**
```
CLOSED → (failure threshold exceeded) → OPEN → (timeout) → HALF-OPEN
  ↑                                                              |
  └──────────────── (probe succeeds) ───────────────────────────┘
```

| State | Behavior |
|-------|----------|
| **Closed** | Normal operation, counting failures |
| **Open** | Fail fast — no calls made, return fallback immediately |
| **Half-Open** | Allow one probe request to test recovery |

```python
# Using `pybreaker`
import pybreaker

breaker = pybreaker.CircuitBreaker(fail_max=5, reset_timeout=30)

@breaker
def call_payment_service(order_id):
    return requests.post("https://payments/charge", json={"order_id": order_id})
```

**Thresholds:** Open after 5 failures in 10s; probe after 30s.

---

### 4. Bulkhead
Isolate resources so one slow dependency doesn't exhaust shared thread/connection pools.

```python
# Separate connection pools per downstream service
db_pool = ConnectionPool(max_size=10)        # for database
payments_pool = ConnectionPool(max_size=5)   # for payment API
search_pool = ConnectionPool(max_size=5)     # for search service
```

If the search service hangs and exhausts `search_pool`, the database pool is unaffected.

**Also applies to:** thread pools, semaphores, Kubernetes resource limits per container.

---

### 5. Fallback
Return a degraded but useful response when a dependency fails.

```python
def get_product_recommendations(user_id):
    try:
        return recommendation_service.get(user_id)
    except (Timeout, CircuitBreakerOpen):
        return get_popular_products()   # static fallback
```

**Fallback options (best to worst):**
1. Cached previous result
2. Default/static data
3. Empty response with graceful UI degradation
4. Fail fast with clear error (last resort)

---

### 6. Rate Limiting (Client-Side)
Throttle outbound calls to respect upstream limits and avoid self-inflicted 429s.

```python
from ratelimit import limits, sleep_and_retry

@sleep_and_retry
@limits(calls=100, period=60)
def call_external_api(payload):
    return requests.post("https://api.example.com", json=payload)
```

---

## Combining Patterns

Typical order of wrapping for a service call:

```
Request
  → Bulkhead (acquire semaphore)
    → Circuit Breaker (check state)
      → Timeout (set deadline)
        → Retry (on transient error)
          → Actual call
      → Fallback (if all retries fail or circuit open)
```

---

## Resilience Checklist

- [ ] All external calls have a timeout set
- [ ] Retries use exponential backoff with jitter
- [ ] Retries only on idempotent or safe operations
- [ ] Circuit breaker configured for each downstream dependency
- [ ] Separate connection/thread pools per dependency (bulkhead)
- [ ] Fallback defined for every critical dependency
- [ ] Failure metrics and circuit state exposed to monitoring
