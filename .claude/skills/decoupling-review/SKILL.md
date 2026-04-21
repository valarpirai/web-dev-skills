---
name: decoupling-review
description: Review code for coupling violations — Law of Demeter breaches, hardcoded config, temporal dependencies, and missing MVC separation. Based on Pragmatic Programmer Ch.5.
origin: local
---

# Decoupling Review

Use this skill to find and fix excessive coupling between modules. Based on The Pragmatic Programmer Tips 36–43.

## When to Use

- Reviewing a PR that adds a new dependency between modules
- Diagnosing why a "simple" change broke multiple unrelated parts of the system
- Designing a new feature and evaluating component boundaries

---

## Core Principles

### Law of Demeter (Tip 36)
A method should only call methods on:
- Itself
- Objects passed as parameters
- Objects it creates
- Its own directly held components

**Never navigate through an object to reach its internals.**

### Configure, Don't Integrate (Tip 37/38)
Behavior that varies should live in metadata (config, database, environment), not compiled code.

### Design for Concurrency (Tip 41)
Avoid time-based dependencies between components. Each component should be able to run at its own pace.

---

## What to Audit

### 1. Law of Demeter Violations (Method Chains)
```java
// BAD — navigates through book → pages → last → text
String text = book.getPages().getLast().getText().toUpperCase();

// GOOD — ask the object for what you need
String text = book.getLastPageText().toUpperCase();
```

```python
# BAD
discount = order.getCustomer().getLoyaltyAccount().getDiscountRate()

# GOOD
discount = order.getApplicableDiscountRate()
```

**Scan for:**
```bash
# Language-agnostic — adjust pattern for your syntax
grep -rn "\.\w\+()\..*\.\w\+()" src/  # chains of 3+ method calls
```

**Flag:** Any chain of 3+ `.get()` calls navigating through domain objects.

### 2. Hardcoded Values That Should Be Config
```java
// BAD — tightly coupled to infrastructure
private static final String DB_URL = "jdbc:postgresql://prod-db:5432/orders";
private static final int TIMEOUT_MS = 5000;
private static final String FEATURE_FLAG = "v2_checkout";

// GOOD — driven by metadata
String dbUrl = config.get("database.url");
int timeoutMs = config.getInt("http.timeout.ms", 3000);
boolean v2Checkout = featureFlags.isEnabled("v2_checkout");
```

**Scan for:**
```bash
# First grep is language-agnostic; second is Java — adjust for your language
grep -rn "localhost\|127\.0\.0\.1\|\.amazonaws\.com\|\.internal" src/
grep -rn "private static final String.*=.*\"" src/ --include="*.java"
```

**Flag:** Hostnames, ports, URLs, feature names, or timeouts hardcoded in source.

### 3. Missing View/Model Separation
```java
// BAD — domain model formats its own response
public class Order {
    public String toJson() { ... }      // model knows about transport format
    public String toHtmlRow() { ... }   // model knows about UI
}

// GOOD — separated concerns
public class OrderJsonSerializer { ... }
public class OrderRowRenderer { ... }
```

**Flag:** Domain model classes with `toJson()`, `toXml()`, `toHtml()`, or UI rendering methods.

### 4. Temporal Coupling in Sequential Code
Temporal coupling = component A must run before component B, but nothing enforces this.

```java
// BAD — order matters, nothing enforces it
emailService.setRecipient(user.email);  // must be called first
emailService.setSubject("Welcome");     // then this
emailService.send();                    // then this — but nothing stops wrong order

// GOOD — all data provided atomically
emailService.send(new Email(user.email, "Welcome", body));
```

**Flag:** Setter-based APIs where callers must call methods in a specific order before calling a final "execute" method.

### 5. Global / Static Mutable State
```java
// BAD — hidden coupling via static state
public class RequestContext {
    public static User currentUser;         // any code can read/corrupt this
    public static String currentTenantId;
}

// GOOD — pass context explicitly
public OrderService(RequestContext ctx) { this.ctx = ctx; }
```

**Scan for:**
```bash
# Java example — adjust for your language (Python: module-level mutable dicts/lists)
grep -rn "static.*=\|static final.*Map\|static.*List" src/ --include="*.java"
```

**Flag:** Static mutable fields shared across requests or threads.

---

## Coupling Severity Levels

| Severity | Pattern |
|----------|---------|
| **Critical** | Global mutable state shared across threads/requests |
| **High** | Domain model containing serialization or rendering logic |
| **High** | Law of Demeter chains navigating 3+ levels deep |
| **Medium** | Hardcoded infrastructure values (URLs, timeouts, ports) |
| **Medium** | Temporal coupling in public APIs (ordered-setter pattern) |
| **Low** | Service calling 3+ peers inline instead of publishing an event |

---

## Decoupling Checklist

- [ ] No method chains navigating 3+ levels through domain objects
- [ ] All infrastructure config (URLs, timeouts, credentials) read from environment/config
- [ ] Domain models contain no serialization, rendering, or transport logic
- [ ] No static mutable fields shared across requests
- [ ] APIs that require ordered calls replaced with builder or atomic parameter passing
- [ ] Each module can be tested in isolation without initializing the whole app
