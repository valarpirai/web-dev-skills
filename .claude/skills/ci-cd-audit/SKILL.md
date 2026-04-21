---
name: ci-cd-audit
description: Audit a project's build and test automation — single-command builds, test pyramid coverage, nightly builds, regression policy, and "find bugs once" discipline.
origin: local
---

# CI/CD Audit

Use this skill to audit a project's build, test, and deployment automation. Based on The Pragmatic Programmer Tips 61–66 (Ubiquitous Automation and Ruthless Testing).

## When to Use

- Evaluating a project's CI/CD pipeline for gaps
- Setting up automation for a new project
- After a production bug that existing tests didn't catch

---

## Core Principles

**Tip 61: Don't Use Manual Procedures.** If it's done by hand, it will eventually be done wrong.

**Tip 62: Test Early. Test Often. Test Automatically.** Tests that run with every build are more effective than test plans on a shelf.

**Tip 63: Coding Ain't Done Til All the Tests Run.** No exceptions.

**Tip 66: Find Bugs Once.** Once a human finds a bug, a test must find it from then on.

---

## What to Audit

### 1. Single-Command Build
The entire project must build, test, and produce a deployable artifact with one command.

```bash
# Java/Maven — should work from a clean checkout
./mvnw clean package

# Node
npm ci && npm run build && npm test

# Python
pip install -r requirements.txt && python -m pytest
```

**Flag:** Any build process that requires:
- Manual steps documented in a README that developers must follow
- Environment setup that isn't scripted
- Secrets or config files that must be manually placed before building

**Check:**
```bash
# Can a fresh clone build and test without extra steps?
git clone <repo> /tmp/fresh-clone && cd /tmp/fresh-clone && ./mvnw test
```

### 2. Test Pyramid Coverage
A healthy test suite has a pyramid shape: many unit tests, fewer integration tests, fewest E2E tests.

| Layer | What it tests | Speed | Cost |
|-------|--------------|-------|------|
| **Unit** | Single class/function in isolation | Milliseconds | Cheap |
| **Integration** | Multiple components together (real DB, real queue) | Seconds | Medium |
| **E2E / API** | Full system from outside | Seconds–minutes | Expensive |
| **Performance** | Throughput, latency under load | Minutes | Expensive |

**Check ratio:**
```bash
# Java/Maven example — adjust naming conventions for your stack
find src/test -name "*Test.java" | wc -l         # unit
find src/test -name "*IT.java" | wc -l           # integration
find src/test -name "*E2ETest.java" | wc -l      # e2e
```

**Flag:**
- No unit tests — only integration tests (slow, fragile suite)
- No integration tests — unit tests that mock everything (false confidence)
- E2E tests as the primary test layer (extremely slow, breaks often)

### 3. Tests Run on Every Commit (CI)
Every push must trigger automated tests.

**Check for CI config:**
```bash
ls .github/workflows/   # GitHub Actions
ls .gitlab-ci.yml       # GitLab CI
ls Jenkinsfile          # Jenkins
ls .circleci/config.yml # CircleCI
```

**A minimal CI pipeline must:**
1. Check out code
2. Install dependencies
3. Build
4. Run all tests
5. Fail the build if any test fails

**Flag:** CI pipelines that:
- Run only a subset of tests to "save time"
- Skip tests on certain branches
- Allow merging when tests fail (no branch protection)

### 4. Regression Policy: Find Bugs Once
Every bug that reaches humans should be trapped by a test after it's fixed.

**Check if bugs have corresponding tests:**
```bash
# Search git log for bug fixes without test additions
git log --oneline --name-only | grep -A5 "fix\|bug"
```

**The rule:** When fixing a bug:
1. Write a test that reproduces the bug (it fails)
2. Fix the bug
3. Verify the test now passes
4. Commit test + fix together

**Flag:** Bug fix commits that contain no new test files.

### 5. Saboteur Testing (Tip 64)
Periodically verify that your tests actually catch bugs.

**Mutation testing tools:**
```bash
# Java
./mvnw org.pitest:pitest-maven:mutationCoverage

# Python
pip install mutmut && mutmut run

# JavaScript
npx stryker run
```

**What to look for:** Mutation score < 70% means tests exist but don't actually verify behavior.

### 6. Test State Coverage, Not Code Coverage (Tip 65)
Line coverage of 80% doesn't mean 80% of behavior is tested.

```java
// 100% line coverage but only 1 of 3 meaningful states tested:
public String classify(int score) {
    if (score >= 90) return "A";           // tested
    else if (score >= 70) return "B";      // not tested
    else return "C";                       // not tested
}
```

**Meaningful states to test for any function:**
- Happy path
- Boundary values (0, max, exactly-at-threshold)
- Empty/null inputs
- Error conditions

---

## CI/CD Pipeline Template

A complete pipeline for a Java/Maven backend service. Adapt `setup-java` and `./mvnw` for your stack (e.g. `setup-node`/`npm test`, `setup-python`/`pytest`).

```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Set up JDK
        uses: actions/setup-java@v4
        with:
          java-version: '21'
      - name: Build and test
        run: ./mvnw clean verify
      - name: Upload test report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: target/surefire-reports/
```

---

## CI/CD Audit Checklist

- [ ] Project builds and tests with a single command from a clean checkout
- [ ] CI runs on every push to every branch
- [ ] Build fails when any test fails — no bypass mechanism
- [ ] Test pyramid is healthy: more unit than integration, more integration than E2E
- [ ] Every bug fix is accompanied by a regression test
- [ ] Test coverage measures state coverage, not just line coverage
- [ ] Mutation testing score ≥ 70% (or mutation testing is in the plan)
- [ ] Performance tests exist and run at minimum weekly
- [ ] No manual steps required for deployment to any environment
- [ ] Build artifacts are versioned and immutable
