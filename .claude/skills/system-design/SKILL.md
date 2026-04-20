---
name: system-design
description: Guide scalable system design decisions — capacity planning, scalability patterns, and architectural trade-offs for backend services.
origin: local
---

# System Design

Use this skill when designing new systems, reviewing architecture, or making scalability decisions.

## When to Use

- Designing a new backend service from scratch
- Reviewing an existing system for scalability bottlenecks
- Making trade-off decisions between architectural approaches

---

## Core Principles

### Start with Requirements
Before any design decision, clarify:
- **Scale**: requests/sec, data volume, user count
- **Latency**: p99 acceptable latency per endpoint
- **Consistency**: strong vs eventual — what breaks if data is stale?
- **Availability**: 99.9% (8.7h downtime/yr) vs 99.99% (52m/yr)

### CAP Theorem
A distributed system can guarantee only 2 of 3:

| Guarantee | Meaning |
|-----------|---------|
| Consistency | Every read returns the latest write |
| Availability | Every request gets a response |
| Partition Tolerance | System works despite network splits |

**In practice:** partition tolerance is mandatory. Choose CP (banks, inventory) or AP (social feeds, analytics).

---

## Scalability Patterns

### Vertical vs Horizontal Scaling
- **Vertical**: bigger machine — simple but has a ceiling
- **Horizontal**: more machines — preferred for stateless services

### Read-Heavy Systems
1. Add read replicas to the database
2. Cache frequently read data (Redis, Memcached)
3. Use a CDN for static and semi-static content
4. Denormalize data for read performance

### Write-Heavy Systems
1. Async writes via message queue (decouple producer from DB)
2. Batch writes with buffering
3. Sharding — partition data by key (user_id, region)
4. Use append-only stores (event log, time-series DB)

### Stateless Services
Design every service instance to be interchangeable:
- No local session state — use Redis
- No local file storage — use S3/GCS
- Configuration from environment, not disk

---

## Capacity Planning

### Back-of-Envelope Formula
```
RPS = DAU × requests_per_user_per_day / 86400
Storage = DAU × data_per_user_per_day × retention_days
Bandwidth = RPS × avg_response_size
```

### Example: URL Shortener
- 100M DAU, 1 write per 10 users per day → ~115 writes/sec
- 10 reads per write → 1,150 reads/sec
- URL record = 500 bytes → 5 years storage ≈ 1 TB

---

## Common Architectural Trade-offs

| Decision | Option A | Option B | Choose A when |
|----------|----------|----------|---------------|
| Sync vs Async | REST call | Message queue | A: needs immediate response; B: can tolerate delay |
| SQL vs NoSQL | PostgreSQL | MongoDB/DynamoDB | A: relations + ACID; B: flexible schema, massive scale |
| Monolith vs Microservices | Monolith | Microservices | A: small team, early stage; B: independent scaling needed |
| Push vs Pull | WebSocket/SSE | Polling | A: real-time, low latency; B: simple clients, infrequent updates |

---

## Design Review Checklist

- [ ] Single points of failure identified and mitigated
- [ ] Database indexed for primary query patterns
- [ ] Caching layer defined with invalidation strategy
- [ ] Async processing for non-critical paths
- [ ] Rate limiting on public-facing endpoints
- [ ] Data partitioning strategy for growth
- [ ] Failure modes documented (what happens when service X is down)
