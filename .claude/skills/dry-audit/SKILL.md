---
name: dry-audit
description: Audit a codebase for DRY violations, orthogonality issues, and unnecessary duplication. Flags repeated logic, shared mutable state, and non-orthogonal components.
origin: local
---

# DRY Audit

Use this skill to scan a codebase for Don't-Repeat-Yourself violations and orthogonality problems. Based on The Pragmatic Programmer Tips 11–13.

## When to Use

- Before a refactoring sprint to identify the worst duplication hotspots
- Reviewing a PR that adds logic already present elsewhere
- Onboarding to a new codebase to understand its hygiene baseline

---

## Core Concepts

### DRY — Don't Repeat Yourself (Tip 11)
Every piece of knowledge must have a single, unambiguous, authoritative representation within a system.

The four types of duplication to hunt:

| Type | Description | Example |
|------|-------------|---------|
| **Imposed** | Environment forces it | Two services maintaining the same DB schema |
| **Inadvertent** | Developer unaware | Same validation logic in controller and service |
| **Impatient** | Laziness ("it's faster") | Copy-pasted error handler |
| **Inter-developer** | Different teams duplicating | Two teams writing the same HTTP client |

### Orthogonality (Tip 13)
Two components are orthogonal if changing one does not require changing the other.

**Signs of poor orthogonality:**
- A "simple" change ripples through many unrelated files
- Developers afraid to touch a module because "it might break X"
- Unit test setup requires 5+ unrelated dependencies to initialize

---

## What to Audit

### 1. Duplicated Business Logic
Search for identical or near-identical conditional logic:

```bash
# Java example — adjust --include for your language
grep -rn "if.*email.*@" src/ --include="*.java"
grep -rn "if.*email.*@" src/ --include="*.java" | wc -l
```

**Flag:** Same validation, parsing, or calculation appearing in 2+ places.

**Fix:** Extract to a shared utility/service class.

### 2. Copy-Pasted Code Blocks
Look for structurally similar methods doing the same thing:

```bash
# Find suspiciously similar method names
grep -rn "def validate_\|def check_\|def verify_" src/
```

**Flag:** `validateUser()`, `validateAdmin()`, `validateGuest()` all containing overlapping checks.

**Fix:** Parameterize — `validate(entity, role)`.

### 3. Schema / Model Duplication
Check if database schema definitions are mirrored in code:

```bash
# Java example — adjust field syntax for your language
grep -rn "private String name" src/
grep -rn "String name" src/dto/
```

**Flag:** Same fields declared in Entity, DTO, and API response object with manual mapping between each.

**Fix:** Use a mapping layer (MapStruct, AutoMapper) or collapse where appropriate.

### 4. Config / Constants Duplication
```bash
grep -rn '"localhost:5432"\|"5432"\|"postgres"' src/
```

**Flag:** Connection strings, magic numbers, or URLs appearing in 2+ places.

**Fix:** Single config source — environment variable or constants file.

### 5. Test Setup Duplication
```bash
# Java example — adjust constructor patterns for your language
grep -rn "new UserService\|new OrderService" src/test/
```

**Flag:** Same object construction repeated in every test class.

**Fix:** Shared test fixtures or a builder pattern.

## Severity Levels

| Severity | Pattern |
|----------|---------|
| **Critical** | Business rules duplicated across services/modules — bugs must be fixed N times |
| **High** | Same validation or parsing in 2+ layers of the same app |
| **Medium** | Structural duplication (copy-paste) with minor variations |
| **Low** | Repeated constants or config values in non-critical paths |

---

## DRY Audit Checklist

- [ ] Business logic lives in exactly one layer (not controller + service + util)
- [ ] No copy-pasted validation or parsing blocks
- [ ] Configuration values (URLs, ports, timeouts) defined once
- [ ] No duplicate field declarations across entity / DTO / response model
- [ ] Shared test fixtures used instead of repeated setup code
- [ ] A change to a core domain concept touches ≤ 2 files
- [ ] No `validateFoo()` / `validateBar()` doing the same check for different types
