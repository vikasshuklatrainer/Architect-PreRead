# Strangler Fig Pattern

## Table of Contents
- [Overview](#overview)
- [Origin of the Name](#origin-of-the-name)
- [Problem](#problem)
- [Solution](#solution)
- [How It Works](#how-it-works)
- [Migration Phases](#migration-phases)
- [Migration Strategies](#migration-strategies)
- [Structure](#structure)
- [When to Use](#when-to-use)
- [When Not to Use](#when-not-to-use)
- [Considerations](#considerations)
- [Popular Tools & Techniques](#popular-tools--techniques)
- [Related Patterns](#related-patterns)
- [References](#references)

---

## Overview

The **Strangler Fig** pattern is an architectural migration pattern that enables the **incremental replacement of a legacy system** with a new system, without requiring a single large-scale "big bang" rewrite. The new system is built alongside the old one, gradually taking over functionality piece by piece, until the legacy system is entirely replaced and can be safely decommissioned.

This pattern is widely used in enterprise software modernization, legacy-to-microservices migrations, and cloud adoption journeys where the risk of rewriting an entire system at once is prohibitively high.

---

## Origin of the Name

The pattern was named by **Martin Fowler** in 2004, inspired by the **strangler fig tree** found in tropical rainforests. This tree grows by sending roots down from the canopy of a host tree, gradually enveloping and surrounding it. Over many years, the strangler fig grows to full strength while the host tree slowly decays and dies within. What remains is a fully self-supporting strangler fig tree standing in the exact shape of its host — the host having been entirely replaced without the canopy ever collapsing.

The analogy maps directly to software: the new system (the strangler fig) grows around and on top of the legacy system (the host tree), gradually absorbing its responsibilities until the legacy system can be switched off.

---

## Problem

Legacy systems present a profound modernization challenge. They are often:

- **Business-critical** — the system cannot be taken offline during migration without significant business impact
- **Poorly understood** — years or decades of accumulated changes, undocumented behavior, and tribal knowledge make the system difficult to reason about in its entirety
- **Tightly coupled** — logic, data, and infrastructure are deeply intertwined, making it nearly impossible to extract or replace individual parts cleanly
- **Technically outdated** — built on aging frameworks, languages, or infrastructure that are no longer supported or compatible with modern tooling
- **High-risk to rewrite** — a full rewrite attempts to recreate all behavior of a complex system at once, carrying an enormous risk of introducing regressions, missing edge cases, or delivering a system that does not match the real-world behavior of the original

The result is that teams are often trapped — the legacy system is too risky to rewrite entirely, but too problematic to leave untouched.

---

## Solution

Rather than rewriting the legacy system all at once, **incrementally intercept and redirect functionality** to a new system, one capability at a time. A facade or proxy layer sits in front of the legacy system and routes incoming requests. As new functionality is built and validated in the new system, traffic is progressively redirected away from the legacy system to the new one. The legacy system continues to serve any functionality not yet migrated, shrinking over time until it can be fully decommissioned.

```
  Before Migration                    During Migration                   After Migration

  ┌───────────┐                   ┌──────────────────┐                ┌───────────────┐
  │  Clients  │                   │     Clients      │                │    Clients    │
  └─────┬─────┘                   └────────┬─────────┘                └───────┬───────┘
        │                                  │                                  │
        ▼                                  ▼                                  ▼
  ┌───────────┐                   ┌──────────────────┐                ┌───────────────┐
  │  Legacy   │                   │ Strangler Facade  │                │   New System  │
  │  System   │                   │     (Proxy)       │                │               │
  └───────────┘                   └────────┬─────────┘                └───────────────┘
                                      │         │
                               routed │         │ not yet
                               to new │         │ migrated
                                      ▼         ▼
                              ┌────────────┐  ┌──────────┐
                              │ New System │  │  Legacy  │
                              │            │  │  System  │
                              └────────────┘  └──────────┘
```

---

## How It Works

1. **Introduce a facade** — place an intercepting layer (a proxy, API gateway, or router) in front of the legacy system. Initially, all traffic passes through this facade directly to the legacy system unchanged.
2. **Identify a migration candidate** — select a discrete, well-bounded piece of functionality to migrate first. This is typically something with clear inputs and outputs, low coupling to other legacy components, and high business value.
3. **Build the replacement** — develop the equivalent functionality in the new system, alongside the legacy system.
4. **Redirect traffic** — once the new implementation is tested and validated, update the facade to route requests for that specific functionality to the new system instead of the legacy one.
5. **Decommission the legacy slice** — once the legacy implementation of that functionality is no longer receiving traffic, remove it.
6. **Repeat** — continue this cycle for the next piece of functionality until the legacy system has been fully replaced.
7. **Decommission the legacy system** — once all functionality has been migrated and the legacy system receives no traffic, it is safely shut down. The facade may also be retired or retained as an API gateway.

---

## Migration Phases

### Phase 1 — Coexistence
The legacy system is fully operational. The facade is introduced but routes all traffic to the legacy system. No user-facing changes occur. This phase establishes the interception point and validates that the facade does not introduce instability.

```
Clients ──► Facade ──► Legacy System (100% of traffic)
```

### Phase 2 — Incremental Migration
Functionality is migrated piece by piece. The facade routes a growing proportion of requests to the new system. Both systems run in parallel. The legacy system shrinks with each completed migration.

```
Clients ──► Facade ──► New System    (migrated capabilities)
                  └──► Legacy System (remaining capabilities)
```

### Phase 3 — Completion & Decommission
All functionality has been migrated to the new system. The facade routes 100% of traffic to the new system. The legacy system is decommissioned. The facade may be retained as an ongoing routing or API management layer.

```
Clients ──► Facade ──► New System (100% of traffic)
                       Legacy System (shut down)
```

---

## Migration Strategies

### Migrate by Capability / Domain
Functionality is grouped by business domain or capability (e.g. user management, payments, notifications) and migrated domain by domain. This aligns well with teams organized around business domains and maps naturally to a microservices target architecture.

### Migrate by User Journey
End-to-end user journeys or workflows are identified and migrated as a unit. This ensures a consistent experience for users following a specific path through the system, at the cost of potentially cutting across multiple technical domains.

### Migrate by Traffic Percentage (Canary Migration)
A small percentage of traffic for a given capability is initially routed to the new system while the majority continues to hit the legacy system. The percentage is gradually increased as confidence in the new system grows. This approach reduces risk and provides a natural rollback mechanism.

### Migrate by User Segment
The new system is initially enabled only for a specific subset of users (e.g. internal users, beta users, or users in a specific region). Once validated with this segment, migration is extended to the broader user base.

---

## Structure

```
┌──────────────────────────────────────────────────────────────────┐
│                        Strangler Facade                          │
│                                                                  │
│   ┌────────────────────┐      ┌──────────────────────────────┐   │
│   │   Routing Rules    │      │     Observability Layer      │   │
│   │  (per capability,  │      │  (traffic split metrics,     │   │
│   │   path, or user)   │      │   error rates, latency)      │   │
│   └────────────────────┘      └──────────────────────────────┘   │
└────────────────────┬──────────────────────┬──────────────────────┘
                     │                      │
                     ▼                      ▼
          ┌──────────────────┐    ┌──────────────────────┐
          │   New System     │    │    Legacy System      │
          │  (growing)       │    │   (shrinking)         │
          │                  │    │                       │
          │  ┌────────────┐  │    │  ┌─────────────────┐  │
          │  │ Service A  │  │    │  │  Module X       │  │
          │  │ Service B  │  │    │  │  Module Y       │  │
          │  │ Service C  │  │    │  │  Module Z       │  │
          │  └────────────┘  │    │  └─────────────────┘  │
          └──────────────────┘    └──────────────────────┘
```

---

## When to Use

- **Large-scale legacy modernization** where rewriting the entire system at once is too risky, too costly, or would take too long
- **Business continuity is non-negotiable** — the legacy system must remain operational throughout the migration with no extended downtime
- **Incremental value delivery is required** — stakeholders need to see tangible progress and early business benefits during the migration, rather than waiting for a full rewrite to complete
- **The legacy system has a well-defined external interface** (HTTP API, message queue, database) that can serve as the interception point for a facade
- **Migrating to a microservices architecture** from a monolith, where each extracted service can be independently deployed and scaled
- **Cloud migration** where on-premises capabilities are progressively replaced by cloud-native equivalents
- When the team wants to **validate the new architecture** with real production traffic before fully committing to it

---

## When Not to Use

- When the **legacy system has no clear external interface** that can be intercepted — if clients communicate directly with internal components, introducing a facade may require significant upfront rework
- When **legacy and new system data models are fundamentally incompatible** and there is no practical path to synchronizing data between the two systems during the coexistence phase
- When the **scope of migration is small** — if the legacy component is small and well-understood, a direct replacement may be simpler and lower risk than the overhead of the Strangler Fig approach
- When the **organization cannot sustain running two systems in parallel** — the coexistence phase incurs operational overhead, as both systems must be maintained, monitored, and supported simultaneously
- When **tight coupling within the legacy system** makes it impossible to extract any discrete piece of functionality without pulling in large portions of the rest of the system

---

## Considerations

- **Define the Facade Early**: The strangler facade is the most critical component of this pattern. Investing time in designing it correctly — with clear routing logic, observability, and the ability to route incrementally by path, header, user, or percentage — pays significant dividends throughout the migration.

- **Data Migration Strategy**: One of the hardest problems in a Strangler Fig migration is data. When functionality moves to the new system, the data it relies on must also be accessible. Common strategies include dual writes (writing to both legacy and new data stores simultaneously), database-level replication, or event-driven synchronization. The data migration strategy must be defined before any functional migration begins.

- **Behavioral Parity**: The new system must behave identically to the legacy system for migrated functionality, including handling edge cases, error conditions, and undocumented behaviors that real users depend on. Invest in **characterization tests** — tests that capture the actual behavior of the legacy system — before migrating each capability.

- **Observability**: Instrument both the legacy system and the new system from the start. Compare error rates, latency, and business metrics side by side during the coexistence phase to catch regressions early. The facade should emit metrics that make the traffic split and the health of each system clearly visible.

- **Rollback Plan**: Each migration step should have a clearly defined rollback procedure. Because the facade controls routing, rolling back a migration is typically a configuration change — re-routing traffic back to the legacy system — rather than a code deployment. This significantly reduces the risk of each migration step.

- **Team Topology**: Running a Strangler Fig migration requires teams to maintain and evolve both systems simultaneously. This is demanding. Establish clear ownership boundaries — ideally, one team owns the migration of specific capabilities and is responsible for both the legacy extraction and the new implementation.

- **Avoiding the Infinite Migration**: Without clear milestones and a commitment to decommissioning legacy slices once migrated, Strangler Fig migrations can stall, leaving teams maintaining two systems indefinitely. Define decommission criteria for each migrated capability and enforce them.

- **Legacy System Freeze**: Consider placing the legacy system in a **feature freeze** once the migration begins — no new features are added to the legacy system. All new development goes into the new system. This prevents the legacy system from growing while you are trying to shrink it.

---

## Popular Tools & Techniques

| Tool / Technique                  | Role in Migration                                                                 |
|-----------------------------------|-----------------------------------------------------------------------------------|
| **API Gateway** (Kong, AWS API GW, NGINX) | Acts as the strangler facade, routing requests between legacy and new systems |
| **Service Mesh** (Istio, Linkerd)  | Provides fine-grained traffic splitting, observability, and canary routing        |
| **Feature Flags**                  | Controls which users or traffic segments are routed to the new system             |
| **Event Streaming** (Kafka, Kinesis) | Enables dual-write and data synchronization between legacy and new data stores  |
| **Change Data Capture (CDC)**      | Streams data changes from the legacy database to the new system in real time      |
| **Reverse Proxy** (NGINX, HAProxy) | Simple, lightweight facade for HTTP-based routing rules                           |
| **Characterization / Contract Tests** | Captures legacy behavior to ensure the new system maintains behavioral parity  |

---

## Related Patterns

| Pattern                          | Relationship                                                                                                        |
|----------------------------------|---------------------------------------------------------------------------------------------------------------------|
| **Anti-Corruption Layer**        | Often used alongside Strangler Fig to translate between the legacy system's domain model and the new system's model |
| **External Configuration Store** | Enables routing rules in the facade to be updated dynamically without redeployment                                  |
| **Feature Flags**                | Controls the gradual rollout of migrated functionality to users or traffic segments                                 |
| **Backends for Frontends (BFF)** | A new BFF can serve as the strangler facade, progressively absorbing backend functionality                          |
| **Sidecar**                      | A sidecar can intercept and redirect traffic at the infrastructure level without modifying application code         |
| **Circuit Breaker**              | Protects the migration by failing fast back to the legacy system if the new system becomes unavailable              |
| **CQRS**                         | Commonly adopted as part of the target architecture during a Strangler Fig migration from a monolith                |

---

## References

- Fowler, M. *StranglerFigApplication*. martinfowler.com, 2004. [https://martinfowler.com/bliki/StranglerFigApplication.html](https://martinfowler.com/bliki/StranglerFigApplication.html)
- [Microsoft Azure Architecture Patterns — Strangler Fig](https://learn.microsoft.com/en-us/azure/architecture/patterns/strangler-fig)
- Newman, S. *Building Microservices*. O'Reilly Media.
- Newman, S. *Monolith to Microservices*. O'Reilly Media.
- [Martin Fowler — Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html)
