---
name: doc-improver
description: Analyse a project's docs/ directory and CLAUDE.md. Identify what can be improved so Claude Code works better, apply changes, then review the result.
origin: local
---

# Doc Improver

Analyse documentation in a project and improve it so Claude Code can work more effectively — clearer context, better commands, less ambiguity.

## When to Use

- Starting work on an unfamiliar project
- CLAUDE.md is missing, thin, or outdated
- Claude keeps asking questions that docs should already answer
- Docs/ directory exists but content is vague or stale

---

## Step 1 — Analyse

Run these checks against the project:

### Check CLAUDE.md

```bash
cat CLAUDE.md 2>/dev/null || echo "CLAUDE.md missing"
```

Look for:
- [ ] Project purpose — what does this app do?
- [ ] Tech stack listed (language, framework, DB, infra)
- [ ] How to run the project locally (exact commands)
- [ ] How to run tests
- [ ] How to build / deploy
- [ ] Folder structure explained
- [ ] Coding conventions (naming, patterns to follow)
- [ ] Common tasks (migrations, code gen, seeding)
- [ ] What NOT to do (pitfalls, off-limits files)

### Check docs/ Directory

```bash
find docs/ -name "*.md" | sort
```

Look for:
- [ ] Are docs up-to-date with the current codebase?
- [ ] Do code examples match real file paths?
- [ ] Are commands copy-pasteable (no placeholder-only examples)?
- [ ] Is the architecture documented with a diagram or description?
- [ ] Are ADRs (Architecture Decision Records) present for major choices?

---

## Step 2 — Improvement Patterns

### Pattern 1: Thin or Missing CLAUDE.md
**Problem:** Claude has no project context — asks redundant questions, guesses conventions.

**Fix:** Create or expand CLAUDE.md with this structure:

```markdown
# <Project Name>

## What This Is
One paragraph: purpose, primary users, core features.

## Tech Stack
- Language: Python 3.12
- Framework: FastAPI
- Database: PostgreSQL 15
- Queue: Kafka
- Infra: Kubernetes on GCP

## Local Setup
```bash
cp .env.example .env
docker compose up -d
make migrate
make dev
```

## Running Tests
```bash
make test          # unit tests
make test-e2e      # end-to-end
```

## Project Structure
```
src/
├── api/        # HTTP route handlers
├── services/   # Business logic
├── models/     # DB models (SQLAlchemy)
├── workers/    # Kafka consumers
└── tests/
```

## Conventions
- Services are stateless — no instance variables
- All DB access goes through the repository layer in `src/repos/`
- Use `make fmt` before committing

## Do Not
- Do not add business logic to route handlers
- Do not commit .env files
```

---

### Pattern 2: Stale Code Examples
**Problem:** Docs show old API signatures or file paths that no longer exist.

**Fix:** Cross-reference each code example against the actual source:
```bash
# verify a referenced function still exists
grep -r "function_name_from_docs" src/
```
Update or remove examples that don't match.

---

### Pattern 3: Vague Setup Instructions
**Problem:** "Install dependencies and run the app" — Claude can't infer the right commands.

**Fix:** Replace every vague step with exact commands:
```markdown
# BAD
Install dependencies and start the server.

# GOOD
```bash
npm install
cp .env.example .env
npm run db:migrate
npm run dev        # starts on http://localhost:3000
```

---

### Pattern 4: Missing Architecture Context
**Problem:** Claude doesn't know how services connect — makes incorrect assumptions.

**Fix:** Add a brief architecture section to CLAUDE.md or `docs/architecture.md`:

```markdown
## Architecture

Request flow:
  Client → API Gateway → AuthService (JWT validation)
         → OrderService → PostgreSQL
                        → Kafka (order.placed event)
                          → NotificationService (email)
                          → InventoryService (reserve stock)
```

---

### Pattern 5: No ADRs for Key Decisions
**Problem:** Claude doesn't know *why* a pattern was chosen and may suggest reverting it.

**Fix:** Add `docs/adr/` with records for major decisions:

```markdown
# ADR-001: Use Outbox Pattern for Event Publishing

## Status: Accepted

## Context
We need to publish Kafka events atomically with DB writes.

## Decision
Use the transactional outbox pattern. A poller reads the outbox table and publishes.

## Consequences
- No dual-write race conditions
- Adds a poller process to maintain
```

---

## Step 3 — Review After Changes

After editing docs, verify:

```bash
# No broken file path references
grep -r "src/" docs/ CLAUDE.md | grep -v ".git"

# All commands in docs are valid
grep -E '```(bash|sh)' -A5 docs/**/*.md CLAUDE.md
```

Ask these questions:
- [ ] Can a new engineer run the project using only CLAUDE.md?
- [ ] Does every code example in docs/ match the real code?
- [ ] Are all major architectural decisions explained with rationale?
- [ ] Is there anything Claude would repeatedly ask that docs should answer?

---

## Quick-Start Prompt

To use this skill on a project, run:

> Analyse the `docs/` directory and `CLAUDE.md` in this project. List what is missing or unclear for Claude Code to work effectively. Then apply improvements following the doc-improver patterns. After changes, review that all examples are accurate and commands work.
