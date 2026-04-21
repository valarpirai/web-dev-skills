---
name: refactoring-playbook
description: Guide principled refactoring — identify when to refactor, enforce safe steps, flag algorithm complexity issues, and prevent mixing refactor with new features.
origin: local
---

# Refactoring Playbook

Use this skill to guide when and how to refactor safely. Based on The Pragmatic Programmer Tips 44–49.

## When to Use

- Planning a refactoring session and need to identify what to prioritize
- Reviewing a PR that mixes refactoring with new functionality
- Identifying performance bottlenecks caused by algorithmic complexity

---

## Core Principles

**Tip 44: Don't Program by Coincidence.** Understand why the code works, not just that it works.

**Tip 47: Refactor Early, Refactor Often.** Like a garden — small, regular tending beats a major overhaul.

**Tip 48: Design to Test.** Code that can't be tested without heroics needs to be refactored before it grows.

---

## When to Refactor

Refactor when you see any of these triggers:

| Trigger | Description |
|---------|-------------|
| **DRY violation** | Same logic in 2+ places — next bug fix will miss one of them |
| **Non-orthogonal design** | Changing one thing requires changing something unrelated |
| **Outdated knowledge** | Requirements evolved, but the code reflects old assumptions |
| **Performance** | A profiler shows a hotspot that algorithmic improvement can fix |
| **Untestable code** | Can't write a unit test without mocking 4 things — the design is wrong |

**Do NOT refactor when:**
- You're in the middle of adding a new feature — finish the feature first
- There are no tests — add tests first, then refactor
- A release deadline is today — document the debt, address it next cycle

---

## The Three Rules of Safe Refactoring

### Rule 1: Never Mix Refactor and Feature
A refactoring commit must not change behavior. A feature commit must not reorganize code.

```
# BAD: one commit that does both
git commit -m "Add retry logic and reorganize service layer"

# GOOD: separate commits
git commit -m "Reorganize service layer (no behavior change)"
git commit -m "Add retry logic to payment service"
```

If you start refactoring and realize you need a new feature to do it cleanly, stop. Create a TODO/ticket. Finish the refactor. Then add the feature.

### Rule 2: Tests First
If the code you're about to refactor has no tests, write tests that capture the current behavior before touching anything.

```bash
# Verify tests pass before refactor
./mvnw test -Dtest=OrderServiceTest

# Make your changes
# Run again — same result = safe refactor
./mvnw test -Dtest=OrderServiceTest
```

### Rule 3: Small, Deliberate Steps
One logical change per step. After each step, tests pass. Never get into a state where "I'll fix it all at once and run tests at the end."

```
Step 1: Extract method → tests green
Step 2: Move method to correct class → tests green
Step 3: Rename for clarity → tests green
Step 4: Remove original → tests green
```

---

## Algorithm Complexity Audit (Tip 45/46)

Before optimizing, identify whether an algorithm problem exists:

| Complexity | Description | Spot it by |
|------------|-------------|-----------|
| O(1) | Constant — no problem | Hash map lookups |
| O(log n) | Logarithmic — fine | Binary search |
| O(n) | Linear — usually fine | Single loop over collection |
| O(n log n) | Slightly worse — watch at scale | Sorting |
| O(n²) | Quadratic — **flag this** | Nested loops over same collection |
| O(2ⁿ) | Exponential — **critical** | Recursive without memoization |

**Scan for nested loops:**
```bash
# Find nested for loops in Java
grep -A5 "for.*:.*\|for.*int " src/ --include="*.java" | grep -B3 "for.*:.*\|for.*int "
```

**The test before investing time:**
> Is this algorithm actually a bottleneck? Profile first — don't optimize O(n²) on a collection of 10 items.

```python
import time
start = time.perf_counter()
result = suspect_function(data)
elapsed = time.perf_counter() - start
print(f"Elapsed: {elapsed:.3f}s for {len(data)} items")
```

---

## Refactoring Recipes

### Extract Method (most common)
When a function does more than one thing:

```python
# BEFORE — one function, two concerns
def process_order(order):
    if not order.items:
        raise ValueError("empty order")
    if order.total > 10000:
        raise ValueError("total exceeds limit")
    charge(order)
    send_confirmation(order)

# AFTER — extracted validation
def process_order(order):
    validate_order(order)
    charge(order)
    send_confirmation(order)

def validate_order(order):
    if not order.items:
        raise ValueError("empty order")
    if order.total > 10000:
        raise ValueError("total exceeds limit")
```

### Replace Magic Numbers with Constants
```java
// BEFORE
if (attempts > 3) { ... }
if (timeout > 5000) { ... }

// AFTER
private static final int MAX_RETRY_ATTEMPTS = 3;
private static final int REQUEST_TIMEOUT_MS = 5000;

if (attempts > MAX_RETRY_ATTEMPTS) { ... }
if (timeout > REQUEST_TIMEOUT_MS) { ... }
```

### Replace Nested Conditionals with Guard Clauses
```python
# BEFORE — deep nesting
def process(user):
    if user is not None:
        if user.is_active:
            if user.has_permission("write"):
                do_the_thing(user)

# AFTER — fail fast, happy path last
def process(user):
    if user is None: return
    if not user.is_active: return
    if not user.has_permission("write"): return
    do_the_thing(user)
```

---

## Refactoring Checklist

- [ ] Refactoring triggered by a concrete reason (DRY, orthogonality, testability, performance)
- [ ] Tests exist and pass before starting
- [ ] No new functionality added in the same commit
- [ ] Each step leaves the codebase in a working state
- [ ] Nested loops over large collections identified and profiled
- [ ] Magic numbers replaced with named constants
- [ ] Methods longer than ~30 lines reviewed for extraction opportunities (heuristic — adjust for your team's context)
- [ ] After refactoring, test coverage is equal or higher than before
