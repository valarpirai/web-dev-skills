---
name: event-driven-architecture
description: Design and implement event-driven systems using message queues, Kafka, CQRS, and async patterns. Covers producer/consumer design, event schema, and failure handling.
origin: local
---

# Event-Driven Architecture

Use this skill when decoupling services, handling async workflows, or designing systems that react to state changes.

## When to Use

- Services need to communicate without tight coupling
- Operations can be processed asynchronously (emails, notifications, reports)
- Audit trail or event replay is required
- Multiple consumers need the same event (fan-out)

---

## Core Concepts

### Events vs Commands vs Queries
| Type | Intent | Example |
|------|--------|---------|
| **Event** | Something happened (past tense) | `OrderPlaced`, `UserSignedUp` |
| **Command** | Do something (imperative) | `SendEmail`, `ProcessPayment` |
| **Query** | Read state | Request/response, not async |

**Rule:** Events are facts — they cannot be rejected. Commands can fail.

---

## Message Queue Patterns

### Point-to-Point (Queue)
One producer, one consumer. Each message processed once.
- Use for: task queues, job workers, email sending

```
Producer → [Queue] → Consumer
```

### Pub/Sub (Topic)
One producer, many consumers. Each subscriber gets a copy.
- Use for: notifications, cache invalidation, fan-out

```
Producer → [Topic] → Consumer A
                   → Consumer B
                   → Consumer C
```

### Tools
| Tool | Best For |
|------|----------|
| **Kafka** | High-throughput, ordered, replayable event log |
| **RabbitMQ** | Task queues, routing, low latency |
| **SQS** | Simple managed queues on AWS |
| **Redis Streams** | Lightweight pub/sub with consumer groups |

---

## Kafka Patterns

### Topic Design
- One topic per event type: `orders.placed`, `payments.completed`
- Partition by entity key (e.g., `order_id`) to preserve ordering per entity
- Set retention based on replay needs (7 days default)

### Producer
```python
producer.produce(
    topic="orders.placed",
    key=str(order_id),        # ensures ordering per order
    value=json.dumps(event),
    headers={"event_version": "1"}
)
```

### Consumer with at-least-once delivery
```python
consumer.subscribe(["orders.placed"])
while True:
    msg = consumer.poll(1.0)
    if msg:
        process(msg)
        consumer.commit()     # commit only after successful processing
```

### Dead Letter Queue (DLQ)
Route failed messages to a DLQ after N retries:
```
[orders.placed] → Consumer → fails 3x → [orders.placed.dlq]
```

---

## CQRS (Command Query Responsibility Segregation)

Separate the write model (commands) from the read model (queries).

```
HTTP POST /orders
  → OrderService (write)
    → persists to orders DB
    → emits OrderPlaced event
      → ReadModelProjector
        → updates orders_view table (optimized for queries)

HTTP GET /orders/:id
  → queries orders_view (fast, denormalized)
```

**When to use:** Read and write patterns differ significantly; heavy read load needs denormalized views.

---

## Event Sourcing

Store all state changes as an immutable event log. Current state = replay of events.

```
events table:
  order_id | event_type    | payload              | timestamp
  123      | OrderCreated  | {items, customer}    | 2024-01-01
  123      | ItemAdded     | {item_id: 42}        | 2024-01-02
  123      | OrderShipped  | {tracking: "XYZ"}    | 2024-01-03

Current state of order 123 = replay of all 3 events
```

**Benefits:** Full audit trail, event replay, temporal queries.
**Cost:** Complexity, eventual consistency, snapshot management.

---

## Failure Handling

### Idempotency
Consumers must handle duplicate delivery. Use an idempotency key:
```python
if already_processed(event_id):
    return  # skip, already handled
process(event)
mark_processed(event_id)
```

### Outbox Pattern
Avoid dual-write problems (DB write + event publish in separate transactions):
```sql
BEGIN;
INSERT INTO orders ...;
INSERT INTO outbox (event_type, payload) VALUES ('OrderPlaced', ...);
COMMIT;
-- separate process polls outbox and publishes to Kafka
```

### Saga Pattern
Coordinate multi-step transactions across services using compensating events:
```
OrderService: OrderPlaced →
PaymentService: PaymentCharged →
InventoryService: StockReserved →
  (if fails) → StockReservationFailed →
PaymentService: PaymentRefunded →
OrderService: OrderCancelled
```

---

## Design Checklist

- [ ] Events named in past tense and represent facts
- [ ] Event schema versioned (add fields, never remove)
- [ ] Consumers are idempotent
- [ ] DLQ configured for failed messages
- [ ] Outbox pattern used for transactional publish
- [ ] Partition key chosen to preserve per-entity ordering
- [ ] Retention period set based on replay requirements
