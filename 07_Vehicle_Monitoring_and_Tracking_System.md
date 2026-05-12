# Vehicle Monitoring and Tracking System

## Objective

Build an end-to-end telematics platform — from on-vehicle data-capture units through to a multi-tenant web application — that ingests, stores, analyses, and visualises vehicle data in real time and after the fact. The platform serves both **fleet operators** (managing real vehicles in service) and **engineering teams** (consuming telemetry from development and test vehicles, including bench rigs and on-vehicle integrations).

Unlike the BMS and VCU tracks where architecture decisions are largely fixed, this track deliberately leaves the major architecture and technology decisions to the team. The document specifies **what the system must be capable of doing** and **what decisions the team owns**, not how to build it. The intent is that any well-structured implementation matching the capability bar and answering the listed decisions is a valid v1.

## Product Scope

- **On-vehicle telematics unit** — captures parameters from the vehicle's buses, sensors, and GPS, and forwards them upstream.
- **Backend services** — ingestion, storage, stream processing, query, alerting, and API exposure.
- **Frontend application** — live dashboards, historical analysis, fleet and vehicle management, tenant administration.
- **Multi-tenant operation** — isolated data and access per tenant, with the security and audit posture a multi-tenant deployment requires.
- **Bench data path** — the mockup rig and BMS testing benches feed the same backend as the on-vehicle units, so engineering and field telemetry land in one platform.

## Capability Examples

A system in this class is typically capable of the following. The team owns which subset is in v1 vs the roadmap, and how each capability is implemented.

### Real-Time Monitoring

- Live vehicle location on a map, status, and key parameters streaming with sub-second to seconds-level freshness.
- Live parameter dashboards (SOC, speed, motor temperature, voltage, current, faults) per vehicle and across fleets.
- Live alert generation when parameters cross configured thresholds, faults are reported, or safety events occur.
- Geofence violation, route deviation, and unauthorised-use alerts.
- Operator-behaviour signals — harsh braking, speeding, hard cornering, idle-time monitoring.

### Historical Analysis

- Trip reconstruction — route taken, speed profile, energy consumption, events along the way.
- Drive-cycle analysis — frequency, intensity, conditions, duration, and how they trend over time.
- Energy and range analytics — kWh per km, efficiency by route, driver, season, ambient temperature.
- Battery state-of-health trending — capacity fade, internal-resistance drift, cell-imbalance evolution.
- Comparative analysis — vehicle-to-vehicle, driver-to-driver, route-to-route, time-window-to-time-window.

### Fleet and Asset Management

- Vehicle inventory, status, and current location across the fleet.
- Maintenance scheduling driven by usage, condition signals, and elapsed time.
- Driver and operator assignment, access control, and shift tracking.
- Charging coordination — which vehicle to charge when, where, and at what energy cost.
- Vehicle commissioning, decommissioning, and configuration workflows.

### Remote Operations and Diagnostics

- Remote diagnostic readout — DTCs, fault history, parameter snapshots — without physical access to the vehicle.
- Remote configuration changes within the safety and security envelope the VCU exposes.
- OTA firmware-update coordination — staged rollouts, version tracking, rollback orchestration.
- Service-event tracking — when which vehicle was serviced for what.

### Anomaly Detection and Predictive Insights

- Detect unusual patterns in vehicle behaviour, sensor readings, or driver behaviour.
- Predictive-maintenance signals — surfacing likely failures before they happen on battery, motor, drivetrain, or auxiliary systems.
- Safety-event detection — accidents, near-misses, rollover, panic stops.
- Fraud detection — mileage tampering, charging fraud, unauthorised access.

### Multi-Tenant Operations

- Tenant administration — onboarding, decommissioning, user provisioning, role assignment.
- Tenant-scoped APIs — every API call carries tenant identity; data is partitioned and isolated.
- Audit trail — every administrative action and every data access logged for compliance review.
- Tenant-specific configurations — dashboards, alert thresholds, vehicle groupings, retention policies.

### Reporting and Integration

- Tenant-facing dashboards and report exports (CSV, PDF, scheduled emails).
- Programmable API for tenant integrations — downstream analytics, ERP/CRM systems, partner platforms.
- Regulatory and compliance reporting where the deployment requires it.
- Data-export paths for offline analysis.

### Bench and Engineering Telemetry

- Ingest data from the mockup rig and BMS testing bench through the same pipeline used for on-vehicle data.
- Make engineering-run analysis available alongside field-vehicle analysis so a regression on the bench can be compared against in-service vehicles.
- Support the engineering use-case of replaying a recorded run through analysis pipelines.

## Functional Targets

### Telematics Unit

- Capture from CAN (and any other in-vehicle buses present), GPS/GNSS, and any auxiliary sensors the target deployment needs.
- Buffering and store-and-forward behaviour for periods without connectivity, with no data loss within a defined offline window.
- Secure, authenticated transport upstream, surviving reconnects and partial outages.
- OTA-updateable firmware with rollback safety.
- Configurable signal selection so different deployments can record different parameters without firmware changes.
- A device-identity model that makes provisioning, decommissioning, and tenant assignment manageable.
- Operational envelope — temperature, vibration, ingress — appropriate to the vehicle class the unit is deployed onto.

### Backend

- Ingestion pipeline able to take device traffic at the team's targeted fleet scale without becoming the bottleneck.
- A time-series-capable store sized for both real-time queries and long-range historical aggregation.
- Tenant-scoped APIs with authentication, authorisation, rate limiting, and audit logging.
- Stream processing for derived signals, alerts, and aggregates.
- Integration points for the bench/test tracks — the mockup rig and BMS testing bench produce data that lands in the same store.
- Operational story — health metrics, tracing, logs, dashboards, runbooks, and on-call expectations.

### Frontend

- Live telemetry dashboards configurable per role, per tenant, and per use case.
- Historical analysis with the ability to compare runs, vehicles, drivers, and time windows.
- Vehicle and fleet management screens — register, decommission, configure.
- Tenant administration — users, roles, API keys, audit trail.
- Export paths for data taken out of the platform.

### Multi-Tenancy and Security

- Per-tenant data isolation enforced beyond "trust the application layer." The mechanism (separate database, separate schema, row-level keying with defence-in-depth) is the team's choice and is justified in a Design Rationale Document.
- Authentication, authorisation, and audit designed assuming the platform will eventually face untrusted users.
- Encryption in transit and at rest, with a credential- and key-management story.
- A clearly written threat model — what the system defends against, what it does not, and where the open questions are.

## Decisions the Team Owns

This track deliberately leaves architecture and technology decisions to the team. The decisions below are the ones that matter most; each should be captured in a Design Rationale Document with alternatives considered and reasoning made explicit. Locking these in early prevents the work from forking.

### Scope and Scale

- **Fleet size targets** for v1 and beyond — how many vehicles concurrently online? What message rate per vehicle? What retention period for raw telemetry versus aggregates?
- **Tenancy model** — what counts as a tenant in the target deployment? OEMs, fleet operators, departments within one customer?
- **Deployment shape** — single SaaS deployment, per-tenant deployments, on-prem-installable, or hybrid?

### Telematics Hardware

- **Hardware platform** — single-board computer (Raspberry Pi class), microcontroller (ESP32 class), an off-the-shelf automotive telematics module, or a custom design? Driven by signal-set richness, connectivity needs, environmental envelope, cost, and supply chain.
- **Connectivity** — cellular (4G/5G), Wi-Fi when in coverage, satellite for remote deployments? Multi-modal with handover?
- **Local storage** — how much can the unit buffer when offline? At what fidelity does it degrade when storage is constrained?
- **Firmware-update strategy** — OTA architecture, rollback safety, fleet-staged rollouts.

### Wire Format and Transport

- **Wire format** — Protocol Buffers, JSON, Avro, MessagePack, custom binary? Versioning strategy?
- **Transport** — MQTT, HTTPS, gRPC, AMQP, or a mix? TLS configuration and certificate management?
- **Schema evolution** — how does the platform handle a vehicle on old firmware sending an old schema while new vehicles send a new one?

### Data Layer

- **Time-series store** — TimescaleDB, InfluxDB, ClickHouse, QuestDB, or rolled on a general-purpose database?
- **Hot / warm / cold tiering** — at what retention does data move where, and how is it queried across tiers?
- **Aggregation strategy** — pre-aggregated rollups (continuous aggregates, materialised views) vs on-the-fly aggregation at query time?
- **Multi-tenancy mechanism** — separate database per tenant, separate schema per tenant, row-level tenant keying, or hybrid?

### Backend Services

- **Language and framework** — Python (FastAPI, Django, Flask), Go, Rust, Node.js, Java? Driven by team skills, ecosystem fit, and performance needs.
- **API style** — REST, GraphQL, gRPC, or a mix?
- **Authentication and authorisation** — OAuth2 / OIDC, custom token model, mTLS for service-to-service?
- **Stream processing** — Kafka with a stream processor, NATS, in-database (continuous aggregates), or batch with streaming added later?

### Frontend

- **Frontend stack** — React, Vue, Svelte, or other? TypeScript as a default or not?
- **Visualisation** — Grafana embedded, Apache Superset, or custom built on D3 / Plotly / Chart.js?
- **Mobile** — responsive web only, or a native app eventually?

### Operations

- **Hosting** — AWS, GCP, Azure, or self-hosted Kubernetes?
- **Observability** — Prometheus + Grafana, OpenTelemetry, or hosted (Datadog, New Relic)?
- **CI/CD** — GitHub Actions, GitLab CI, Jenkins, Drone?
- **Disaster recovery** — RTO and RPO targets, backup strategy, multi-region story?

### Cost Model

- **Cost per device per month** at target scale — informs every infrastructure decision.
- **License costs vs operational costs** trade-offs in the chosen stack.
- **Commercial positioning** — is the platform a SaaS product, a deployable system customers run themselves, or both?

## Starting Points

Concrete tools and patterns the team can evaluate as accelerators. None of these are commitments — they are starting references that the team can adopt, modify, or replace.

- **Wire format**: Protocol Buffers as a sensible default for binary efficiency and versioning; JSON for early prototypes where readability matters.
- **Transport**: MQTT for telemetry-style data (broker buffering, resilience to flaky connectivity); HTTPS or gRPC for command/control and admin APIs.
- **Time-series**: TimescaleDB (PostgreSQL-based, SQL-friendly), InfluxDB (purpose-built TSDB), ClickHouse (columnar analytics).
- **Backend**: Python with FastAPI for development ergonomics, or Go for runtime efficiency and easy deployment.
- **Frontend**: TypeScript with React for the application layer; Grafana embedded for quick time-series visualisation, custom dashboards for differentiated UX.
- **Telematics hardware**: Raspberry Pi for richer compute and Python ecosystem during prototyping; ESP32 for cost-sensitive and battery-powered deployments; automotive telematics modules (Quectel, Teltonika, similar) for ready-made integration into vehicles.
- **Hosting**: a single-region Kubernetes deployment is usually enough for v1; multi-region comes when SLA and DR demand it.
- **Authentication**: an OIDC provider (Auth0, Keycloak, AWS Cognito) for v1 to avoid building identity in-house.

The team should evaluate these against their own constraints and lock in their choices early enough that the work doesn't fork.

## Deliverables

- Hardware build of the telematics unit, with documented BOM, firmware, and provisioning workflow.
- Backend services with deployment manifests, runbooks, and observability dashboards.
- Frontend application with documented features and a user guide.
- API specification, message-schema repository, and a versioning policy.
- Threat model and security review document.
- A load-test report demonstrating the platform meets the team's stated scale target.
- A reference deployment used by the VCU, race car, and Vayve teams as their data plane.
- Design Rationale Documents for each of the major decisions listed above.

## Out of Scope

- Designing the VCU or BMS that produce the signals being captured.
- Designing the analytical models that consume the data — the platform makes data accessible cleanly; specific analytics are a downstream concern.
- Building tenant-specific applications on top of the API — the platform exposes the data; tenants build their own apps.

## Inter-Track Dependencies

- **VCU design track** — a primary data source. The two tracks agree on the message schema, signal catalogue, and emission cadence.
- **48V BMS firmware track** — a secondary data source. Battery telemetry lands in the same platform.
- **Vehicle mockup track** — the rig is also a data source; mockup-bench data and on-vehicle data share the same backend.
- **48V BMS testing track** — bench data from cell-level testing flows into the same monitoring stack eventually.
- **Race team and Vayve vehicle teams** — first realistic consumers; their requirements shape dashboard and analysis features.
