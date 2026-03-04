# Retry Pattern

## Table of Contents
- [Overview](#overview)
- [Problem](#problem)
- [Solution](#solution)
- [How It Works](#how-it-works)
- [Retry Strategies](#retry-strategies)
- [Configuration Parameters](#configuration-parameters)
- [When to Use](#when-to-use)
- [When Not to Use](#when-not-to-use)
- [Considerations](#considerations)
- [Popular Libraries](#popular-libraries)
- [Related Patterns](#related-patterns)
- [References](#references)

---

## Overview

The **Retry** pattern is a fault-tolerance design pattern that enables an application to handle transient failures by automatically re-attempting a failed operation a specified number of times before giving up. Rather than immediately surfacing an error to the caller, the application transparently retries the operation under the assumption that the failure is temporary and the next attempt may succeed.

This pattern is foundational in distributed systems, cloud-native architectures, and any environment where network instability, temporary resource unavailability, or brief service interruptions are expected.

---

## Problem

In distributed systems, calls to remote services, databases, or external APIs can fail for reasons that are temporary and self-correcting. These include:

- **Transient network faults** such as brief connectivity drops or packet loss
- **Service throttling** where a downstream service temporarily rejects requests due to load
- **Momentary resource unavailability** such as a database connection pool being exhausted briefly
- **Timeout spikes** caused by short-lived latency bursts in the network or service
- **Partial deployments** where a service is momentarily unavailable during a rolling update

These failures are not indicative of a deeper problem — they resolve on their own within milliseconds to a few seconds. Without a retry mechanism, these transient faults propagate as hard errors to the end user, even when a simple re-attempt would have succeeded.

---

## Solution

When an operation fails, instead of immediately returning an error, the application **waits for a brief period and tries the operation again**. This cycle repeats up to a configured maximum number of attempts. If all attempts are exhausted without success, the failure is surfaced to the caller.

```
  ┌────────┐        ┌─────────────────┐        ┌────────────────┐
  │ Caller │───────►│  Retry Handler  │───────►│ Remote Service │
  └────────┘        └─────────────────┘        └────────────────┘
                            │                          │
                      On failure                       │
                            │◄─────────────────────────┘
                            │
                    Wait (delay strategy)
                            │
                    Retry attempt (up to max)
                            │
                    On success → return result
                    On max attempts → throw error
```

---

## How It Works

1. The caller invokes an operation through the retry handler
2. If the operation **succeeds**, the result is returned immediately
3. If the operation **fails** with a retriable error:
   - The retry handler increments the attempt counter
   - It checks whether the maximum number of attempts has been reached
   - If not, it waits for the configured delay period (which may vary by strategy)
   - It then re-attempts the operation
4. If the operation **fails** with a non-retriable error (e.g. a 400 Bad Request), the error is surfaced immediately without retrying
5. If the **maximum attempts** are reached without success, the final error is returned to the caller

---

## Retry Strategies

The delay between retry attempts is a critical design decision. Different strategies suit different use cases.

### Immediate Retry
The operation is retried instantly with no delay. Suitable only for failures caused by extremely brief transient conditions such as a single dropped packet.

```
Attempt:  1 ──FAIL──► 2 ──FAIL──► 3 ──SUCCESS
Delay:           0ms        0ms
```

> ⚠️ Use sparingly. Immediate retries can amplify load on an already struggling service.

---

### Fixed Interval
A constant delay is applied between every retry attempt. Simple to configure and reason about, but does not adapt to sustained failures.

```
Attempt:  1 ──FAIL──► 2 ──FAIL──► 3 ──SUCCESS
Delay:          500ms       500ms
```

---

### Incremental (Linear Backoff)
The delay increases by a fixed amount with each subsequent attempt. Reduces pressure on the downstream service compared to fixed interval.

```
Attempt:  1 ──FAIL──► 2 ──FAIL──► 3 ──FAIL──► 4
Delay:          500ms      1000ms      1500ms
```

---

### Exponential Backoff
The delay doubles (or grows exponentially) with each retry attempt. This is the most widely recommended strategy in distributed systems as it naturally backs off load during sustained failures.

```
Attempt:  1 ──FAIL──► 2 ──FAIL──► 3 ──FAIL──► 4
Delay:          500ms      1000ms      2000ms
```

---

### Exponential Backoff with Jitter
A randomized offset (jitter) is added to the exponential delay. This prevents the **thundering herd problem**, where many clients that failed at the same time retry in synchronized waves, flooding the recovering service simultaneously.

```
Attempt:  1 ──FAIL──► 2 ──FAIL──► 3 ──FAIL──► 4
Delay:         ~530ms     ~1120ms     ~1870ms   (randomized)
```

> ✅ This is the recommended default strategy for most distributed systems.

---

## Configuration Parameters

| Parameter             | Description                                                                                   |
|-----------------------|-----------------------------------------------------------------------------------------------|
| `maxAttempts`         | Total number of attempts allowed, including the initial one                                   |
| `initialDelay`        | Delay before the first retry attempt                                                          |
| `maxDelay`            | Upper bound on the delay to prevent excessively long waits                                    |
| `backoffMultiplier`   | Factor by which the delay grows on each attempt (used in exponential backoff)                 |
| `jitter`              | Random variance added to the delay to avoid synchronized retries across clients               |
| `retryableExceptions` | The specific error types or status codes that should trigger a retry                          |
| `nonRetryableExceptions` | Errors that should immediately stop retrying (e.g. authentication errors, bad requests)  |
| `timeout`             | Maximum total time allowed across all retry attempts combined                                 |

---

## When to Use

- Operations that call **remote services or APIs** over a network that may experience transient faults
- Interactions with **cloud services** (storage, queues, databases) that enforce throttling or rate limits
- **Message processing pipelines** where delivery failures are expected and retryable
- Services with **rolling deployments** where brief unavailability is a normal part of operations
- Any scenario where **transient failures are expected** and the operation is safe to repeat

---

## When Not to Use

- When the operation is **not idempotent** — retrying a non-idempotent operation (such as charging a payment or sending an email) can result in duplicate side effects. Ensure idempotency before applying retries.
- When the failure is **not transient** — errors such as `404 Not Found`, `400 Bad Request`, or authentication failures indicate a logical problem that will not resolve on retry.
- When retrying would **amplify load** on an already overloaded service. In this case, combine the Retry pattern with the Circuit Breaker pattern to stop retrying when the service is sustainably unhealthy.
- When **strict latency requirements** exist and the cumulative delay of retry attempts would be unacceptable to the caller.
- When failures are **consistently occurring** — this is a signal of a systemic issue, not a transient one, and retrying will not help.

---

## Considerations

- **Idempotency**: Before applying retries to any operation, confirm that the operation is idempotent — that is, executing it multiple times produces the same result as executing it once. Retrying non-idempotent operations can lead to data corruption or duplicate actions.

- **Error Classification**: Not all errors should be retried. Distinguish between retriable errors (transient network faults, 503 Service Unavailable, 429 Too Many Requests) and non-retriable errors (400 Bad Request, 401 Unauthorized, 404 Not Found). Retrying non-retriable errors wastes resources.

- **Retry Budgets**: Define a maximum total time or attempt count for retries to prevent operations from hanging indefinitely. Always pair retries with a timeout.

- **Thundering Herd**: When many clients experience the same failure simultaneously and all retry at the same fixed interval, they can flood the recovering service with a synchronized burst of requests. Use jitter to randomize retry delays and spread the load.

- **Logging & Observability**: Log every retry attempt with context (attempt number, delay, error reason). This is essential for diagnosing systemic problems that may be masked by successful retries.

- **Interaction with Circuit Breaker**: Retries and circuit breakers are complementary. Retries handle brief, isolated transient faults. The circuit breaker stops retrying when a service is sustaining failures over a longer period. Without a circuit breaker, aggressive retry policies can worsen the load on an already failing service.

- **Upstream Awareness**: If a downstream service applies rate limiting (e.g. HTTP 429), respect the `Retry-After` header if provided rather than retrying on a fixed schedule.

---

## Popular Libraries

| Language   | Library                                  |
|------------|------------------------------------------|
| Java       | Resilience4j, Spring Retry, Guava Retryer |
| .NET       | Polly                                    |
| Node.js    | async-retry, p-retry, cockatiel          |
| Python     | tenacity, backoff, retry                 |
| Go         | go-retry, avast/retry-go                 |

---

## Related Patterns

| Pattern              | Relationship                                                                                                   |
|----------------------|----------------------------------------------------------------------------------------------------------------|
| **Circuit Breaker**  | Complements Retry — stops retrying when failures are sustained; prevents overloading a struggling service      |
| **Timeout**          | Should always accompany Retry — sets an upper bound on how long a single attempt or the entire retry cycle can take |
| **Bulkhead**         | Isolates retry attempts to prevent a single failing dependency from consuming all available threads or connections |
| **Fallback**         | Provides an alternative result when all retry attempts are exhausted                                           |
| **Idempotency Key**  | Enables safe retries for non-idempotent operations by deduplicating duplicate requests on the server side      |
| **Throttling**       | A downstream service applying throttling (429) is a common trigger for retry logic with backoff                |

---

## References

- Nygard, M. *Release It! Design and Deploy Production-Ready Software*. Pragmatic Bookshelf.
- [Microsoft Azure Architecture Patterns — Retry](https://learn.microsoft.com/en-us/azure/architecture/patterns/retry)
- [AWS — Exponential Backoff and Jitter](https://aws.amazon.com/blogs/architecture/exponential-backoff-and-jitter/)
- [Google Cloud — Retry Strategy Best Practices](https://cloud.google.com/storage/docs/retry-strategy)
- [Resilience4j Retry Documentation](https://resilience4j.readme.io/docs/retry)
