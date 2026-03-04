# Domain-Driven Design (DDD)

## Table of Contents
- [Overview](#overview)
- [Origin](#origin)
- [Core Philosophy](#core-philosophy)
- [The Ubiquitous Language](#the-ubiquitous-language)
- [Strategic Design](#strategic-design)
  - [Domain & Subdomains](#domain--subdomains)
  - [Bounded Context](#bounded-context)
  - [Context Map](#context-map)
  - [Context Relationships](#context-relationships)
- [Tactical Design](#tactical-design)
  - [Entities](#entities)
  - [Value Objects](#value-objects)
  - [Aggregates](#aggregates)
  - [Domain Events](#domain-events)
  - [Repositories](#repositories)
  - [Domain Services](#domain-services)
  - [Factories](#factories)
  - [Application Services](#application-services)
- [The Layered Architecture](#the-layered-architecture)
- [When to Use](#when-to-use)
- [When Not to Use](#when-not-to-use)
- [Considerations](#considerations)
- [Popular Tools & Frameworks](#popular-tools--frameworks)
- [Related Patterns](#related-patterns)
- [References](#references)

---

## Overview

**Domain-Driven Design (DDD)** is a software design philosophy and collection of patterns that places the **business domain** at the center of all technical decisions. It advocates for building a deep, shared understanding of the problem space between domain experts and software engineers, and for expressing that understanding directly in the structure, language, and behavior of the software itself.

DDD provides both a **strategic toolkit** — for decomposing and mapping large, complex business domains — and a **tactical toolkit** — for implementing rich, expressive domain models in code. Together, these tools help teams build software that accurately reflects business reality, remains comprehensible as it grows, and can evolve alongside the business without accumulating crippling complexity.

---

## Origin

Domain-Driven Design was introduced by **Eric Evans** in his seminal 2003 book, *Domain-Driven Design: Tackling Complexity in the Heart of Software*, widely referred to as the **"Blue Book"** in the DDD community. Evans synthesized patterns and practices from his years of experience building complex software systems and formalized them into a coherent approach to managing domain complexity.

The work was later extended and complemented by **Vaughn Vernon** in *Implementing Domain-Driven Design* (2013), known as the **"Red Book"**, and by Evans himself in *Domain-Driven Design Reference* (2014), a concise summary of the pattern catalog.

DDD has grown from a niche methodology into a widely adopted design philosophy, particularly influential in the microservices and event-driven architecture movements, where its concept of **Bounded Contexts** provides the intellectual foundation for service decomposition.

---

## Core Philosophy

DDD is built on several foundational beliefs:

- **The domain is the heart of the software** — the primary value of most business software lies in the domain logic it encodes. Technical infrastructure, databases, and frameworks are secondary concerns. The domain model must be the central focus of design effort.

- **Collaboration between domain experts and developers is essential** — software that accurately reflects business reality can only be built when the people who understand the business and the people who build the software work together continuously, not just at requirements-gathering time.

- **The model is not a diagram — it is the code** — the domain model lives in the implementation. Diagrams, documents, and conversations are tools for developing understanding, but the definitive model is expressed through the structure and behavior of the running software.

- **Language shapes understanding** — the vocabulary used to talk about the domain in meetings, in documents, and in the codebase must be the same. Divergence between the language of the business and the language of the code is a symptom of a design problem.

- **Complexity must be isolated** — not all parts of a system are equally complex or equally important. Strategic design identifies where the real complexity lies and focuses modeling effort there, while protecting that complexity from being polluted by simpler, more generic concerns.

---

## The Ubiquitous Language

The **Ubiquitous Language** is the single most important concept in DDD. It is a **shared, precise vocabulary** built collaboratively by domain experts and developers, used consistently in all conversations, documentation, and — critically — in the code itself.

```
  Domain Expert                    Developer
       │                               │
       │   "When a customer places     │
       │    an Order, we reserve       │
       │    Inventory for each         │
       │    Line Item before           │
       │    confirming the Order"      │
       │                               │
       └───────────────┬───────────────┘
                       │
                       ▼
           ┌────────────────────────┐
           │    Ubiquitous Language │
           │                        │
           │  - Customer            │
           │  - Order               │
           │  - LineItem            │
           │  - Inventory           │
           │  - Reserve             │
           │  - Confirm             │
           └────────────────────────┘
                       │
          ┌────────────┼────────────┐
          ▼            ▼            ▼
     Conversations  Documents    Codebase
```

### Properties of the Ubiquitous Language

- **Shared** — every member of the team, business and technical, uses the same terms with the same meanings
- **Precise** — ambiguous terms are resolved. If "account" means different things to the finance team and the customer support team, this ambiguity is made explicit and resolved
- **Evolving** — the language grows and refines as understanding of the domain deepens. When the language changes, the code changes with it
- **Bounded** — the same word may mean different things in different parts of the business. The Ubiquitous Language is defined within the scope of a Bounded Context

### Why It Matters

When the language of the business and the language of the code diverge, developers must constantly translate between the two mental models. This translation is a source of bugs, misunderstandings, and design decisions that do not reflect business intent. A well-maintained Ubiquitous Language eliminates this translation layer entirely.

---

## Strategic Design

Strategic design addresses the **big picture** — how a large, complex domain is decomposed into manageable parts, how those parts relate to each other, and where to invest the most design effort.

---

### Domain & Subdomains

The **Domain** is the entire problem space that the software addresses — the business area the system exists to serve.

Because most real-world domains are large and complex, DDD encourages decomposing the domain into **Subdomains** — cohesive areas of the business that can be reasoned about relatively independently.

```
                    ┌──────────────────────────────────────┐
                    │            Business Domain           │
                    │        (e.g. E-Commerce Platform)    │
                    │                                      │
                    │  ┌──────────┐  ┌──────────────────┐  │
                    │  │  Order   │  │    Inventory     │  │
                    │  │Management│  │    Management    │  │
                    │  └──────────┘  └──────────────────┘  │
                    │                                      │
                    │  ┌──────────┐  ┌──────────────────┐  │
                    │  │  Payment │  │    Customer      │  │
                    │  │Processing│  │    Management    │  │
                    │  └──────────┘  └──────────────────┘  │
                    │                                      │
                    │  ┌──────────┐  ┌──────────────────┐  │
                    │  │Shipping &│  │  Notifications   │  │
                    │  │ Logistics│  │                  │  │
                    │  └──────────┘  └──────────────────┘  │
                    └──────────────────────────────────────┘
```

#### Subdomain Types

| Type                  | Description                                                                                                         | Design Investment      |
|-----------------------|---------------------------------------------------------------------------------------------------------------------|------------------------|
| **Core Domain**       | The area that provides the primary competitive advantage of the business. This is what makes the business unique and valuable. | Highest — apply full DDD tactical patterns |
| **Supporting Subdomain** | Areas that are necessary for the core domain to function but do not differentiate the business on their own.     | Medium — custom built but simpler models acceptable |
| **Generic Subdomain** | Areas that are common across many businesses and industries (e.g. authentication, billing, email delivery). No competitive advantage exists here. | Low — buy, use open source, or use SaaS |

Distinguishing between these three types is one of the most strategically important decisions in DDD. It directs where design effort is focused and where it is deliberately not focused.

---

### Bounded Context

A **Bounded Context** is the explicit boundary within which a particular domain model is defined and applicable. Inside a Bounded Context, every term in the Ubiquitous Language has a precise, unambiguous meaning. Outside that boundary, the same term may have a different meaning or a different model may apply entirely.

```
  ┌─────────────────────────────────┐    ┌──────────────────────────────────┐
  │    Order Management Context     │    │     Shipping & Logistics Context  │
  │                                 │    │                                  │
  │  "Order" = a customer's         │    │  "Order" = a fulfillment          │
  │  purchase intent with           │    │  work item with a destination,    │
  │  line items, pricing,           │    │  package dimensions, and          │
  │  and payment status             │    │  carrier assignment               │
  │                                 │    │                                  │
  └─────────────────────────────────┘    └──────────────────────────────────┘
```

The same real-world concept — an "Order" — is modeled differently in each context because each context cares about different attributes and behaviors of that concept. Attempting to build a single, unified "Order" model that satisfies both contexts produces a large, brittle model that serves neither well.

#### Key Properties of Bounded Contexts

- **Explicit boundaries** — the boundary of a context is a conscious design decision, not an emergent artifact
- **Owns its model** — the model inside a Bounded Context is owned by the team responsible for that context and is not shared or modified by other contexts
- **Owns its data** — each Bounded Context typically has its own data store, preventing other contexts from accessing its data directly
- **Communicates through interfaces** — contexts communicate with each other through well-defined interfaces — events, APIs, or messaging — never through shared databases

---

### Context Map

A **Context Map** is a document (often a diagram) that captures the landscape of Bounded Contexts in a system and the relationships between them. It makes explicit how contexts interact, who is upstream and downstream in each relationship, and where translation between models is required.

```
  ┌─────────────────┐         ┌──────────────────┐        ┌──────────────────┐
  │  Order Mgmt     │──────── │  Inventory Mgmt  │ ───────│  Shipping &      │
  │  Context        │ (ACL)   │  Context         │        │  Logistics       │
  │                 │         │                  │        │  Context         │
  └────────┬────────┘         └──────────────────┘        └──────────────────┘
           │
           │ (Published Language)
           ▼
  ┌─────────────────┐         ┌──────────────────┐
  │  Payment        │         │  Notification    │
  │  Processing     │         │  Context         │
  │  Context        │         │  (Conformist)    │
  └─────────────────┘         └──────────────────┘
```

The Context Map is a living document that evolves as the system grows. It is one of the most valuable communication tools in a DDD project, making invisible organizational and technical dependencies visible.

---

### Context Relationships

When two Bounded Contexts interact, DDD defines several patterns that describe the nature of that relationship:

| Relationship Pattern          | Description                                                                                                                                                               |
|-------------------------------|---------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **Partnership**               | Two contexts are developed jointly by teams that collaborate closely. Changes are coordinated and planned together.                                                        |
| **Shared Kernel**             | Two contexts share a small, explicitly agreed-upon portion of their domain model. Changes to the shared kernel require consent from both teams.                            |
| **Customer / Supplier**       | One context (supplier/upstream) produces output that another context (customer/downstream) consumes. The downstream team negotiates its needs with the upstream team.     |
| **Conformist**                | The downstream context has no influence over the upstream context and simply conforms to the upstream model, even if it is not ideal.                                     |
| **Anti-Corruption Layer (ACL)** | The downstream context introduces a translation layer that converts the upstream model into its own model, protecting its domain from being shaped by another context's design choices. |
| **Open Host Service**         | The upstream context defines a formal, versioned protocol or API for integration, making it easy for multiple downstream contexts to consume it.                          |
| **Published Language**        | The upstream context publishes a well-documented, shared language (often a schema or event format) for integration, decoupled from its internal model.                   |
| **Separate Ways**             | Two contexts have no meaningful integration. They solve overlapping problems independently, accepting duplication over the cost of integration.                           |

---

## Tactical Design

Tactical design provides the **building blocks** for implementing a rich domain model within a single Bounded Context. These patterns work together to accurately express domain concepts in the structure and behavior of the code.

---

### Entities

An **Entity** is a domain object defined by its **identity** rather than its attributes. Two entities are considered the same if they share the same identity, regardless of whether their attributes differ. Entities have a lifecycle — they are created, they change over time, and they may eventually be archived or deleted.

**Examples:** `Customer`, `Order`, `BankAccount`, `Employee`

A `Customer` named "Alice Smith" who changes her name to "Alice Jones" is still the same `Customer` — her identity (customer ID) has not changed, only her attributes have.

---

### Value Objects

A **Value Object** is a domain object defined entirely by its **attributes**. It has no conceptual identity. Two value objects with the same attributes are considered identical and interchangeable. Value objects are immutable — if you need a different value, you create a new value object rather than modifying an existing one.

**Examples:** `Money`, `Address`, `DateRange`, `EmailAddress`, `Coordinates`

A monetary amount of `$100 USD` is the same regardless of which instance represents it. If you need `$150 USD`, you create a new `Money` value object rather than changing the existing one.

| Dimension         | Entity                                    | Value Object                              |
|-------------------|-------------------------------------------|-------------------------------------------|
| **Identity**      | Has a unique, persistent identity         | No identity — defined by its attributes   |
| **Mutability**    | Mutable — attributes change over time     | Immutable — replaced, not modified        |
| **Equality**      | Equal if same identity (ID)               | Equal if all attributes are equal         |
| **Lifecycle**     | Has a lifecycle (created, changed, ended) | No lifecycle — created and discarded      |
| **Examples**      | Customer, Order, Account                  | Money, Address, DateRange, Color          |

---

### Aggregates

An **Aggregate** is a cluster of domain objects (Entities and Value Objects) that are treated as a single unit for the purposes of data changes. Each Aggregate has a single **Aggregate Root** — the Entry through which all external access to the aggregate must pass.

```
  ┌─────────────────────────────────────────────────────────┐
  │                      Order Aggregate                     │
  │                                                         │
  │   ┌──────────────────────────┐                          │
  │   │     Order (Root Entity)  │ ◄── All external access  │
  │   │                          │     goes through here    │
  │   │  - orderId               │                          │
  │   │  - status                │                          │
  │   │  - totalAmount (VO)      │                          │
  │   └──────────────┬───────────┘                          │
  │                  │  contains                            │
  │         ┌────────┴─────────┐                            │
  │         ▼                  ▼                            │
  │   ┌───────────┐     ┌───────────┐                       │
  │   │ LineItem  │     │ LineItem  │                       │
  │   │ (Entity)  │     │ (Entity)  │                       │
  │   │           │     │           │                       │
  │   │ quantity  │     │ quantity  │                       │
  │   │ price(VO) │     │ price(VO) │                       │
  │   └───────────┘     └───────────┘                       │
  └─────────────────────────────────────────────────────────┘
```

#### Aggregate Design Rules

- **Access through the root only** — no external object holds a direct reference to an internal entity of the aggregate. All operations go through the aggregate root.
- **Consistency boundary** — the aggregate enforces all business invariants within its boundary. All objects within the aggregate are consistent with each other at the end of every transaction.
- **Transactional boundary** — one transaction saves one aggregate. Saving multiple aggregates in a single transaction is a design smell indicating that boundaries may be wrong.
- **Small aggregates** — aggregates should be kept as small as possible. Large aggregates with many children create contention, performance problems, and overly broad consistency boundaries.
- **Reference by identity** — aggregates reference other aggregates by identity (ID), not by direct object reference.

---

### Domain Events

A **Domain Event** is a record of something significant that has happened in the domain — a fact about a state change that domain experts care about and that other parts of the system may need to react to.

```
  Command ──► Aggregate ──► State Change ──► Domain Event Published
                                             │
                          ┌──────────────────┤
                          ▼                  ▼
                   Event Handler A    Event Handler B
                  (Update Read Model) (Send Notification)
```

#### Properties of Domain Events

- **Named in the past tense** — a domain event records something that has already happened: `OrderPlaced`, `PaymentProcessed`, `InventoryReserved`, `CustomerRegistered`
- **Immutable** — events are facts about the past and cannot be changed or deleted
- **Carry relevant context** — an event carries enough information for consumers to act on it without needing to query back to the source
- **Decoupled** — the aggregate that raises an event does not know who consumes it or what they do with it

Domain Events are the mechanism by which the write side of a CQRS system notifies the read side of state changes, and the mechanism by which Bounded Contexts communicate with each other in an event-driven architecture.

---

### Repositories

A **Repository** provides an abstraction over the persistence mechanism for Aggregates. It presents a collection-like interface — the domain model retrieves and stores aggregates through a repository without knowing anything about the underlying database, ORM, or storage technology.

#### Key Principles

- **One repository per aggregate root** — repositories exist for aggregate roots only, not for every entity or value object
- **Persistence ignorance** — the domain model does not depend on database concepts. The repository interface is defined in the domain layer; its implementation lives in the infrastructure layer
- **Collection semantics** — from the domain model's perspective, a repository behaves like an in-memory collection of aggregates. The fact that data is being loaded from or saved to a database is an infrastructure detail hidden behind the repository interface

---

### Domain Services

A **Domain Service** is a stateless operation that expresses an important domain concept or business rule that does not naturally belong to any single Entity or Value Object.

When a significant domain operation involves multiple aggregates or requires coordination that feels awkward to place on any one aggregate, a Domain Service is the appropriate home for that logic.

**Examples:** `FundsTransferService` (coordinating a transfer between two `BankAccount` aggregates), `TaxCalculationService`, `OrderPricingService`

Domain Services are distinct from Application Services — they contain domain logic and are expressed in the Ubiquitous Language, whereas Application Services coordinate use cases and contain no business logic of their own.

---

### Factories

A **Factory** encapsulates the complex creation logic required to construct a valid Aggregate or Entity. When creating a domain object requires significant business logic, assembling multiple objects, or enforcing invariants during construction, a Factory makes the creation intent explicit and keeps the creation complexity out of the domain objects themselves.

Factories may be standalone Factory objects, Factory Methods on the Aggregate Root, or Domain Services that handle creation.

---

### Application Services

**Application Services** sit at the boundary between the domain layer and the outside world (APIs, UI, messaging). They orchestrate the steps of a use case — loading aggregates from repositories, invoking domain logic, persisting changes, and raising events — without containing any business logic themselves. All business rules live in the domain model.

```
  API / UI / Message Consumer
           │
           ▼
  ┌─────────────────────────────────┐
  │       Application Service       │
  │                                 │
  │  1. Load aggregate              │
  │  2. Invoke domain method        │
  │  3. Persist via repository      │
  │  4. Publish domain events       │
  └───────────────┬─────────────────┘
                  │
                  ▼
         ┌─────────────────┐
         │  Domain Model   │
         │  (Aggregate,    │
         │   Entities,     │
         │   Value Objects,│
         │   Domain Svc)   │
         └─────────────────┘
```

---

## The Layered Architecture

DDD is most effective when combined with a **layered architecture** that enforces a clean separation between the domain model and its surrounding infrastructure. The dependency rule states that layers may only depend on layers below them — and the domain layer depends on nothing outside itself.

```
  ┌──────────────────────────────────────────────────────────────┐
  │                    Presentation Layer                        │
  │         (REST Controllers, GraphQL, gRPC, Web UI)            │
  └──────────────────────────────┬───────────────────────────────┘
                                 │
  ┌──────────────────────────────▼───────────────────────────────┐
  │                    Application Layer                         │
  │   (Application Services, Use Case Orchestration, DTOs)       │
  └──────────────────────────────┬───────────────────────────────┘
                                 │
  ┌──────────────────────────────▼───────────────────────────────┐
  │                      Domain Layer                            │
  │  (Entities, Value Objects, Aggregates, Domain Events,        │
  │   Domain Services, Repositories (interfaces), Factories)     │
  │                                                              │
  │              ★ The heart of the system ★                    │
  │         No dependencies on infrastructure or frameworks      │
  └──────────────────────────────┬───────────────────────────────┘
                                 │
  ┌──────────────────────────────▼───────────────────────────────┐
  │                  Infrastructure Layer                        │
  │  (Repository Implementations, ORM, Databases, Message Buses, │
  │   External APIs, Email, File Storage, Caching)               │
  └──────────────────────────────────────────────────────────────┘
```

### The Dependency Inversion Principle in DDD

The Domain Layer defines **interfaces** for Repositories and other infrastructure concerns. The Infrastructure Layer provides the **implementations** of those interfaces. This means the Domain Layer never depends on infrastructure — infrastructure depends on the domain. This is the Dependency Inversion Principle applied at the architectural level, and it is what allows the domain model to be developed, tested, and reasoned about in complete isolation from databases, frameworks, and external services.

---

## When to Use

- **Complex business domains** with rich, non-trivial business rules, processes, and invariants that require careful modeling — DDD's value scales with domain complexity
- **Long-lived systems** that will evolve alongside the business over years or decades — the investment in a well-structured domain model pays dividends as requirements change
- **Collaborative environments** where domain experts are available and willing to participate in the design process — the Ubiquitous Language requires ongoing collaboration to build and maintain
- **Microservices decomposition** — DDD's Bounded Context concept provides the principled basis for defining service boundaries in a microservices architecture
- **Event-driven architectures** — Domain Events map naturally to the messages flowing between services in an event-driven system
- **Teams that have adopted CQRS or Event Sourcing** — DDD provides the domain modeling foundation that makes these architectural patterns coherent and manageable

---

## When Not to Use

- **Simple CRUD applications** with little or no business logic — applying the full DDD tactical pattern catalog to a system that is essentially a data entry form over a database adds significant complexity for no benefit
- **Data-centric systems** where the primary purpose is reporting, analytics, or data transformation rather than enforcing complex business rules
- **Small teams or short-lived projects** where the investment in building a shared Ubiquitous Language and a layered architecture is not justified by the project's lifespan or scope
- **When domain experts are unavailable** — DDD fundamentally requires collaboration with people who deeply understand the business domain. Without that collaboration, the Ubiquitous Language cannot be built and the domain model will not reflect reality
- **Generic subdomains** — applying rich domain modeling to commodity concerns such as authentication, billing, or email delivery is wasteful. Use off-the-shelf solutions for generic subdomains and reserve DDD for the core domain

---

## Considerations

- **Strategic Before Tactical**: The most impactful DDD work is strategic — defining Bounded Contexts, building the Ubiquitous Language, and drawing the Context Map. Many teams jump directly to tactical patterns (Entities, Aggregates, Repositories) without doing the strategic work first, and end up with technically correct but business-misaligned models. Always start with strategic design.

- **The Ubiquitous Language Requires Ongoing Investment**: Building the Ubiquitous Language is not a one-time workshop exercise. It requires continuous refinement as understanding deepens. When domain experts and developers use different words for the same thing, or the same word for different things, this is a signal that the language — and likely the model — needs work.

- **Aggregate Design is Hard**: Getting aggregate boundaries right is one of the most difficult and consequential design decisions in DDD. Aggregates that are too large create contention and performance problems. Aggregates that are too small may not be able to enforce their invariants. Iterate on aggregate design as you learn more about the domain's consistency requirements.

- **Not All Code Is the Domain Model**: DDD does not mean that every line of code must be expressive, rich domain logic. Infrastructure code, configuration, logging, and deployment scripts are not the domain model. Reserve the investment in rich modeling for the core domain, and keep everything else as simple as possible.

- **DDD and Microservices**: DDD's Bounded Context is the best available heuristic for defining microservice boundaries. However, not every Bounded Context needs to be a separate microservice — a modular monolith with well-separated Bounded Contexts implemented as modules is often a better starting point than immediately decomposing into microservices.

- **Refactoring Towards DDD**: It is not necessary to apply DDD to an entire existing system at once. The Strangler Fig pattern can be used to incrementally extract well-modeled Bounded Contexts from a legacy system over time, applying DDD principles to each new context as it is built.

- **Documentation of the Model**: The domain model, the Ubiquitous Language glossary, and the Context Map are living artifacts that must be actively maintained. A domain model that is documented once and never updated provides false confidence and misleads new team members. Treat these artifacts as first-class deliverables.

---

## Popular Tools & Frameworks

| Tool / Framework                        | Language / Platform   | Description                                                                                |
|-----------------------------------------|-----------------------|--------------------------------------------------------------------------------------------|
| **Axon Framework**                      | Java                  | Full DDD + CQRS + Event Sourcing framework with command bus, event bus, and projections     |
| **EventStoreDB**                        | Platform-agnostic     | Purpose-built event store for Event Sourcing, widely used as the DDD write store           |
| **Marten**                              | .NET                  | PostgreSQL-based document and event store with native DDD and CQRS support                 |
| **NServiceBus / MassTransit**           | .NET                  | Message bus frameworks for distributing domain events across bounded contexts              |
| **MediatR**                             | .NET                  | In-process command/query dispatching, commonly used in DDD Application Service layers      |
| **Domain Storytelling**                 | Modeling tool         | Visual collaborative modeling technique for building the Ubiquitous Language with domain experts |
| **EventStorming**                       | Modeling workshop     | Collaborative workshop format for rapidly discovering domain events, aggregates, and bounded contexts |
| **Context Mapper**                      | Modeling / Code gen   | DSL and tooling for drawing Context Maps and generating DDD-aligned code structures        |
| **Whirlpool Process / Example Mapping** | Discovery technique   | Structured conversations for exploring domain rules and building shared understanding       |

---

## Related Patterns

| Pattern                           | Relationship                                                                                                                    |
|-----------------------------------|---------------------------------------------------------------------------------------------------------------------------------|
| **CQRS**                          | A natural complement to DDD — Aggregates process commands on the write side; Domain Events populate read-side projections       |
| **Event Sourcing**                | Domain Events become the storage mechanism — the aggregate's full history is preserved as an immutable event stream             |
| **Bounded Context / Microservices** | DDD Bounded Contexts provide the principled basis for defining microservice boundaries                                        |
| **Anti-Corruption Layer**         | A DDD integration pattern that protects a Bounded Context's domain model from being corrupted by an upstream context's model   |
| **Strangler Fig**                 | DDD is commonly the target architecture when decomposing a legacy monolith using the Strangler Fig pattern                     |
| **Sidecar**                       | Infrastructure concerns (logging, tracing, security) are delegated to sidecars, keeping the domain model free of infrastructure |
| **Saga / Process Manager**        | A DDD pattern for managing long-running, multi-step business processes that span multiple Aggregates or Bounded Contexts        |
| **Repository**                    | A core DDD tactical pattern providing persistence abstraction for Aggregates                                                   |
| **Specification Pattern**         | Encapsulates business rules as composable objects, often used alongside DDD Entities and Repositories                          |

---

## References

- Evans, E. *Domain-Driven Design: Tackling Complexity in the Heart of Software*. Addison-Wesley, 2003. (The "Blue Book")
- Vernon, V. *Implementing Domain-Driven Design*. Addison-Wesley, 2013. (The "Red Book")
- Vernon, V. *Domain-Driven Design Distilled*. Addison-Wesley, 2016.
- Evans, E. *Domain-Driven Design Reference: Definitions and Pattern Summaries*. Domain Language, Inc., 2014.
- Richardson, C. *Microservices Patterns*. Manning Publications, 2018.
- Fowler, M. *BoundedContext*. martinfowler.com. [https://martinfowler.com/bliki/BoundedContext.html](https://martinfowler.com/bliki/BoundedContext.html)
- [DDD Community — dddcommunity.org](https://www.dddcommunity.org/)
- [Microsoft — Domain Analysis for Microservices](https://learn.microsoft.com/en-us/azure/architecture/microservices/model/domain-analysis)
- [EventStorming — Alberto Brandolini](https://www.eventstorming.com/)
