# Module Architecture

## Overview
The Hospital Management System is organized into bounded modules that reflect the primary business capabilities of the platform. Each module exposes a well-defined API layer and, where appropriate, a background job processor that works with shared infrastructure (messaging, storage, observability). Cross-cutting concerns—authentication, authorization, audit logging, and notification dispatch—are implemented as platform services consumed by all modules.

## Module Boundaries
### Patient Management
* **Scope**: Registration, demographic maintenance, insurance information, patient portal profile management.
* **Interfaces**: REST/GraphQL APIs for CRUD operations, bulk import endpoint for demographic data, event stream for patient updates.
* **Shared Dependencies**: Identity provider, document storage for consents, messaging bus for triggering care-plan workflows.

### Clinical Services
* **Scope**: Encounter documentation, orders, care plans, vitals, diagnostic results ingestion.
* **Interfaces**: Clinical documentation API, FHIR-compliant ingestion endpoints, subscription to device telemetry topics.
* **Shared Dependencies**: Patient Management for demographics, Scheduling for appointment context, Reporting for analytics feeds.

### Scheduling
* **Scope**: Provider calendars, appointment booking, check-in/check-out workflows, resource allocation (rooms, equipment).
* **Interfaces**: Calendar synchronization API, appointment search service, webhook notifications for schedule changes.
* **Shared Dependencies**: Patient Management for contact details, Billing for pre-authorization checks.

### Billing & Claims
* **Scope**: Charge capture, insurance verification, claim submission, payment posting, financial reporting.
* **Interfaces**: Claim submission gateway, invoice API, integration connectors for payers and clearinghouses.
* **Shared Dependencies**: Patient Management for insurance data, Clinical Services for procedure codes, Reporting for revenue dashboards.

### Inventory & Pharmacy
* **Scope**: Medication formulary, stock levels, procurement, dispensing workflows, recall tracking.
* **Interfaces**: Inventory reconciliation API, dispensing queue service, external supplier integrations.
* **Shared Dependencies**: Clinical Services for medication orders, Billing & Claims for charge codes, Notification service for recall alerts.

### Reporting & Analytics
* **Scope**: Operational dashboards, regulatory reporting, predictive analytics, data warehouse exports.
* **Interfaces**: Data mart SQL endpoints, scheduled report generator, BI tool connectors.
* **Shared Dependencies**: Consumes event streams from all modules, uses platform data lake and ML feature store.

## Service Responsibilities
* **API Gateway**: Request routing, authentication, rate limiting, schema validation.
* **Identity & Access Management (IAM)**: Single sign-on, role-based permissions, multi-factor enforcement, audit trails.
* **Notification Service**: Email/SMS/push delivery, templating, preference management.
* **Integration Hub**: HL7/FHIR translators, external system connectors, queuing for outbound interfaces.
* **Observability Stack**: Centralized logging, distributed tracing, metrics aggregation, anomaly detection alerts.
* **Data Platform**: Managed PostgreSQL clusters, object storage, streaming infrastructure (Kafka), ETL orchestration.

## Data Retention Policies
Retention policies apply per module and are enforced through lifecycle management jobs and database partitioning strategies.

| Module | Data Types | Retention Period | Archival Strategy | Deletion Notes |
| --- | --- | --- | --- | --- |
| Patient Management | Demographics, consents, insurance documents | Active + 10 years | Cold storage archive of inactive records every quarter | Automatic purge of records 24 months after patient-initiated deletion request, except legal hold cases |
| Clinical Services | Encounter notes, diagnostic results, telemetry | Encounter close + 15 years | Partitioned clinical data lake with yearly snapshots | PHI redaction applied when exporting beyond retention horizon; research datasets anonymized |
| Scheduling | Appointments, resource allocation logs | Event date + 5 years | Monthly archival to warm storage with indexed search | Removes stale waitlist entries after 18 months |
| Billing & Claims | Invoices, remittances, payment history | Fiscal close + 7 years | Encrypted archive buckets with write-once storage | PII minimization after financial dispute window (24 months) closes |
| Inventory & Pharmacy | Stock transactions, dispensing records, supplier contracts | Transaction date + 3 years | Rolling archive into supply-chain warehouse DB | Controlled substance logs retained 10 years per regulation |
| Reporting & Analytics | Aggregated metrics, derived datasets, machine-learning features | Source-dependent (max 7 years) | Versioned parquet storage with retention tags | Differential purges ensure aggregates remain compliant with source module policies |

Automated retention jobs run nightly, emitting metrics to the observability stack and raising alerts for exceptions or legal hold overrides.
