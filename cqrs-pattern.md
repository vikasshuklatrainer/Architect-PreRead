# CQRS (Command Query Responsibility Segregation) Pattern

## Table of Contents
- [Overview](#overview)
- [Origin](#origin)
- [Problem](#problem)
- [Solution](#solution)
- [How It Works](#how-it-works)
- [Structure](#structure)
- [Commands vs Queries](#commands-vs-queries)
- [CQRS with Event Sourcing](#cqrs-with-event-sourcing)
- [Consistency Model](#consistency-model)
- [Read Model Projections](#read-model-projections)
- [When to Use](#when-to-use)
- [When Not to Use](#when-not-to-use)
- [Considerations](#considerations)
- [Popular Tools & Frameworks](#popular-tools--frameworks)
- [Related Patterns](#related-patterns)
- [References](#references)

---

## Overview

**CQRS (Command Query Responsibility Segregation)** is an architectural pattern that separates the **read** and **write** operations of a system into two distinct models. The **Command** side handles all operations that mutate state (create, update, delete), while the **Query** side handles all operations that retrieve data. Each side is optimized independently for its specific workload, allowing the system to scale, evolve, and perform more effectively than a single unified model can.

CQRS is frequently applied in complex domains, event-driven architectures, and systems with significant asymmetry between read and write load patterns.

---

## Origin

CQRS was formulated by **Greg Young** around 2010, building on the earlier **Command Query Separation (CQS)** principle introduced by **Bertrand Meyer** in his work on the Eiffel programming language. Meyer's CQS principle states that a method should either be a **command** (which performs an action and changes state but returns nothing) or a **query** (which returns data but causes no side effects) — never both.

CQRS elevates this principle from the method level to the **architectural level**, applying the separation not just to individual operations but to entire data models, data stores, and processing pipelines.

---

## Problem

In traditional architectures, a single unified data model is responsible for handling both reads and writes. This approach, while simple, introduces growing friction as systems scale in complexity:

- **Read and write workloads have fundamentally different characteristics** — writes require strong consistency, validation, and transactional integrity, while reads require speed, flexibility, and often highly denormalized data shaped for specific views
- **The same data model cannot be optimized for both** — a normalized schema optimized for write consistency is poorly suited to serving complex, aggregated read queries efficiently
- **Read-heavy systems are bottlenecked by write-optimized storage** — most real-world systems have far more reads than writes, yet both compete for the same data store resources
- **Complex domain logic is entangled with data retrieval logic** — business rules, validation, and state transitions become mixed with query logic, making both harder to understand and evolve independently
- **Scaling reads and writes independently is not possible** — a single model forces both concerns to scale together, even when only one is under pressure

---

## Solution

Segregate the system into two separate models:

- A **Command Model (Write Side)** — responsible for receiving and processing commands that change the state of the system. Optimized for consistency, validation, and domain logic.
- A **Query Model (Read Side)** — responsible for answering queries by returning pre-shaped, optimized views of the data. Optimized purely for read performance.

These two models may share the same physical data store or use entirely separate stores, depending on the degree of separation required. When separate stores are used, they are kept synchronized through an asynchronous process — typically via domain events raised by the command side and consumed by the query side.

```
                       ┌──────────────┐
                       │    Client    │
                       └──────┬───────┘
                    ┌─────────┴──────────┐
                    │                    │
               Commands               Queries
                    │                    │
                    ▼                    ▼
          ┌──────────────────┐  ┌──────────────────┐
          │   Command Model  │  │   Query Model    │
          │   (Write Side)   │  │   (Read Side)    │
          │                  │  │                  │
          │ - Validates input│  │ - Returns views  │
          │ - Enforces rules │  │ - Denormalized   │
          │ - Mutates state  │  │ - Read-optimized │
          └────────┬─────────┘  └──────────────────┘
                   │                    ▲
                   │  Domain Events /   │
                   │  Synchronization   │
                   └────────────────────┘
```

---

## How It Works

1. A **client** (user interface, API consumer, or another service) submits either a **command** or a **query**
2. **Commands** are routed to the command side:
   - The command is validated against business rules
   - The relevant domain aggregate is loaded from the write store
   - The aggregate processes the command and transitions to a new state
   - The updated state (or the events representing the state change) is persisted to the write store
   - One or more **domain events** are raised to signal that a state change has occurred
3. **Domain events** flow from the write side to the read side:
   - Event handlers on the read side consume these events
   - The read store (materialized views) is updated to reflect the new state
4. **Queries** are routed to the query side:
   - The query is executed directly against the read store
   - Pre-shaped, denormalized data is returned to the caller with minimal processing
   - No domain logic is involved in query execution

---

## Structure

```
┌─────────────────────────────────────────────────────────────────────┐
│                            CQRS Architecture                        │
│                                                                     │
│   ┌───────────┐        ┌────────────────────────────────────────┐   │
│   │  Client   │        │              Command Side              │   │
│   │           │──────► │                                        │   │
│   │  (sends   │Commands│  ┌──────────────┐  ┌───────────────┐  │   │
│   │  commands │        │  │Command Handler│  │    Domain     │  │   │
│   │  and      │        │  │(Validation & │  │   Aggregate   │  │   │
│   │  queries) │        │  │ Dispatch)    │  │  (Business    │  │   │
│   │           │        │  └──────┬───────┘  │   Logic)      │  │   │
│   └─────┬─────┘        │         │          └───────┬───────┘  │   │
│         │              │         ▼                  ▼          │   │
│         │              │  ┌─────────────────────────────────┐  │   │
│         │              │  │         Write Store             │  │   │
│         │              │  │  (Normalized / Event Store)     │  │   │
│         │              │  └────────────────┬────────────────┘  │   │
│         │              └───────────────────│────────────────────┘   │
│         │                                  │                        │
│         │                          Domain Events                    │
│         │                                  │                        │
│         │              ┌───────────────────│────────────────────┐   │
│         │              │    Read Side      ▼                    │   │
│         │              │  ┌──────────────────────────────────┐  │   │
│         │              │  │         Event Handlers           │  │   │
│         │              │  │   (Update Materialized Views)    │  │   │
│         │              │  └────────────────┬─────────────────┘  │   │
│         │              │                   ▼                    │   │
│         │              │  ┌──────────────────────────────────┐  │   │
│         │              │  │          Read Store              │  │   │
│         │              │  │  (Denormalized / View-Optimized) │  │   │
│         │              │  └────────────────┬─────────────────┘  │   │
│         │              └───────────────────│────────────────────┘   │
│         │                                  │                        │
│         └────────────Queries───────────────┘                        │
└─────────────────────────────────────────────────────────────────────┘
```

---

## Commands vs Queries

| Dimension            | Command                                          | Query                                           |
|----------------------|--------------------------------------------------|-------------------------------------------------|
| **Purpose**          | Change the state of the system                   | Retrieve data from the system                   |
| **Return value**     | Typically none, or an acknowledgement            | Always returns data                             |
| **Side effects**     | Intentional — state mutation is the goal         | None — must not modify state                    |
| **Validation**       | Enforced — commands are rejected if invalid      | Not applicable                                  |
| **Data model**       | Normalized, domain-consistent write model        | Denormalized, view-optimized read model         |
| **Consistency**      | Strong consistency required                      | Eventual consistency acceptable                 |
| **Examples**         | `PlaceOrder`, `UpdateAddress`, `CancelBooking`   | `GetOrderSummary`, `ListProductsByCategory`     |

---

## CQRS with Event Sourcing

CQRS is frequently used in combination with the **Event Sourcing** pattern, though the two are independent and can be used separately.

When combined:

- The **write store becomes an event store** — rather than persisting the current state of an aggregate, every state change is stored as an immutable, ordered sequence of domain events
- The **event store is the single source of truth** — the current state of any aggregate can always be reconstructed by replaying its event history
- **Materialized views on the read side** are built by consuming and projecting these events into read-optimized representations
- Because all history is preserved in the event store, **new read models can be built at any time** by replaying the full event history — even views that were not anticipated when the system was first designed
- The event store enables **full auditability** — every state change, its cause, and its timestamp are permanently recorded

```
  Command Side                        Read Side
  ──────────────                      ──────────────────────────────────
  Command                             Projection A  →  View Store A
     │                                (e.g. Order Summary by Customer)
     ▼
  Aggregate  ──► Event Store ────────► Projection B  →  View Store B
                 (Source of Truth)     (e.g. Sales Report by Region)
                 [Event 1]
                 [Event 2]            Projection C  →  View Store C
                 [Event 3]            (e.g. Inventory Dashboard)
                 [Event N]
```

---

## Consistency Model

CQRS-based systems with separate read and write stores are **eventually consistent** by nature. Understanding the consistency model is critical when designing systems with this pattern.

### Write Side
The write side enforces **strong consistency** within a single aggregate. A command either fully succeeds or fully fails — partial state mutations do not occur.

### Read Side
The read side is **eventually consistent** with the write side. After a command is processed and events are raised, there is a propagation delay before the read store reflects the new state. This delay is typically very short (milliseconds to seconds) but is non-zero.

### Implications for UI Design
Applications built on eventually consistent CQRS systems must account for this delay:

- After submitting a command, the UI should not immediately re-query the read side expecting to see the updated state
- Common strategies include **optimistic UI updates** (updating the UI immediately based on the assumed outcome of the command), **polling with a delay**, or **event-driven UI refresh** triggered by a notification from the server

---

## Read Model Projections

A **projection** is the process of transforming a stream of domain events into a specific read model (materialized view) optimized for a particular query or screen. Each projection subscribes to relevant events and maintains its own view store.

### Key Characteristics of Projections
- **Purpose-built** — each projection is shaped for a specific query or display requirement, not a general-purpose data store
- **Rebuilt on demand** — because the event store is the source of truth, any projection can be torn down and rebuilt by replaying the event history. This makes it safe to change or add projections at any time
- **Independently scalable** — each projection and its view store can be scaled independently based on the query load it serves
- **Potentially multiple per event** — a single domain event may update multiple projections simultaneously

### Projection Types

| Type                     | Description                                                                                  |
|--------------------------|----------------------------------------------------------------------------------------------|
| **Synchronous**          | The projection is updated in the same transaction as the command. Provides immediate consistency but reduces write throughput |
| **Asynchronous**         | The projection is updated after the command completes, via an event queue or message bus. Provides better write performance at the cost of read-side lag |
| **On-demand / Live**     | The projection is computed in real time from the event store when a query is received. Eliminates view store maintenance but can be slow for large event streams |

---

## When to Use

- Systems with **complex domain logic** where rich business rules govern state transitions and the domain model benefits from being isolated from read concerns
- Applications with a **significant asymmetry between read and write load** — where reads vastly outnumber writes and need to be served at high speed from optimized views
- Systems that require **multiple representations of the same data** — different clients (mobile app, web dashboard, reporting system) need different shapes of the same underlying data
- **Collaborative domains** where multiple users may be acting on the same data simultaneously, and the system needs to handle contention carefully
- Systems adopting **Event Sourcing**, where CQRS provides the natural architectural complement
- Applications requiring a **full audit trail** of all state changes for compliance, debugging, or business intelligence purposes
- **Microservices architectures** where individual services own their data and communicate state changes through events

---

## When Not to Use

- **Simple CRUD applications** where domain logic is minimal and a single unified model is sufficient — CQRS adds significant complexity that is not justified for basic create, read, update, delete operations
- **Small teams or early-stage products** where the overhead of maintaining two models, two stores, and an event pipeline outweighs the benefits
- When the **team is not yet familiar with eventual consistency** — applications and UIs must be explicitly designed to tolerate the read-write propagation delay, which requires a different mental model from traditional synchronous systems
- When **strong read-after-write consistency is an absolute requirement** — if users must always immediately see the exact result of their own actions in a query, eventual consistency is problematic
- **Time-sensitive or low-latency domains** where the overhead of event propagation and projection updates introduces unacceptable delays

---

## Considerations

- **Eventual Consistency Management**: The most common source of bugs and user confusion in CQRS systems is the propagation delay between the write and read sides. Establish clear guidelines for how the application layer handles this — use optimistic UI updates, version tokens, or server-sent notifications to bridge the gap where necessary.

- **Projection Rebuilding**: One of the most powerful features of CQRS with Event Sourcing is the ability to rebuild projections from the event store. Plan for this from the start — ensure the event store retains full history, and build tooling to replay and rebuild projections on demand without impacting running services.

- **Event Schema Evolution**: As the system evolves, the shape of domain events will change. Plan an event versioning and schema evolution strategy early to avoid breaking existing projections when event schemas are updated.

- **Command Validation**: Commands must be validated before being processed. Distinguish between **syntactic validation** (is the input well-formed?) which can happen at the API boundary, and **semantic validation** (is this command valid given the current state?) which must happen inside the domain aggregate.

- **Idempotency**: Command handlers and event handlers should be designed to be idempotent where possible — processing the same command or event more than once should produce the same result as processing it once. This is essential for reliability in the presence of retries or at-least-once delivery guarantees.

- **Complexity Trade-off**: CQRS introduces significant architectural complexity. There are now two models to design, two stores to operate, an event pipeline to manage, and an eventual consistency contract to uphold. This complexity is justified in complex domains with the right requirements, but should not be applied as a default architectural choice.

- **Operational Overhead**: Running separate read and write stores, event buses, and projection workers requires robust DevOps practices — monitoring the health of each component, managing event consumer lag, and maintaining projection freshness are ongoing operational responsibilities.

- **Bounded Contexts**: CQRS is most effective when applied within well-defined **bounded contexts** (a concept from Domain-Driven Design). Applying CQRS across an entire system without domain boundaries leads to a sprawling, difficult-to-manage architecture. Apply it selectively to the bounded contexts where complexity and load justify it.

---

## Popular Tools & Frameworks

| Tool / Framework                        | Role                                                                           |
|-----------------------------------------|--------------------------------------------------------------------------------|
| **Axon Framework** (Java)               | Full CQRS + Event Sourcing framework with command bus, event bus, and projections |
| **EventStoreDB**                        | Purpose-built event store for Event Sourcing, with projection support          |
| **MediatR** (.NET)                      | In-process command/query dispatching via mediator pattern                      |
| **NServiceBus / MassTransit** (.NET)    | Message bus for distributing commands and events across services               |
| **Apache Kafka**                        | Distributed event streaming platform for propagating domain events to projections |
| **AWS EventBridge / SNS + SQS**         | Managed event routing for cloud-native CQRS pipelines                         |
| **PostgreSQL / MySQL**                  | Commonly used as the write store for normalized domain data                    |
| **Elasticsearch / Redis / MongoDB**     | Commonly used as read stores for denormalized, query-optimized views           |
| **Marten** (.NET)                       | PostgreSQL-based document and event store with built-in CQRS support           |

---

## Related Patterns

| Pattern                      | Relationship                                                                                                       |
|------------------------------|--------------------------------------------------------------------------------------------------------------------|
| **Event Sourcing**           | Frequently combined with CQRS — the event store becomes the write model and the source of truth for all projections |
| **Domain-Driven Design (DDD)** | CQRS aligns naturally with DDD — aggregates process commands, and bounded contexts define the scope of each CQRS model |
| **Saga / Process Manager**   | Manages long-running, multi-step business processes that span multiple aggregates or services on the command side   |
| **Materialized View**        | The read side of a CQRS system is a collection of materialized views, each maintained by a projection              |
| **Event-Driven Architecture**| CQRS relies on domain events to synchronize the write and read sides, making it a natural fit for event-driven systems |
| **Strangler Fig**            | CQRS is commonly adopted as the target architecture when decomposing a monolith using the Strangler Fig pattern    |
| **Retry**                    | Event handlers and projection workers should implement retry logic to handle transient failures in event processing |
| **Circuit Breaker**          | Protects downstream read stores from being overwhelmed during projection rebuilds or high event throughput bursts   |

---

## References

- Young, G. *CQRS Documents*. [https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf](https://cqrs.files.wordpress.com/2010/11/cqrs_documents.pdf)
- Fowler, M. *CQRS*. martinfowler.com. [https://martinfowler.com/bliki/CQRS.html](https://martinfowler.com/bliki/CQRS.html)
- [Microsoft Azure Architecture Patterns — CQRS](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs)
- Vernon, V. *Implementing Domain-Driven Design*. Addison-Wesley.
- Richardson, C. *Microservices Patterns*. Manning Publications.
- [EventStoreDB Documentation](https://developers.eventstore.com/)
- [Axon Framework Documentation](https://docs.axoniq.io/)
