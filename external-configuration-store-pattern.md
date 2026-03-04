# External Configuration Store Pattern

## Table of Contents
- [Overview](#overview)
- [Problem](#problem)
- [Solution](#solution)
- [How It Works](#how-it-works)
- [Structure](#structure)
- [Configuration Store Types](#configuration-store-types)
- [Configuration Scopes & Hierarchy](#configuration-scopes--hierarchy)
- [Configuration Delivery Models](#configuration-delivery-models)
- [Configuration Parameters](#configuration-parameters)
- [When to Use](#when-to-use)
- [When Not to Use](#when-not-to-use)
- [Considerations](#considerations)
- [Popular Tools & Services](#popular-tools--services)
- [Related Patterns](#related-patterns)
- [References](#references)

---

## Overview

The **External Configuration Store** pattern is an operational design pattern that moves application configuration data out of the application deployment package and into a centralized, external location. Rather than bundling configuration values directly with the application — in files, environment variables baked into images, or hardcoded constants — all configuration is stored, managed, and retrieved from a dedicated external store.

This pattern is a cornerstone of cloud-native and twelve-factor application design, enabling configuration to change independently of the application code and deployment lifecycle.

---

## Problem

In traditional application deployments, configuration is often tightly coupled to the application itself — embedded in property files, packaged inside container images, or scattered across individual server environments. This approach introduces several significant challenges:

- **Configuration drift** across environments (development, staging, production) as each environment maintains its own copy of configuration files that can fall out of sync over time
- **Redeployment required for config changes** — updating a single configuration value forces a full rebuild and redeployment of the application, introducing unnecessary risk and downtime
- **Secrets exposure** — sensitive values such as database passwords, API keys, and tokens are often committed to source control alongside application code, creating security vulnerabilities
- **Lack of auditability** — there is no centralized record of who changed a configuration value, when it was changed, or what it was changed from
- **Scaling difficulties** — in distributed systems with many service instances, updating configuration consistently across all running instances is complex and error-prone
- **No dynamic updates** — applications must be restarted to pick up configuration changes, even for values that could safely be changed at runtime

---

## Solution

Extract all configuration data from the application and store it in a **dedicated external configuration store**. The application reads its configuration from this store at startup and, optionally, subscribes to configuration change notifications to receive updates at runtime without requiring a restart.

```
  ┌─────────────────────────────────────────────────────────┐
  │                External Configuration Store              │
  │                                                         │
  │   [ dev/* ]   [ staging/* ]   [ production/* ]         │
  │   [ service-a/* ]  [ service-b/* ]  [ shared/* ]       │
  └──────────────────────────┬──────────────────────────────┘
                             │  read / subscribe
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐
   │  Service A  │   │  Service B  │   │  Service C  │
   │  Instance 1 │   │  Instance 1 │   │  Instance 1 │
   └─────────────┘   └─────────────┘   └─────────────┘
          │
   ┌─────────────┐
   │  Service A  │
   │  Instance 2 │
   └─────────────┘
```

All service instances — regardless of how many are running — read from the same authoritative configuration source, ensuring consistency across the entire system.

---

## How It Works

1. At startup, the application connects to the external configuration store and fetches the configuration values relevant to its environment and service identity
2. The application uses these values in place of any local configuration files or hardcoded constants
3. Optionally, the application registers a **listener or subscription** with the configuration store to be notified of changes in real time
4. When a configuration value is updated in the store, the store **pushes a notification** to all subscribed service instances, or the service **polls** the store at a regular interval to detect changes
5. The application applies the updated configuration — either immediately for dynamic values, or on next restart for values that require it
6. All changes to configuration are recorded with a timestamp, the identity of the actor who made the change, and the previous and new values, providing a full **audit trail**

---

## Structure

```
┌────────────────────────────────────────────────────────────────┐
│                     Configuration Store                        │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  Key-Value   │  │   Versioning │  │   Access Control     │ │
│  │    Store     │  │   & History  │  │   (RBAC / Policies)  │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
│                                                                │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────────────┐ │
│  │  Namespacing │  │  Change      │  │   Secrets            │ │
│  │  & Scoping   │  │  Notification│  │   Management         │ │
│  └──────────────┘  └──────────────┘  └──────────────────────┘ │
└────────────────────────────────────────────────────────────────┘
          ▲                    │
          │ write              │ read / subscribe
   ┌──────────────┐    ┌───────────────────┐
   │  Operators / │    │   Applications /  │
   │  CI/CD Tools │    │   Service Instances│
   └──────────────┘    └───────────────────┘
```

---

## Configuration Store Types

### Key-Value Store
The most common form. Configuration is stored as a flat or hierarchical collection of key-value pairs. Keys are typically namespaced using a path-like convention (e.g. `production/service-a/database/timeout`). Simple, fast, and widely supported.

### Hierarchical / Tree-Based Store
Configuration is organized as a tree of nodes, each of which can hold a value and have child nodes. This maps naturally to the nested structure of most application configuration. Changes can be watched at any level of the hierarchy.

### Secret Store
A specialized store designed specifically for sensitive configuration values such as passwords, API keys, certificates, and tokens. Secret stores provide enhanced security through encryption at rest and in transit, fine-grained access policies, dynamic secret generation, and automatic secret rotation.

### Distributed Configuration Store
A strongly consistent, highly available store designed for distributed systems. Typically built on consensus algorithms (such as Raft or Paxos) to guarantee that all service instances see the same configuration values even under network partitions or node failures.

---

## Configuration Scopes & Hierarchy

Configuration values are typically organized into scopes that allow more specific values to override more general ones. A common hierarchy is:

```
Global / Shared
    └── Environment (development / staging / production)
            └── Service / Application
                    └── Instance / Region
```

This allows shared values (such as logging standards or common feature flags) to be defined once at the global level, while environment-specific or service-specific overrides are applied at lower levels. The most specific scope wins.

---

## Configuration Delivery Models

### Pull Model (Polling)
The application periodically queries the configuration store at a configured interval to check for updates. Simple to implement and does not require a persistent connection to the store. The trade-off is that there is always a delay between a configuration change being made and the application detecting it, proportional to the polling interval.

### Push Model (Event-Driven / Subscription)
The application subscribes to change notifications from the configuration store. When a value changes, the store immediately pushes the update to all subscribed listeners. This enables near-real-time configuration updates with minimal latency but requires the application to maintain a persistent connection to the store.

### Hybrid Model
Many production systems combine both approaches — using push notifications for low-latency updates during normal operation, and falling back to periodic polling as a safety net to catch any missed notifications.

---

## Configuration Parameters

| Parameter                  | Description                                                                                  |
|----------------------------|----------------------------------------------------------------------------------------------|
| `storeEndpoint`            | The address of the external configuration store                                              |
| `namespace` / `prefix`     | The scope or path prefix used to isolate configuration for this environment and service      |
| `pollingInterval`          | How frequently the application polls the store for updates (in pull model)                   |
| `cacheEnabled`             | Whether the application caches configuration values locally to reduce store round trips      |
| `cacheTTL`                 | How long locally cached configuration values remain valid before being refreshed             |
| `fallbackEnabled`          | Whether the application falls back to local configuration if the store is unreachable        |
| `accessCredentials`        | Authentication credentials used by the application to connect to the store                   |
| `refreshOnChange`          | Whether certain configuration changes should trigger an application reload or restart        |

---

## When to Use

- Applications deployed across **multiple environments** (development, staging, production) that require different configuration values per environment
- **Microservices architectures** where many services need a consistent, centralized way to manage and access configuration
- Systems that require **dynamic configuration updates** at runtime without restarting or redeploying services
- Applications that need to **manage secrets securely** and prevent sensitive values from being committed to source control
- Environments that require **configuration auditability** — tracking who changed what and when for compliance or operational purposes
- **Horizontally scaled services** with many running instances that must all share the same configuration values consistently
- Teams adopting **GitOps or CI/CD pipelines** where configuration changes should go through a managed, reviewable process

---

## When Not to Use

- **Simple, single-instance applications** with minimal configuration that does not change between environments — the operational overhead of an external store is not justified
- When the **configuration store itself becomes a single point of failure** and no fallback strategy or high-availability setup is in place
- When the team **lacks the operational maturity** to manage, secure, and monitor an additional infrastructure component
- For **extremely latency-sensitive startup paths** where the overhead of fetching configuration from a remote store at boot time is unacceptable without local caching
- When **all configuration is truly static** and tied to the application version — in this case, embedding it in the deployment artifact is simpler and safer

---

## Considerations

- **Availability & Resilience**: The configuration store must be treated as a critical infrastructure component. If the store becomes unavailable and the application cannot read its configuration, service startup or runtime behavior may be severely impacted. Always implement a **local cache or fallback** to gracefully handle store unavailability.

- **Security**: The configuration store often holds sensitive values. Enforce **strict access controls** (RBAC), encrypt all data in transit and at rest, rotate credentials regularly, and audit all access. Use a dedicated secret store for sensitive values rather than mixing them with general configuration.

- **Versioning & Rollback**: Store configuration changes with version history. This enables rollback to a previous configuration state when a bad configuration change causes a service incident — without requiring a full code deployment.

- **Consistency**: In distributed systems, there may be a brief window during a configuration update where different service instances are running with different configuration values. Design services to tolerate this **temporary inconsistency**, especially for configuration that affects cross-service behavior.

- **Change Management**: Not all configuration changes are safe to apply at runtime. Some values (e.g. server port, TLS certificate paths) may require a service restart to take effect. Document clearly which values are dynamically reloadable and which require a restart, and enforce this in the deployment process.

- **Bootstrapping Problem**: The application needs credentials or connection details to reach the configuration store in the first place. This bootstrapping configuration (the config needed to get config) must be managed carefully — typically through environment variables, a minimal local file, or a platform-provided identity mechanism such as IAM roles or workload identity.

- **Observability**: Monitor the configuration store for availability, latency, and change events. Alert on unexpected configuration changes, failed reads, or access control violations.

- **Namespace Discipline**: Establish a clear and consistent naming convention for configuration keys from the start. Poor naming conventions lead to confusion, accidental overwrites, and difficulty navigating large configuration trees.

---

## Popular Tools & Services

| Tool / Service                        | Type                          | Notable Features                                              |
|---------------------------------------|-------------------------------|---------------------------------------------------------------|
| **HashiCorp Consul**                  | Key-Value + Service Discovery | Health checking, ACLs, multi-datacenter support               |
| **HashiCorp Vault**                   | Secret Store                  | Dynamic secrets, lease management, encryption as a service    |
| **AWS Systems Manager Parameter Store** | Key-Value + Secrets         | Native AWS IAM integration, SecureString for secrets          |
| **AWS Secrets Manager**               | Secret Store                  | Automatic rotation, fine-grained IAM policies                 |
| **Azure App Configuration**           | Key-Value + Feature Flags     | Hierarchical keys, snapshot support, feature management       |
| **Azure Key Vault**                   | Secret Store                  | HSM-backed keys, certificate management, RBAC                 |
| **Google Cloud Secret Manager**       | Secret Store                  | Versioning, IAM-based access, audit logging                   |
| **etcd**                              | Distributed Key-Value Store   | Strong consistency (Raft), used as Kubernetes backing store   |
| **Spring Cloud Config Server**        | Config Server                 | Git-backed configuration, environment-based profiles          |
| **Kubernetes ConfigMaps & Secrets**   | Platform-Native               | Native Kubernetes integration, volume or env var mounting     |

---

## Related Patterns

| Pattern                        | Relationship                                                                                                     |
|--------------------------------|------------------------------------------------------------------------------------------------------------------|
| **Strangler Fig**              | When migrating legacy applications, external configuration enables incremental config extraction without full rewrites |
| **Sidecar**                    | A sidecar agent can handle configuration retrieval and injection on behalf of the main application container     |
| **Health Endpoint Monitoring** | Configuration store availability can be exposed as part of an application's health check                         |
| **Feature Flags**              | Often implemented on top of an external configuration store, enabling runtime toggling of features               |
| **Circuit Breaker**            | Configuration store unavailability should trigger circuit breaker logic to fall back to cached configuration     |
| **Secrets Management**         | A specialized application of this pattern focused exclusively on sensitive configuration values                  |

---

## References

- [Microsoft Azure Architecture Patterns — External Configuration Store](https://learn.microsoft.com/en-us/azure/architecture/patterns/external-configuration-store)
- [The Twelve-Factor App — Config](https://12factor.net/config)
- [HashiCorp Consul Documentation](https://developer.hashicorp.com/consul/docs)
- [HashiCorp Vault Documentation](https://developer.hashicorp.com/vault/docs)
- [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [Azure App Configuration Documentation](https://learn.microsoft.com/en-us/azure/azure-app-configuration/overview)
