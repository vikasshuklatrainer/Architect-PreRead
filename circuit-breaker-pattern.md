# Circuit Breaker Pattern

## Table of Contents
- [Overview](#overview)
- [Problem](#problem)
- [Solution](#solution)
- [How It Works](#how-it-works)
- [States](#states)
- [Configuration Parameters](#configuration-parameters)
- [When to Use](#when-to-use)
- [When Not to Use](#when-not-to-use)
- [Considerations](#considerations)
- [Popular Libraries](#popular-libraries)
- [Related Patterns](#related-patterns)
- [References](#references)

---

## Overview

The **Circuit Breaker** pattern is a fault-tolerance design pattern used in distributed systems to detect failures and prevent an application from repeatedly attempting operations that are likely to fail. It acts as a proxy between a service caller and the target service, monitoring for failures and "tripping" to stop the flow of requests when a failure threshold is reached — much like an electrical circuit breaker that trips to protect a circuit from damage.

This pattern was popularized by **Michael Nygard** in his book *Release It!* and is widely adopted in microservices architectures.

---

## Problem

In distributed systems, services often depend on remote services, APIs, or databases over a network. These remote calls can:

- **Fail silently** due to timeouts or transient faults
- **Cascade failures** — a single slow or failing dependency can exhaust thread pools or connection pools, causing the entire system to degrade
- **Waste resources** by repeatedly sending requests to a service that is known to be unavailable
- **Increase latency** as callers wait for timeouts before receiving an error

Without a circuit breaker, a failing dependency can bring down otherwise healthy services.

---

## Solution

Wrap calls to a remote service or resource in a **Circuit Breaker object** that monitors for failures. When failures reach a defined threshold, the circuit "opens" and subsequent calls fail immediately without attempting the operation. After a timeout period, the circuit enters a "half-open" state to test if the underlying problem has been resolved.

```
Caller  ──►  [ Circuit Breaker ]  ──►  Remote Service
                     │
              Monitors failures
              Opens on threshold
              Auto-recovers
```

---

## How It Works

The circuit breaker sits between the caller and the service. It tracks the outcome of every request and transitions between states based on the results.

```
                          ┌─────────────────────────────────┐
                          │                                 │
             failure      │                                 │  success
             threshold    ▼                                 │  threshold
             reached   ┌──────┐    timeout expires    ┌──────────┐
  ┌──────────────────► │ OPEN │ ──────────────────────► HALF-OPEN │
  │                    └──────┘                        └──────────┘
  │                       │                                 │
  │                       │  all calls fail fast            │ failure
  │                                                         │
  │                    ┌────────┐ ◄───────────────────────┘
  └────────────────────│ CLOSED │
        on failure     └────────┘
                          │
                    all calls pass
                       through
```

---

## States

### 1. Closed (Normal Operation)
- All requests pass through to the remote service
- The circuit breaker monitors and counts failures
- If the failure count exceeds the configured **threshold** within a given time window, the circuit **trips to Open**
- On success, the failure count resets

### 2. Open (Failing Fast)
- All requests **immediately fail** without calling the remote service
- An error or fallback response is returned to the caller instantly
- The circuit breaker starts a **timeout timer**
- Once the timer expires, the circuit transitions to **Half-Open**

### 3. Half-Open (Testing Recovery)
- A **limited number of test requests** are allowed through to the remote service
- If these requests **succeed**, the circuit transitions back to **Closed** — the service has recovered
- If these requests **fail**, the circuit transitions back to **Open** and resets the timeout timer

---

## Configuration Parameters

| Parameter            | Description                                                                  |
|----------------------|------------------------------------------------------------------------------|
| `failureThreshold`   | Number of failures before the circuit opens                                  |
| `recoveryTimeout`    | Duration the circuit stays open before transitioning to half-open            |
| `successThreshold`   | Number of successful calls in half-open state needed to close the circuit    |
| `monitoringWindow`   | Time window used to count failures (sliding or fixed window)                 |
| `fallback`           | Optional function or response to return when the circuit is open             |

Tuning these parameters correctly is critical. Values that are too aggressive can cause the circuit to trip on normal transient failures; values that are too lenient can allow cascading failures to propagate.

---

## When to Use

- Your application calls **remote services or APIs** that may become unavailable
- You want to **fail fast** and preserve system resources instead of waiting for timeouts
- You need to **prevent cascading failures** across a microservices architecture
- You want to provide a **graceful degradation** path or fallback behavior to users
- You need **automatic recovery** without manual intervention when a service comes back online

---

## When Not to Use

- For handling **local in-process exceptions** — the pattern is designed for remote calls over a network
- When failures are **expected and recoverable** within the same operation — use retry logic instead
- For operations that **must not be skipped** without a robust fallback, as the circuit breaker will reject requests when open
- In **simple, single-service** applications where the overhead of the pattern is not justified

---

## Considerations

- **Logging & Monitoring**: Always log state transitions (Closed → Open → Half-Open). Observability into circuit breaker state is essential for debugging and incident response.
- **Fallback Strategy**: Design a meaningful fallback — return cached data, a default response, or a user-friendly error message. Failing fast without a fallback can still degrade user experience significantly.
- **Concurrency**: In multi-threaded environments, the circuit breaker state must be managed in a thread-safe way to avoid race conditions during state transitions.
- **Per-Service Instances**: Use separate circuit breaker instances per downstream dependency. A shared instance can cause unrelated services to be incorrectly affected by a single failure.
- **Testing**: Simulate failures in staging environments to verify that circuit breakers open, remain open, and recover correctly under realistic load conditions.
- **Timeout Tuning**: Ensure that the `recoveryTimeout` is long enough for the downstream service to actually recover, but not so long that recovery takes an unnecessarily extended amount of time.

---

## Popular Libraries

| Language   | Library                        |
|------------|--------------------------------|
| Java       | Resilience4j, Hystrix          |
| .NET       | Polly                          |
| Node.js    | opossum                        |
| Python     | pybreaker, circuitbreaker      |
| Go         | gobreaker, sony/gobreaker      |

---

## Related Patterns

| Pattern             | Relationship                                                                                        |
|---------------------|-----------------------------------------------------------------------------------------------------|
| **Retry**           | Often used together — retry handles transient faults; circuit breaker stops retrying on sustained failures |
| **Bulkhead**        | Isolates resources to limit blast radius; complements circuit breaker in resilience strategies       |
| **Timeout**         | Should be combined with circuit breaker — a timeout triggers the failure count                      |
| **Fallback**        | Provides an alternative response when the circuit is open                                           |
| **Health Endpoint** | Circuit breaker state can be exposed as a health check endpoint for monitoring and alerting         |

---

## References

- Nygard, M. *Release It! Design and Deploy Production-Ready Software*. Pragmatic Bookshelf.
- [Martin Fowler — CircuitBreaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Microsoft Azure Architecture Patterns — Circuit Breaker](https://learn.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
- [Resilience4j Documentation](https://resilience4j.readme.io/docs/circuitbreaker)
