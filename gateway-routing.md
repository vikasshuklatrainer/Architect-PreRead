# Gateway Routing Pattern

Use a single endpoint to route requests to multiple services. This pattern is useful when you want to expose multiple services through a single endpoint and route requests to the appropriate service.

## Context and Problem

When a client needs to consume multiple services, maintaining a separate endpoint for each service and having the client manage each one is highly challenging. For example, an e-commerce application might provide multiple services such as search, reviews, shopping cart, payments, and order history.

Each service has a different API that the client must interact with, meaning the client must know every endpoint in order to connect to each service. If an endpoint changes or is updated, the client must be updated as well. If a service is refactored into two or more independent services, the code must be changed on both the server side and the client side.

## Solution

Place a gateway in front of a group of applications, services, or deployments. Use Layer 7 application routing to route requests to the appropriate instances.

With this pattern, the client application only needs to know about and communicate with a single endpoint. If services are consolidated or decomposed, the client does not need to be updated. The client can continue sending requests to the gateway — only the routing changes.

The gateway also allows you to abstract backend services from the client's perspective, enabling changes to backend services behind the gateway while keeping client calls simple. Client calls can be routed to any one or more services needed to handle the desired client behavior, allowing you to add, split, and reorganize services behind the gateway without modifying the client.

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/gateway-routing.png)

This pattern also helps with deployments, allowing you to manage how updates are rolled out to users. When a new version of a service is deployed, the old version can be run in parallel. You can use routing to control which version of the service is presented to users, giving you the flexibility to use different release strategies — whether incremental, parallel, or full rollout. Any issues discovered after deploying a new service can be quickly rolled back by simply changing the configuration at the gateway, without impacting the client.

## Issues and Considerations

* The gateway service may introduce a single point of failure. Ensure that availability requirements are considered at design time. Consider resiliency and fault tolerance during implementation.
* The gateway service can become a bottleneck. Ensure the gateway has sufficient performance to handle the load and can scale easily in line with expected growth.
* Perform load testing on the gateway to ensure you do not introduce cascading failures for your services.
* Gateway routing operates at Layer 7 and can be based on IP, port, header, or URL.

## When to Use This Pattern

Use this pattern in the following scenarios:

* A client needs to consume multiple services that can be accessed through a gateway.
* You want to simplify client applications by using a single endpoint.
* You need to route requests from an externally addressable endpoint to internal virtual endpoints, such as exposing ports on a virtual machine that point to a cluster's virtual IP address.

This pattern may not be suitable when your application is relatively simple and only has one or two services.

## Example

Using Nginx as a router, the following is a simple example of a server configuration file that routes application requests from different virtual directories to different backend machines.

```
server {
    listen 80;
    server_name domain.com;

    location /app1 {
        proxy_pass http://10.0.3.10:80;
    }

    location /app2 {
        proxy_pass http://10.0.3.20:80;
    }

    location /app3 {
        proxy_pass http://10.0.3.30:80;
    }
}
```

## Related Guidance

* [Backends for Frontends Pattern](patterns/backends-for-frontends.md)
* [Gateway Aggregation Pattern](patterns/gateway-aggregation.md)
* [Gateway Offloading Pattern](patterns/gateway-offloading.md)
