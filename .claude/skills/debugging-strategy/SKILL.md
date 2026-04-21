---
name: debugging-strategy
description: Structured debugging playbook — reproduce first, prove assumptions, trace systematically, and eliminate false causes. Based on Pragmatic Programmer Ch.3 debugging tips.
origin: local
---

# Debugging Strategy

A systematic approach to finding root causes rather than symptoms. Based on The Pragmatic Programmer Tips 24–27 and the Debugging Checklist.

## When to Use

- Stuck on a bug that keeps reappearing or can't be reproduced reliably
- Investigating a production incident where the root cause is unclear
- Reviewing a bug fix to verify it addresses the root cause, not the symptom

---

## Core Principles

**Tip 24: Fix the Problem, Not the Blame.** It doesn't matter whose fault it is — it needs to be fixed.

**Tip 25: Don't Panic.** Resist the urge to start changing things randomly. Think first.

**Tip 26: "select" Isn't Broken.** The bug is almost certainly in your application, not the OS, compiler, or framework.

**Tip 27: Don't Assume It — Prove It.** Every assumption must be verified with real data.

---

## The Debugging Process

### Step 1: Reproduce with a Single Command
The most important step. You cannot reliably fix what you cannot reliably reproduce.

```bash
# Good: reproducible with one command
curl -X POST http://localhost:8080/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": "u123", "items": []}'

# Good: a failing test
./mvnw test -Dtest=OrderServiceTest#createOrder_emptyItems_shouldFail
```

If you can't reproduce it:
- Add logging to capture the exact inputs that triggered the failure
- Check if it's environment-specific (data, config, load)
- Check for race conditions — reproduce under concurrent load

### Step 2: Address Compiler Warnings First (statically typed languages)
Before debugging logic, eliminate all compiler warnings. They are often pointing directly at the bug. For dynamically typed languages (Python, Ruby, JS), run your linter instead.

```bash
# Java
./mvnw compile 2>&1 | grep -i "warning\|deprecated\|unchecked"

# TypeScript
npx tsc --noEmit

# Python (linter equivalent)
python -W all -m py_compile mymodule.py
flake8 mymodule.py
```

### Step 3: Visualize Data at Failure Point
Don't guess at what the data looks like — print it.

```python
# Add a checkpoint right before the failure
print(f"DEBUG order state: {vars(order)}")
print(f"DEBUG items: {[vars(i) for i in items]}")
```

Use a debugger to inspect live state rather than adding print statements:
- Set a breakpoint at the last known-good point
- Step forward and watch which value becomes wrong
- Identify the exact line where an invariant breaks

### Step 4: Trace Before and After
For bugs in data transformations, log the state entering and leaving each stage:

```java
log.debug("BEFORE transform: {}", input);
Result result = transform(input);
log.debug("AFTER transform: {}", result);
```

If a pipeline has 5 stages, binary-search the bug: log stage 3. If good, the bug is in stage 4 or 5. Log stage 4. Narrow down in O(log n) steps.

### Step 5: Rubber Duck the Problem
Explain the bug out loud to a colleague or rubber duck. The act of articulating the problem forces you to examine your assumptions. Often the bug reveals itself mid-explanation.

Describe:
1. What you expected to happen
2. What actually happened
3. What you have already tried
4. What you are currently assuming

### Step 6: Process of Elimination
Start with what's most likely wrong (your code), then work outward.

```
Your code → your dependencies → framework → OS/runtime → hardware
```

Only move to the next layer after exhausting the previous one. The bug is almost always in your code.

When a test passes but the live system fails:
- Check for environment differences (config, data, load, OS)
- Check for time-dependent behavior
- Check for state left over from a previous test run

---

## Debugging Checklist

Run through this for each bug investigation:

- [ ] Is the reported symptom the actual bug, or just a consequence of a deeper bug?
- [ ] Can I reproduce it with a single command or failing test?
- [ ] Have I checked all compiler warnings?
- [ ] Have I confirmed the exact input values at the point of failure (not assumed them)?
- [ ] Is the bug in my code, or am I blaming the framework/OS prematurely?
- [ ] Have I explained the bug to someone else (or a rubber duck)?
- [ ] Does the same condition that caused this bug exist elsewhere in the codebase?
- [ ] After fixing, does a test exist to prevent regression?

---

## Common Patterns and Fixes

| Symptom | Likely Cause | Where to Look |
|---------|-------------|---------------|
| Works locally, fails in CI | Environment difference | Config, env vars, timezone, locale |
| Works first time, fails on retry | State not reset between runs | Static fields, caches, DB state |
| Fails only under load | Race condition or connection pool exhaustion | Shared mutable state, thread safety |
| Fails for some users, not others | Data-specific edge case | Null/empty/special characters in user data |
| Fix worked, then broke again | Symptom fixed, not root cause | Re-examine what the real invariant is |
| Intermittent failure | Non-determinism | Random seeds, timing, network flakiness |

---

## After the Fix

**Tip 66: Find Bugs Once.** Once a human finds a bug, a test should find it from then on.

After every bug fix:
1. Write a test that reproduces the original failure
2. Verify the test fails before your fix, passes after
3. Commit the test alongside the fix
4. Check if the same root cause exists elsewhere — search the codebase for the same pattern
