# Gatekeeper Pattern

Protect applications and services by using a dedicated host instance that acts as a proxy between clients and the application or service. It validates and sanitizes requests, and passes requests and data between them. This can provide an additional layer of security and limit the attack surface of the system.

### Context and Problem

Applications expose their functionality to clients by accepting and processing requests. In cloud-hosted scenarios, applications expose endpoints to which clients connect, typically including code that handles client requests. This code performs authentication and validation, handles some or all request processing, and may access storage and other services on behalf of the client.

If a malicious user is able to compromise the system and gain access to the application's hosting environment, the security mechanisms it uses (such as credentials and storage keys) and the services and data it accesses will be exposed. As a result, the malicious user can gain unrestricted access to sensitive information and other services.

## Solution

To minimize the risk of clients accessing sensitive information and services, decouple the host or task that exposes public endpoints from the code that processes requests and accesses storage. This can be achieved by using a facade or dedicated task that interacts with clients and then hands off requests (perhaps through a decoupled interface) to the host or task that will handle them. The diagram below provides an overview of this pattern.

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/gatekeeper-diagram.png)

The Gatekeeper pattern can be used to simply protect storage, or it can be used as a more comprehensive facade to protect all of the application's functionality. The important factors are:

* **Controlled Validation.** The gatekeeper validates all requests and rejects those that do not meet validation requirements.
* **Limited Risk and Exposure.** The gatekeeper does not have access to the credentials or keys used by the trusted host to gain access to storage and services. If the gatekeeper is compromised, an attacker will not be able to access these credentials or keys.
* **Appropriate Security.** The gatekeeper runs in a limited privilege mode, while the rest of the application runs in the full trust mode required to access storage and services. If the gatekeeper is compromised, it cannot directly access application services or data.

This pattern acts like a firewall in a typical network topology. It allows the gatekeeper to inspect requests and make a decision about whether to pass the request to a trusted host (sometimes called the key master) that performs the required task. This decision typically requires the gatekeeper to validate and sanitize the request content before passing it to the trusted host.

### Issues and Considerations

Consider the following points when deciding how to implement this pattern:

* Ensure that the trusted hosts the gatekeeper passes requests to only expose internal or protected endpoints, and connect only to the gatekeeper. Trusted hosts should not expose any external endpoints or interfaces.
* The gatekeeper must run in a limited privilege mode. This typically means running the gatekeeper and trusted hosts in separate hosted services or virtual machines.
* The gatekeeper should not perform any processing related to the application or service, or access any data. Its function is purely to validate and sanitize requests. Trusted hosts may need to perform additional validation on requests, but the core validation should be performed by the gatekeeper.
* Use a secure communication channel (HTTPS, SSL, or TLS) between the gatekeeper and trusted hosts or tasks where possible. However, some hosting environments do not support HTTPS on internal endpoints.
* Adding the extra layer to the application to implement the Gatekeeper pattern will likely have some impact on performance, due to the additional processing and network communication required.
* The gatekeeper instance could be a single point of failure. To minimize the impact of a failure, consider deploying additional instances and using an autoscaling mechanism to ensure the capacity to maintain availability.

## When to Use This Pattern

This pattern is suitable for the following scenarios:

* Applications that handle sensitive information, expose services that must be highly protected from malicious attacks, or perform mission-critical operations that should not be disrupted.
* Distributed applications where it is necessary to perform request validation separately from the main tasks, or to centralize this validation to simplify maintenance and management.

## Example

In a cloud-hosted scenario, the gatekeeper role or virtual machine can be decoupled by separating trusted roles and services in the application. This is done by using internal endpoints, queues, or storage as an intermediate communication mechanism. The diagram below illustrates the use of internal endpoints.

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/gatekeeper-endpoint.png)

## Related Patterns

When implementing the Gatekeeper pattern, the [Valet Key pattern](valet-key.md) may also be relevant. When communicating between the Gatekeeper and trusted roles, it is good practice to enhance security by using keys or tokens that restrict permissions for accessing resources. This describes how to use tokens or keys to provide clients with restricted direct access to a specific resource or service.
