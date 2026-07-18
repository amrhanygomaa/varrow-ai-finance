# ADR-003: Event Bus Design

| | |
|---|---|
| **ADR ID** | ADR-003 |
| **Title** | Event Bus Design for Modular Monolith |
| **Status** | Proposed |
| **Author** | Architecture Team |
| **Date** | 2026-07-17 |
| **Project** | VArrow AI Finance |
| **Company** | VArrow.tech |

---

## Context

VArrow AI Finance is designed as a **Modular Monolith** per [ADR-001](ADR-001-technology-stack.md) using Node.js, NestJS, and PostgreSQL on AWS. The [System Architecture](../05-architecture/SYSTEM_ARCHITECTURE.md) defines 10 bounded contexts that interact heavily via domain events (e.g., `SalesInvoiceIssued`, `QuotationApproved`).

A critical requirement is defining how these events are published, transported, and consumed internally between modules. The chosen mechanism must provide reliability, maintain decoupling between bounded contexts, support a future migration path to microservices, and operate within the constraints of a startup focusing on low operational cost (AWS Free Tier-friendly).

---

## Decision

We will implement an **internal event bus** using **NestJS EventEmitter2** for synchronous in-memory routing, combined with the **Transactional Outbox Pattern** (backed by PostgreSQL) for reliable, asynchronous event delivery.

### 1. Architecture Overview

- **Domain Model:** Emits events strictly within the bounds of a transaction.
- **Transactional Outbox:** Events are persisted to an `outbox_events` table in PostgreSQL *in the same transaction* as the domain state changes.
- **Relay Mechanism:** A background worker polls the outbox table to publish events asynchronously.
- **EventEmitter2:** The relay uses NestJS's `EventEmitter2` to route published events to internal subscribers within the monolith.
- **Subscribers:** Other bounded contexts subscribe to events using `@OnEvent()` decorators and process them independently.

### 2. The Transactional Outbox Pattern

**Why the Outbox Pattern?**
In a financial system, dual-write failures (e.g., saving an invoice to the database but failing to emit the `SalesInvoiceIssued` event to the broker) are unacceptable. The Outbox Pattern guarantees at-least-once delivery by writing the event to the database within the same ACID transaction as the business entity update.

**Outbox Flow:**
1. A domain service (e.g., `SalesService`) starts a database transaction.
2. The service saves business data (e.g., `sales_invoices` table).
3. The service writes the domain event payload to the `outbox_events` table.
4. The transaction commits.
5. An asynchronous background process (Relay) reads unpublished events from `outbox_events`.
6. The Relay publishes the event via `EventEmitter2`.
7. Once successfully dispatched (or acknowledged by subscribers), the Relay marks the outbox event as `published`.

**Outbox Retention & Immutability Strategy:**
- **Audit Logs Immutability**: The `administration.audit_logs` table is strictly append-only. Updates and deletions are blocked globally via database triggers.
- **Outbox Mutability Rules**: To support the retry and tracking lifecycle, the `infrastructure.outbox_events` table allows updates **ONLY** for the following fields: `published`, `retry_count`, `next_retry_at`, `last_error`, and `dead_letter_at`. All other columns (e.g. `id`, `event_name`, `payload`, `created_at`) are immutable. Deletions are blocked by database triggers, and old events are moved to an archive database via a scheduled archival job after a configurable retention period.
- **Never permanently delete financial events automatically.**

### 3. Relay Strategy & Concurrency Control

**V1 Implementation: Polling with Distributed Locking**
- The outbox relay will poll the `outbox_events` table every 2–5 seconds for unpublished events.
- **Concurrency Control in Multi-Instance Deployments (ECS Fargate)**: To prevent multiple application instances from processing the same event, the outbox worker executes a lock-free distributed polling query using PostgreSQL's concurrency features:
  ```sql
  SELECT * FROM infrastructure.outbox_events
  WHERE published = false
    AND dead_letter_at IS NULL
    AND next_retry_at <= NOW()
  ORDER BY created_at ASC
  LIMIT 100
  FOR UPDATE SKIP LOCKED;
  ```
- **Mechanism**: The `FOR UPDATE SKIP LOCKED` clause locks the selected rows for the duration of the transaction. If another instance queries the table simultaneously, it skips the locked rows and immediately processes the next available block of events, preventing double-publishing and lock contention.
- **Why this is acceptable for V1**: It provides native distributed locking without introducing external dependencies (like Redis or Zookeeper), keeping deployment costs inside the AWS Free Tier.

**Future Optimization: LISTEN / NOTIFY**
- In the future, PostgreSQL's `LISTEN / NOTIFY` mechanism can be used to wake the relay immediately when new events arrive.
- **Why this is optional**: It reduces latency to near-zero and eliminates polling overhead, but adds complexity to the database connection logic. We will defer this until polling overhead becomes a measurable issue.

### 4. EventEmitter2 for Internal Routing

**Why EventEmitter2?**
- Built directly into the NestJS ecosystem (`@nestjs/event-emitter`).
- Supports wildcard routing, namespaces, and asynchronous event listeners.
- No external infrastructure required, keeping AWS operational costs at zero for this component.

---

## Technical Standards

### 1. Event Categories

Events in the system are classified into three distinct categories:

1. **Domain Events:** Business events occurring inside the system.
   - *Examples:* `Sales.SalesInvoiceIssued`, `Procurement.PurchaseOrderApproved`
2. **Integration Events:** Events exchanged with external systems or third-party integrations.
   - *Examples:* `ETA.InvoiceSubmitted`, `PaymentGateway.PaymentReceived`
3. **System Events:** Infrastructure and platform-level events.
   - *Examples:* `Notification.EmailSent`, `OCR.DocumentProcessed`

*Purpose:* This categorization helps in filtering, prioritizing, and applying different retention and retry policies depending on the nature of the event.

### 2. Event Naming Conventions

Events must follow the Past-Tense Domain Event convention:
`[Context].[Entity][Action(Past Tense)]`

**Format:**
- Context: PascalCase (e.g., `Sales`, `Finance`)
- Entity: PascalCase (e.g., `Invoice`, `Quotation`)
- Action: Past Tense (e.g., `Issued`, `Approved`, `Created`)

**Examples:**
- `Sales.SalesInvoiceIssued`
- `Identity.UserAuthenticated`
- `Procurement.PurchaseOrderApproved`

### 3. Event Versioning Strategy

To support future microservices extraction and backward compatibility, events must include a version identifier.

- **Standard:** Use integer versioning attached to the event payload.
- **Rule:** If the schema of an event introduces breaking changes (removing fields, changing data types), the version must be incremented (e.g., `v1` to `v2`), and subscribers must explicitly handle or ignore the new version.
- **Routing:** Event names may optionally incorporate the version (e.g., `v1.Sales.SalesInvoiceIssued`) if routing logic requires it, but embedding the version in the payload is mandatory.

### 4. Event Payload Standards

All domain events must conform to a standardized wrapper (similar to CloudEvents) to ensure predictable routing and tracing.

**Required Payload Structure:**
```typescript
interface DomainEvent<T> {
  eventId: string;          // UUID v4
  eventName: string;        // e.g., 'Sales.SalesInvoiceIssued'
  version: number;          // e.g., 1
  correlationId: string;    // For distributed tracing across contexts
  causationId?: string;     // ID of the event/command that caused this event
  requestId: string;        // Used for tracing HTTP requests, logs, and debugging
  tenantId?: string;        // Reserved for future SaaS migration (always NULL in V1)
  timestamp: string;        // ISO-8601 UTC timestamp
  producer: string;         // e.g., 'SalesContext'
  payload: T;               // The strongly-typed event data
}
```

### 5. Ordering Guarantees

- **Ordering is guaranteed only inside the same Aggregate Root.** If an invoice has multiple events, they will be processed in the order they were emitted.
- **Ordering across different aggregates or bounded contexts is NOT guaranteed.** 
- **Consumers must never assume global ordering** and should design their handlers to tolerate out-of-order delivery where possible.

### 6. Idempotency Strategy

Because the Outbox Pattern provides **at-least-once delivery**, subscribers *must* be idempotent. They may receive the same event more than once.

- **Inbox Pattern:** Subscribers should implement an `inbox_events` table (or similar mechanism) to track the `eventId`s they have successfully processed.
- **Rule:** Before processing an event, the subscriber checks its Inbox. If the `eventId` exists, the event is acknowledged and ignored.

### 7. Retry & Dead Letter Design

If an event subscriber throws an error during processing, the outbox relay must capture the failure and apply a robust retry mechanism.

**Outbox Record Fields:**
The `outbox_events` table must contain tracking fields for the retry lifecycle:
- `retryCount`: Number of times the event has been retried.
- `nextRetryAt`: Timestamp for the next scheduled retry attempt.
- `lastError`: The error message or stack trace from the latest failure.
- `deadLetterAt`: Timestamp when the event permanently failed.

**Retry Lifecycle:**
1. **Exponential Backoff:** The relay will retry failed events using an exponential backoff strategy (e.g., retry at 1m, 5m, 15m, 1h).
2. **Dead Letter State:** After a configurable number of maximum retries (e.g., 5), the event is marked as a **DeadLetter** (setting `deadLetterAt`).
3. **Preservation:** Dead letter events are **preserved for investigation** by system administrators.
4. **No Deletion:** Dead letter events are **never deleted automatically**.

---

## Alternatives Considered & Rejected for V1

### Kafka / RabbitMQ

**Why not used in V1:**
- **Operational Cost & Complexity:** Managing a Kafka cluster or RabbitMQ broker adds significant operational overhead, infrastructure costs, and complexity for a startup. It contradicts the goal of maintaining a low-cost, AWS Free Tier-friendly footprint.
- **Overkill for a Monolith:** In a modular monolith, all contexts share the same memory space. Network-level brokers introduce unnecessary latency and serialization overhead when an in-memory emitter (EventEmitter2) backed by the database outbox achieves the same reliability.

### AWS SNS / SQS / EventBridge

**Why not used in V1:**
- **Cost:** While SNS/SQS/EventBridge scale well, continuous polling and message volume incur costs.
- **Development Friction:** Requires AWS connectivity for local development or mocking tools (like LocalStack), which slows down developer onboarding.
- **Premature Optimization:** The Outbox + EventEmitter2 pattern keeps the system entirely self-contained. If the system scales out into microservices later, the outbox relay can simply be reconfigured to publish to SNS/SQS instead of EventEmitter2, with zero changes required to the domain logic.

---

## Future Migration to Microservices

This design explicitly preserves the domain model if VArrow AI Finance migrates to microservices:

1. **Domain Logic remains untouched:** Domain services continue to write to the `outbox_events` table.
2. **Relay swap:** The Outbox Relay is updated to publish events to a distributed message broker (e.g., Kafka, SNS, or EventBridge) instead of NestJS `EventEmitter2`.
3. **Subscribers adapt:** Instead of using `@OnEvent()`, the extracted microservice uses NestJS Microservices (e.g., `@MessagePattern()`) or standard consumers to read from the broker. The idempotency logic (Inbox) remains identical.

This guarantees a clean separation between business logic (domain events) and transport infrastructure.

---

## Summary of Decisions

| Concern | Decision |
|---|---|
| Event Routing | NestJS `EventEmitter2` (synchronous, in-memory) |
| Delivery Guarantee | At-least-once via Transactional Outbox Pattern (PostgreSQL) |
| Outbox Relay | Polling (2-5s) in V1; optional LISTEN/NOTIFY in the future |
| Event Retention | Audit-retained, never automatically deleted for financial events |
| Idempotency | Subscribers must implement the Inbox Pattern |
| Ordering | Guaranteed only within the same Aggregate Root |
| Retry Strategy | Exponential backoff, transitioning to Dead Letter State on max failures |
| Event Naming | `[Context].[Entity][ActionPastTense]` |
| Versioning | Integer versioning in the payload |
| Infrastructure Cost | $0 (Database + Compute only) |

---

*This ADR defines the internal event transport strategy. Future changes, such as adopting an external message broker, must be documented in a superseding ADR.*
