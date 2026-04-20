---
name: twelve-factor-app
description: Audit and enforce The Twelve-Factor App methodology for SaaS backends. Checks compliance, flags violations, and suggests fixes per factor.
origin: local
---

# Twelve-Factor App

The Twelve-Factor App is a methodology for building scalable, maintainable SaaS applications. Use this skill to audit a codebase, flag violations, and apply fixes.

## When to Use

- Auditing a new or existing backend service for production-readiness
- Reviewing config, process, or deployment code for anti-patterns
- Onboarding a service to cloud-native or containerized infrastructure

## The 12 Factors

### I. Codebase
One codebase tracked in version control, many deploys.

**Violations to flag:**
- Multiple apps sharing the same repo without a monorepo strategy
- Code copied between services instead of extracted to a shared library

**Fix:** One repo per app. Shared code → versioned library.

---

### II. Dependencies
Explicitly declare and isolate all dependencies.

**Violations to flag:**
- Relying on system-level tools (curl, imagemagick) without declaring them
- Missing lockfiles (`package-lock.json`, `Gemfile.lock`, `go.sum`)
- `pip install` without `requirements.txt` or `pyproject.toml`

**Fix:** Use a dependency manifest + lockfile. Isolate via virtualenv, Docker, or devcontainers.

---

### III. Config
Store config in the environment, not in code.

**Violations to flag:**
- Hardcoded API keys, DB URLs, or credentials in source files
- Config values that differ per environment baked into code
- `.env` files committed to version control

**Fix:** Use environment variables. Load via `os.environ`, `process.env`, or a config library. Never commit `.env`.

```python
# BAD
DB_URL = "postgres://prod-host/mydb"

# GOOD
import os
DB_URL = os.environ["DATABASE_URL"]
```

---

### IV. Backing Services
Treat databases, queues, and caches as attached resources.

**Violations to flag:**
- Hardcoded `localhost` for DB/Redis/Kafka
- Service URLs not configurable via environment

**Fix:** All backing services accessed via URL from config. Swap without code changes.

```yaml
# docker-compose.yml — local dev
DATABASE_URL: postgres://localhost/myapp

# production env
DATABASE_URL: postgres://rds.amazonaws.com/myapp
```

---

### V. Build, Release, Run
Strictly separate build, release, and run stages.

**Violations to flag:**
- Config injected at build time (e.g., baked into Docker image)
- Code modified at runtime
- No distinct release artifact (image tag, version)

**Fix:**
- **Build**: compile code, bundle assets → immutable artifact
- **Release**: artifact + config → versioned release
- **Run**: execute release, no modifications

---

### VI. Processes
Execute the app as stateless processes.

**Violations to flag:**
- In-memory session storage (`express-session` with memory store)
- Local filesystem used for user uploads or state
- Sticky sessions required for correctness

**Fix:** Store sessions in Redis. Store files in S3/GCS. Any process instance should serve any request.

```js
// BAD
app.use(session({ store: new MemoryStore() }))

// GOOD
app.use(session({ store: new RedisStore({ client }) }))
```

---

### VII. Port Binding
Export services via port binding — app is self-contained.

**Violations to flag:**
- App requires external web server (Apache, Nginx) to function
- Port hardcoded instead of read from environment

**Fix:** App binds its own port from `$PORT`.

```python
port = int(os.environ.get("PORT", 8080))
app.run(host="0.0.0.0", port=port)
```

---

### VIII. Concurrency
Scale out via the process model.

**Violations to flag:**
- Spawning threads to handle scale instead of multiple processes
- Architecture that can't run multiple replicas (shared local state)

**Fix:** Design for horizontal scale. Use a process manager (Procfile, Kubernetes Deployment replicas).

```
# Procfile
web: gunicorn app:app
worker: celery -A tasks worker
```

---

### IX. Disposability
Fast startup, graceful shutdown.

**Violations to flag:**
- Long startup times (> 5s for web processes)
- No SIGTERM handler — in-flight requests dropped on shutdown
- Jobs not resumable after crash

**Fix:** Handle `SIGTERM` gracefully. Use job queues with at-least-once delivery.

```js
process.on('SIGTERM', () => {
  server.close(() => process.exit(0))
})
```

---

### X. Dev/Prod Parity
Keep dev, staging, and production as similar as possible.

**Violations to flag:**
- SQLite in dev, Postgres in prod
- Different OS, runtime version, or service versions across environments
- Long gap between commits and deploys

**Fix:** Use Docker Compose locally to mirror prod services. Deploy frequently.

---

### XI. Logs
Treat logs as event streams — write to stdout/stderr only.

**Violations to flag:**
- Writing logs to files (`logging.FileHandler`, `winston` file transport)
- Log rotation managed by the app
- Structured logging missing (plain strings instead of JSON)

**Fix:** Log to stdout in structured JSON. Let the platform (Kubernetes, CloudWatch) collect and route.

```python
import structlog
log = structlog.get_logger()
log.info("request_received", path="/api/users", status=200)
```

---

### XII. Admin Processes
Run admin/management tasks as one-off processes.

**Violations to flag:**
- DB migrations run automatically on app startup
- Admin scripts embedded in the main app process
- No way to run one-off tasks in the same environment as the app

**Fix:** Run migrations and admin scripts as separate one-off commands in the same release environment.

```bash
# Kubernetes one-off job
kubectl run migrate --image=myapp:v1.2.3 --restart=Never -- python manage.py migrate
```

---

## Audit Checklist

Run this against any backend service:

| # | Factor | Status |
|---|--------|--------|
| I | Single codebase, one repo per app | ☐ |
| II | Lockfile present, no system deps assumed | ☐ |
| III | No secrets or URLs in source code | ☐ |
| IV | All services accessed via env-configured URLs | ☐ |
| V | Build/release/run stages are distinct | ☐ |
| VI | No in-memory sessions or local file state | ☐ |
| VII | App binds port from `$PORT` env var | ☐ |
| VIII | Can run multiple replicas without conflict | ☐ |
| IX | Handles SIGTERM, fast startup | ☐ |
| X | Dev environment mirrors prod services | ☐ |
| XI | Logs go to stdout as structured JSON | ☐ |
| XII | Admin tasks run as one-off commands | ☐ |

## How to Audit a Codebase

1. Search for hardcoded secrets: `grep -r "password\|secret\|api_key" --include="*.py" src/`
2. Check for file-based sessions or local storage patterns
3. Read Dockerfile — confirm `CMD` doesn't run migrations inline
4. Check logging config — `FileHandler` or file transport = violation
5. Check startup code for `SIGTERM` handling
6. Review `docker-compose.yml` for dev/prod parity
