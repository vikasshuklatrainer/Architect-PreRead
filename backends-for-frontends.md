# Backends for Frontends Pattern

Create separate backend services for specific frontend applications or interfaces. This pattern is useful when you want to avoid customizing a single backend to serve multiple interfaces.

## Context and Problem

Applications are often initially designed with a desktop web UI as the target. Typically, a backend service is developed in parallel to provide the features required by that UI. As the application's user base grows, a mobile app is developed and must interact with the same backend. This backend service then becomes a general-purpose backend that must simultaneously satisfy the needs of both the desktop and mobile interfaces.

However, mobile devices differ significantly from desktop browsers in terms of screen size, performance, and display constraints. As a result, the requirements that a mobile app places on the backend are also inconsistent with those of the desktop web UI.

These differences manifest as conflicting demands on the backend. The backend requires frequent and significant changes to simultaneously meet the needs of both the desktop web UI and the mobile app. Often, each interface is built by a different frontend team, which causes the backend to become a bottleneck in the development process. Conflicting update requirements, and the need to keep the backend working for both interfaces at the same time, can result in a great deal of effort being spent on this single deployable resource.

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/backend-for-frontend.png)

As development activity increasingly focuses on the backend service, a separate team may form to manage and maintain it. This ultimately leads to a disconnect between the interface and backend development teams, and places an increased burden on the backend team to balance the conflicting requirements from different UI teams. When one interface team requires changes to the backend, those changes must be validated by other interface teams before they can be integrated into the backend service.

## Solution

Create a separate backend for each user interface. Fine-tune the behavior and performance of each backend to best match the needs of its corresponding frontend environment, without worrying about affecting other frontend experiences.

![](https://docs.microsoft.com/en-us/azure/architecture/patterns/_images/backend-for-frontend-example.png)

Because each backend is tailored to a specific interface, it can be optimized exclusively for that interface. As a result, compared to a general-purpose backend trying to satisfy all types of interfaces, a dedicated backend will be smaller, simpler, and likely faster. Each interface team has autonomy over their own backend without depending on a centralized backend development team. This gives interface teams flexibility in their choice of backend language, release cadence, work prioritization, and feature integration.

## Issues and Considerations

* Consider how many backends need to be deployed.
* If different interfaces (such as mobile clients) will be making the same requests, consider whether it is necessary to implement a backend for each interface, or whether a single backend would suffice.

* Code duplication across services is very likely to occur when implementing this pattern.
* Frontend-specific backend services should only contain client-specific logic and behavior. General-purpose logic and other shared functionality should be managed elsewhere in your application.

* Think about how this pattern should be reflected in the responsibilities of development teams.
* Consider how long it will take to implement this pattern. Will the effort of building new backends, while continuing to support the existing general-purpose backend, introduce technical debt?

## When to Use This Pattern

Use this pattern in the following scenarios:

* A shared or general-purpose backend service requires significant overhead to maintain.
* You want to optimize the backend for the needs of a specific client interface.
* Customizations are made to a general-purpose backend to accommodate the requirements of multiple interfaces.
* An alternative language is better suited for the backend of a particular user interface.

This pattern may not be suitable when:

* Interfaces make the same or similar requests to the backend.
* Only one interface interacts with the backend.

## Related Guidance

* [Gateway Aggregation Pattern](patterns/gateway-aggregation.md)
* [Gateway Offloading Pattern](patterns/gateway-offloading.md)
* [Gateway Routing Pattern](patterns/gateway-routing.md)
