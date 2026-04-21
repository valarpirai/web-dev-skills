---
name: defensive-coding
description: Audit and enforce defensive coding practices — Design by Contract, assertions, crash-early, exception hygiene, and resource cleanup. Based on Pragmatic Programmer Ch.4.
origin: local
---

# Defensive Coding

Use this skill to audit code for missing contracts, silent failures, swallowed exceptions, and resource leaks. Based on The Pragmatic Programmer Tips 30–35.

## When to Use

- Reviewing a PR that adds new public APIs or service boundaries
- Auditing a module after a production incident involving silent data corruption
- Setting coding standards for a new team or project

---

## Core Principles

### Design by Contract (Tip 31)
Every function makes a contract with its callers:
- **Preconditions** — what must be true before the function runs
- **Postconditions** — what will be true after it returns
- **Invariants** — what stays true throughout

Write "lazy" code: strict about what you accept, minimal about what you promise.

### Crash Early (Tip 32)
A dead program does less damage than a crippled one.

If something impossible just happened, stop immediately. Don't limp forward with corrupt state.

### Use Assertions for Invariants (Tip 33)
Assertions document and enforce assumptions. They are not error handling — they guard against programmer mistakes.

---

## What to Audit

### 1. Missing Input Validation at Boundaries
Every public method, API endpoint, and service entry point must validate its inputs.

```java
// BAD — trusts caller completely
public Order createOrder(String userId, List<Item> items) {
    return orderRepo.save(new Order(userId, items));
}

// GOOD — explicit contract enforcement
public Order createOrder(String userId, List<Item> items) {
    if (userId == null || userId.isBlank()) throw new IllegalArgumentException("userId required");
    if (items == null || items.isEmpty()) throw new IllegalArgumentException("items must not be empty");
    return orderRepo.save(new Order(userId, items));
}
```

**Flag:** Public methods that accept nullable/unchecked parameters silently.

### 2. Swallowed Exceptions
```java
// BAD — exception hidden, program continues in unknown state
try {
    processPayment(order);
} catch (Exception e) {
    // TODO: handle this
}

// BAD — logged but execution continues as if nothing happened
} catch (Exception e) {
    log.error("error", e);
}

// GOOD — fail fast with context
} catch (PaymentException e) {
    throw new OrderProcessingException("Payment failed for order " + order.getId(), e);
}
```

**Flag:** Empty catch blocks, catch blocks that only log without re-throwing or returning a meaningful failure.

### 3. Assertions for Internal Invariants
Use assertions to document assumptions that must never be violated:

```python
def apply_discount(price, discount_pct):
    assert 0 <= discount_pct <= 100, f"discount_pct out of range: {discount_pct}"
    assert price >= 0, f"negative price: {price}"
    return price * (1 - discount_pct / 100)
```

**Flag:** Core algorithms with no assertions on intermediate values. Functions where "if this is ever negative, everything breaks" but no guard exists.

### 4. Resource Leaks
Resources must be released even when errors occur.

```java
// BAD — connection leaked on exception
Connection conn = dataSource.getConnection();
ResultSet rs = conn.createStatement().executeQuery(sql);
processResults(rs);  // if this throws, conn never closed
conn.close();

// GOOD — try-with-resources guarantees cleanup
try (Connection conn = dataSource.getConnection();
     ResultSet rs = conn.createStatement().executeQuery(sql)) {
    processResults(rs);
}
```

```python
# GOOD — context manager guarantees cleanup
with open("report.csv", "w") as f:
    write_report(f)
```

**Flag:** Manual `close()` / `release()` calls outside `finally` or `try-with-resources`. File handles, DB connections, or locks opened without guaranteed cleanup.

### 5. Exception Misuse for Control Flow
```java
// BAD — exception used as an if-else
try {
    User user = userRepo.findById(id);
    return user;
} catch (NotFoundException e) {
    return null;
}

// GOOD — return Optional, check explicitly
return userRepo.findById(id);  // returns Optional<User>
```

**Flag:** Catch blocks that return null or a default value when the absence of a result is expected — use `Optional` or nullable return types instead.

### 6. Null Propagation
```java
// BAD — NPE waiting to happen
String city = user.getAddress().getCity().toUpperCase();

// GOOD
String city = Optional.ofNullable(user)
    .map(User::getAddress)
    .map(Address::getCity)
    .map(String::toUpperCase)
    .orElse("UNKNOWN");
```

**Flag:** Chains of `.getX().getY().getZ()` without null checks.

---

## Severity Levels

| Severity | Pattern |
|----------|---------|
| **Critical** | Swallowed exceptions hiding data corruption or security failures |
| **Critical** | Resource leaks in high-throughput paths (DB connections, file handles) |
| **High** | No input validation at service or API boundaries |
| **High** | Null dereference chains in production paths |
| **Medium** | Missing assertions on algorithm invariants |
| **Low** | Exceptions used as control flow in non-critical paths |

---

## Defensive Coding Checklist

- [ ] Every public API/service entry validates all inputs before use
- [ ] No empty catch blocks (`catch (Exception e) {}`)
- [ ] All catch blocks either re-throw, wrap, or return a typed failure
- [ ] Resources (connections, files, locks) released in `finally` / `try-with-resources`
- [ ] Assertions guard internal algorithm invariants
- [ ] No null chains without Optional or explicit null checks
- [ ] Exceptions reserved for exceptional conditions — not expected absence of data
- [ ] Fail fast: corrupt/impossible state triggers immediate failure, not silent continuation
