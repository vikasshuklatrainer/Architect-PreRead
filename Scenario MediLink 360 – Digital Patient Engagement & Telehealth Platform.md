
# Scenario: MediLink 360 – Digital Patient Engagement & Telehealth Platform

## Business Context

A mid-size hospital chain (15 hospitals + 40 clinics) wants a single digital platform called **MediLink 360** where patients can:

- Book / manage appointments (in-person + teleconsultation)  
- View prescriptions, lab reports, discharge summaries  
- Chat asynchronously with care teams  
- Receive reminders for medication, follow-ups, vaccines  
- Pay bills online and download invoices  

## Integrations

The platform must integrate with:

- Existing on-prem **Hospital Information System (HIS)**
- Third-party **Lab Information System (LIS)**
- Existing **Insurance / TPA portals**
- **SMS / Email / Push notification providers**

## Non-Functional Goals

- Handle **20,000 concurrent users**
- Ensure **99.9% uptime**
- Maintain strong **patient data protection** (HIPAA-like, local health regulations)
- Provide full **auditability & traceability** of access to health records


# MediLink 360 – C4 Architecture Scenario

---

## C1 – System Context Diagram

### Primary System
**MediLink 360 Platform** – The main system being architected.

### People (Actors)
- **Patient** – Uses mobile app / web app to access services  
- **Doctor** – Uses portal to view appointments, patient records, and conduct teleconsultations  
- **Nurse / Care Coordinator** – Manages patient schedules, reminders, and follow-up tasks  
- **Hospital Admin / IT Ops** – Manages configurations, users, and monitors system health  
- **Insurance Assessor (External)** – Accesses claim information via portal/API  

### External Systems
- **Hospital Information System (HIS)** – Master clinical data (admissions, diagnoses, prescriptions)  
- **Lab Information System (LIS)** – Lab orders and results  
- **Insurance / TPA Portals** – Policy validation, pre-authorizations, claim status  
- **Payment Gateway** – Processes online payments  
- **Notification Gateway** – SMS / Email / Push notifications  
- **Identity Provider (IdP)** – Corporate IdP / National Digital ID for SSO  

This forms the basis for a classic C1 System Context Diagram with:
- Users around the central system  
- External systems integrated with MediLink 360  

---

## C2 – Container Diagram

Break **MediLink 360** into containers:

### Frontend Applications
- **Web App (SPA)**  
  - Angular / React / Vue  
  - Used by Patients, Doctors, Admins via browser  

- **Mobile App (iOS/Android)**  
  - Native / Flutter / React Native  
  - Used by Patients & Doctors  

### API Layer
- **API Gateway**  
  - Single entry point  
  - Handles routing, rate limiting, authN/authZ, API keys, JWT  

### Backend Services
- **Patient Profile Service** – Manages patient demographics, preferences, consents  
- **Appointment Service** – Schedules, cancels, reschedules, and syncs with HIS  
- **Teleconsultation Service** – Manages video sessions and consultation lifecycle  
- **Clinical Data Proxy Service** – Secure interface to HIS and LIS  
- **Notification Service** – Orchestrates SMS, Email, Push via Notification Gateway  
- **Billing & Payments Service** – Manages invoices and payment flow  
- **Insurance Integration Service** – Handles policy validation, claims, and pre-authorization  
- **Audit & Logging Service** – Stores security and compliance events  

---

### Datastores
- **Operational DB (Relational)** – App data (appointments, sessions, user preferences)  
- **Audit Log Store (Append-only, WORM-like)** – Compliance and security logs  
- **Cache (Redis)** – Session data and frequently used reference data  
- **Object Storage** – Documents, reports, invoices, recordings metadata  

---

### Integration Layer
- **Message Broker / Event Bus**  
  - Async communication between services  
  - Example: Appointment created → Notify HIS → Send Reminder  

- **Integration Adapters**  
  - For HIS, LIS, Insurance APIs  
  - Protocols: SOAP / REST / HL7 / FHIR  

---

### Cross-Cutting Components
- **Identity & Access Management (IAM)**  
  - OAuth2 / OIDC, RBAC  
  - Integrated with external Identity Provider  

- **Monitoring & Observability Stack**  
  - Metrics, logs, distributed tracing  

- **API Management Portal**  
  - Exposes APIs to external partners (e.g., Insurance platforms)  

This C2 prepares a full container-level architecture view with containers, data stores, integrations, and cross-cutting concerns.

---

## C3 – Component Diagram (Appointment Service)

Inside the **Appointment Service**:

### Components
- **AppointmentController**  
  REST endpoints: `/appointments`, `/availability`, `/cancel`, etc.  

- **Scheduling Engine**  
  Applies scheduling business rules (availability, clinic hours, teleconsult vs in-person, time zones)  

- **Availability Provider**  
  Aggregates doctor availability from HIS, Clinic Config DB, and local overrides  

- **Appointment Repository**  
  CRUD operations on the Appointments table  

- **Notification Orchestrator**  
  Publishes appointment lifecycle events to Message Broker for downstream Notification Service  

- **Audit Logger**  
  Sends audit events to Audit & Logging Service  

---

### Component Interactions
- AppointmentController → Scheduling Engine → Availability Provider  
- AppointmentController → Appointment Repository  
- AppointmentController → Notification Orchestrator  
- All components → Audit Logger  

This same component-level style can be applied to other services like Teleconsultation service or Notification service.

---

## C4 – Code-Level Diagram (Optional)

For a deeper code-level view (e.g., Scheduling Engine):

### Core Classes
- `ScheduleCalculator`  
- `DoctorAvailabilityRules`  
- `HolidayCalendarService`  
- `SlotGenerator`  

### Interfaces
- `IHISAvailabilityClient` – Fetches schedules from HIS  
- `ILocalOverrideRepository` – Fetches doctor availability overrides  

### Design Patterns
- **Strategy Pattern** – For different scheduling policies  
- **Factory Pattern** – For selecting availability providers dynamically  

This level is ideal for detailed technical and developer-focused architecture representations.

---







# C1 – System Context Diagram (C4Context)
```plaintext
You are an expert software architect. Generate a C4 System Context diagram for the following system in Mermaid.js C4 syntax.

System name: MediLink 360 – Digital Patient Engagement & Telehealth Platform

Description:
A mid-size hospital chain (15 hospitals + 40 clinics) wants a single digital platform where patients can:
- Book/manage appointments (in-person + teleconsultation)
- View prescriptions, lab reports, discharge summaries
- Chat asynchronously with care teams
- Receive reminders for medication, follow-ups, vaccines
- Pay bills online and download invoices

The platform integrates with:
- Hospital Information System (HIS)
- Lab Information System (LIS)
- Insurance/TPA portals
- Payment Gateway
- Notification Gateway (SMS, Email, Push)
- Identity Provider (IdP)

Actors (People):
- Patient
- Doctor
- Nurse / Care Coordinator
- Hospital Admin / IT Ops
- Insurance Assessor

External systems:
- HIS, LIS, Insurance Portals, Payment Gateway, Notification Gateway, Identity Provider

Use Mermaid.js C4Context syntax.
- Use Person, System, System_Ext
- Name the central system MediLink360
- Include all relationships using Rel(...)
- Add a title

IMPORTANT:
- Respond ONLY with a ```mermaid``` code block.
- Do NOT include any explanation outside the diagram.
```

# Prompt 2: C2 – Container Diagram (Mermaid C4)
```plaintext
You are an expert software architect. Generate a C4 Container diagram in Mermaid.js C4 syntax for the system below.

System: MediLink 360 – Digital Patient Engagement & Telehealth Platform

Containers:
- Web App (SPA)
- Mobile App (iOS/Android)
- API Gateway
- Patient Profile Service
- Appointment Service
- Teleconsultation Service
- Clinical Data Proxy Service
- Notification Service
- Billing & Payments Service
- Insurance Integration Service
- Audit & Logging Service

Datastores:
- Operational Database (Relational)
- Audit Log Store (Append-only)
- Cache (Redis)
- Object Storage

Integration:
- Message Broker / Event Bus
- Identity & Access Management (IAM) – OAuth2/OIDC, RBAC

External actors & systems:
Patient, Doctor, Nurse, Hospital Admin, Insurance Assessor
HIS, LIS, Insurance Portals, Payment Gateway, Notification Gateway, Identity Provider

Instructions:
- Use Mermaid.js C4Container syntax
- Wrap everything in System_Boundary(MediLink360, "MediLink 360 Platform")
- Use Container, ContainerDb, Person, System_Ext
- Show key relationships with Rel(...)
  - Apps → API Gateway
  - API Gateway → Services
  - Services → Databases
  - Services → External systems
- Add a title

IMPORTANT:
- Respond ONLY with a ```mermaid``` code block.
- No explanation text.
```

# Prompt 3: C3 – Component Diagram (Appointment Service)
```plaintext
You are an expert software architect. Generate a C4 Component diagram in Mermaid.js C4 syntax for the Appointment Service of MediLink 360.

Container: Appointment Service

Internal components:
- AppointmentController (REST endpoints)
- Scheduling Engine (business rules)
- Availability Provider (aggregates data from HIS + clinic configs)
- Appointment Repository (CRUD for Appointments DB)
- Notification Orchestrator (publishes appointment events)
- Audit Logger (security/compliance logging)

External dependencies:
- Clinical Data Proxy Service
- Notification Service
- Message Broker / Event Bus
- Audit & Logging Service
- Operational Database

Instructions:
- Use Container_Boundary for Appointment Service
- Use Component for internal components
- Use ContainerDb or System_Ext for dependencies
- Show relationships using Rel(...)
  - Controller → Engine → Availability Provider
  - Controller → Repository
  - Controller → Notification Orchestrator
  - All components → Audit Logger
  - Provider → Clinical Data Proxy
  - Orchestrator → Message Broker
  - Repository → Operational Database

IMPORTANT:
- Respond ONLY with a ```mermaid``` code block.
- Do NOT include any explanation.
```
# Prompt 4: C4 – Code Level (Scheduling Engine)
```plaintext
You are an expert software architect. Create a Code-Level C4 view using Mermaid.js classDiagram for the Scheduling Engine component.

Context:
The Scheduling Engine is part of the Appointment Service in MediLink 360 and calculates available slots for doctors and patients.

Model the following classes:
- SchedulingEngine
- ScheduleCalculator
- DoctorAvailabilityRules
- HolidayCalendarService
- SlotGenerator

Interfaces:
- IHISAvailabilityClient
- ILocalOverrideRepository

Relationships:
- SchedulingEngine uses ScheduleCalculator
- ScheduleCalculator uses DoctorAvailabilityRules, HolidayCalendarService, SlotGenerator
- ScheduleCalculator depends on IHISAvailabilityClient and ILocalOverrideRepository

Add 2–3 example methods per class.

Instructions:
- Use Mermaid.js classDiagram syntax
- Show associations/arrows clearly
- Focus on structure and dependencies only

IMPORTANT:
- Respond ONLY with a ```mermaid``` code block.
- Do NOT include any explanation.
```

# Generic C4 Prompt (Reusable for Any System)
You are an expert in C4 architecture and Mermaid.js.

Task:
Generate a C4 [CONTEXT | CONTAINER | COMPONENT | CODE] diagram in Mermaid syntax.

Requirements:
- Use C4Context, C4Container, or C4Component
- If CODE level, use classDiagram
- Use Person, System, Container, Component, System_Ext, ContainerDb accordingly
- Show all key relationships using Rel(...)
- Add a title
- Respond ONLY in ```mermaid``` code block format

System description:
[INSERT YOUR SCENARIO HERE]


