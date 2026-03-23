# SPHYNX — Final Design & Architecture Specification

**Workflow Orchestration Engine | SPHUTA Platform**

> Consolidated from All Project Documents & Design Sessions
> Pommala LLC | SPHUTA Platform
> Version 1.0 | March 2026
> CONFIDENTIAL

---

## 1. Executive Summary

Sphynx is the workflow orchestration engine within the SPHUTA platform, a proprietary enterprise suite built by Pommala LLC. It delivers low-code, AI-augmented, compliance-first workflow automation for regulated industries including financial services, healthcare, insurance, government, and logistics.

The platform follows a "design once, execute anywhere" philosophy, enabling workflows to run on-premises, in the cloud, or in hybrid environments without re-coding. Sphynx unifies process automation, decision intelligence, integration, and governance under a single cohesive architecture.

### 1.1 Core Architectural Decision

**Sphynx is a fully native runtime engine.** It is not a wrapper around Temporal, Camunda, or any external workflow engine. This decision is final and locked. Sphynx borrows proven design patterns and concepts from Temporal (durable execution, event sourcing, deterministic replay) and Camunda (BPMN modeling, human task inbox, DMN decision tables) but implements them entirely in its own codebase. The only external engine dependency is **Apache Camel**, used exclusively as the connector and integration layer.

### 1.2 Key Value Proposition

- **For Stakeholders:** Future-proof, multi-industry platform unifying process automation, integration, and decision intelligence in one stack.
- **For Clients:** Faster time-to-market, reduced integration costs, and built-in compliance tooling across SOX, HIPAA, GDPR, and PCI DSS.
- **For Engineering Teams:** Specialized roles in software architecture, workflow automation, AI/ML integration, BPMN/DMN design, and cloud-native deployment.

---

## 2. Architectural Philosophy & Design Principles

### 2.1 Guiding Principles

- **Native First:** Sphynx owns its entire execution runtime. No vendor lock-in, no licensing risk, full control over every feature.
- **Separation of Concerns:** Latency-critical tasks execute in-process. Orchestration, audit, and integration layers handle non-latency-critical business logic separately.
- **Declarative Workflow Definition:** All workflows, service tasks, mappings, and schemas are defined in the Sphynx JSON DSL. No static Java code needed for new integrations or field mappings.
- **Runtime Dynamic Generation:** Camel routes are generated at runtime from DSL definitions. Custom object schemas are registered and validated dynamically.
- **Event Sourcing as Foundation:** Append-only history per workflow instance; all state is reconstructable by replay. This is a design requirement inherited from Temporal's proven model.
- **Compliance First:** Immutable audit logs with cryptographic hash-chain integrity, WORM-compliant storage, and pre-built regulatory templates.
- **Vertical Scaling First:** Vertical scaling takes precedence by default; automatic horizontal scale-out triggers on threshold breach.

### 2.2 What Sphynx Takes from Temporal (Concepts Only)

Sphynx implements Temporal's proven patterns natively in Spring Boot + PostgreSQL + Redis + Kafka:

- Token-based dispatcher with event-sourced state
- Deterministic replay for crash-safe, resumable workflows
- Per-node retries with configurable backoff (exponential, linear, fixed)
- Four-timeout taxonomy: Schedule-To-Start, Start-To-Close, Schedule-To-Close, Heartbeat
- Saga/compensation chains for distributed rollback
- Signal park-and-wake pattern for external event correlation
- Durable timers with deadline scheduling
- Continue-as-new / snapshot rollover (warn at 5K events, force at 10K)
- Version markers / patching for in-flight instance migration
- Queue-based scaling triggers (backlog size, schedule-to-start latency)

### 2.3 What Sphynx Takes from Camunda (Concepts Only)

- Parallel gateway with waitAll/waitAny/NofM join semantics
- Human task service with role-based inbox and delegation
- SLA escalation engine with configurable timeout policies
- Process monitoring console (the "Operate" pattern)
- DMN-compatible decision table evaluator
- BPMN 2.0 import/export for standards interoperability

### 2.4 What Sphynx Adds Uniquely

- **AIDecisionTask:** Native LLM/ML model invocation as a first-class node type, not a code hook. Supports model versioning, A/B testing, explainability (LIME/SHAP), and inline Python/JS/Groovy scoring.
- **Compliance-First Audit:** Cryptographically verifiable hash-chain audit log, not just event logging. WORM-compliant, with digital signatures and non-repudiation.
- **Unified Multi-Format DSL:** JSON, YAML, and BPMN all converge to a single internal model. Workflow authors never think about which runtime executes their definition.
- **Dynamic Camel Route Generation:** ServiceTask and ConnectorTask DSL nodes generate Apache Camel routes at runtime with zero boilerplate.

---

## 3. Platform Topology & Layered Architecture

### 3.1 Sphynx Engine vs. Sphynx Platform

| Layer | Scope | Responsibilities |
|-------|-------|-----------------|
| Sphynx Engine | Runtime Core | DSL parsing, validation, execution, task routing, policy enforcement, mapping, Camel route generation, audit publishing |
| Sphynx Platform | Business Layer | UI/portal, workflow modeler, business rule editors, AI decision consoles, human task inbox, audit dashboards, governance controls |

### 3.2 Nine-Layer Logical Architecture

| Layer | Components | Technology |
|-------|-----------|-----------|
| 1. API Gateway | REST/gRPC ingress, rate limiting, auth, tenant routing | Spring Boot, Spring Security 7, OAuth2 |
| 2. Orchestrator | DSL parser, runtime plan builder, workflow executor, SLA enforcer | Sphynx Engine Core |
| 3. Execution Runtime | Worker pods, step executors, executor registry | Spring Boot Workers, Virtual Threads |
| 4. State Store | Instance state, execution tokens, node results | PostgreSQL (primary), Redis (cache/locks) |
| 5. Timer & Scheduler | Deadline scheduling, SLA timers, retry scheduling | Quartz, custom timer_wheel table |
| 6. Message Bus | Signals, events, audit stream, DLQ/replay | Kafka (primary), RabbitMQ (optional) |
| 7. Policy & Rules | OPA guards, decision tables, AI model adapters | OPA WASM, Drools, SpEL, REST ML endpoints |
| 8. Connector Service | Integration routes, 350+ connectors, EIP patterns | Apache Camel |
| 9. Observability | Metrics, tracing, audit logs, visibility index | OpenTelemetry, Prometheus, Grafana, Jaeger |

### 3.3 Service Decomposition

The engine's internal package boundaries are designed for eventual service split. In v1, all run together in a single process. The boundaries are enforced by Spring Modulith 2.0:

- **sphynx-runtime/gateway —** Admission, rate limiting, auth, routing
- **sphynx-runtime/engine —** Mutable instance state, event history, timers, task queuing
- **sphynx-runtime/dispatch —** Task queue dispatch, worker polling, throttling
- **sphynx-runtime/system —** Internal system workflows, background jobs
- **sphynx-runtime/signals —** Signal correlation, event subscriptions, webhook waking
- **sphynx-runtime/visibility —** Projection-based query index for workflow search and monitoring

---

## 4. Repository Structure & Module Layout

Sphynx is organized across four primary repositories, each with a specific mission:

### 4.1 sphynx-engine (or sphynx-runtime)

The native execution runtime. This is the heart of the platform — the dispatcher, state machine, executor registry, token model, and all core runtime logic.

| Internal Module | Purpose |
|----------------|---------|
| schema/postgres/ | Versioned Flyway migration scripts for all persistence tables |
| gateway/ | REST/gRPC API admission, rate limiting, tenant routing |
| engine/ | Mutable state, event history, timers, task queuing |
| dispatch/ | Task queue dispatch, worker polling, throttling |
| signals/ | Signal correlation, external event waking |
| visibility/ | Query projection index for monitoring |
| system/ | Internal system workflows |
| executors/ | Node executor implementations (UserTask, ServiceTask, RuleTask, AIDecisionTask, etc.) |
| connectors/ | Apache Camel integration layer |

### 4.2 sphynx-api

**Purpose:** Canonical contract definitions. Protobuf schemas, OpenAPI specs, shared types. No Spring Boot runtime application.

- Protobuf + OpenAPI + buf tooling
- Contract test sub-module validates specs against running engine
- Spring REST Docs generation from live tests

### 4.3 sphynx-features

**Purpose:** History compatibility testing harness. Recorded execution histories per feature, replayed against older and newer engine versions in CI.

- Full Spring Boot 4 test application
- RestTestClient + Testcontainers 2.0 + @ServiceConnection
- Three-check CI replay pipeline: (1) current engine replays old histories, (2) old engine replays current histories, (3) new features produce valid histories
- This is the primary quality gate — replay divergence is a data integrity incident

### 4.4 sphynx-worker-controller

**Purpose:** Kubernetes operator for safe worker versioning and rollout.

- Java Operator SDK (not Spring Cloud Kubernetes) for reconciliation loops and CRD handling
- Spring Boot 4 for wiring, metrics, HTTP clients
- Two CRDs: SphynxConnection and SphynxWorkerDeployment
- Three rollout strategies: Manual, AllAtOnce, Progressive
- Six worker version lifecycle states for safe blue/green deployments

---

## 5. Technology Stack (Locked Decisions)

### 5.1 Core Runtime Stack

| Component | Technology | Version / Policy |
|-----------|-----------|-----------------|
| Language | Java | 21 LTS |
| Framework | Spring Boot | 4.0.4 (native, no Boot 3 compat) |
| Core Framework | Spring Framework | 7.x |
| Module Boundaries | Spring Modulith | 2.0 |
| Primary Database | PostgreSQL | 16+ |
| Cache / Fast Locks | Redis | 7+ |
| Message Bus | Apache Kafka | 3.x |
| Integration Layer | Apache Camel | 4.x |
| Timer / Scheduler | Quartz | 2.3+ |
| DB Migrations | Flyway | 11.x (from Boot BOM) |
| Serialization | Jackson | 3 (Boot 4 default, no legacy) |
| Null Safety | JSpecify | 1.0.0 (project policy) |
| Security | Spring Security | 7 + OAuth2 Resource Server |
| HTTP Clients | @HttpExchange | Typed declarative clients |
| Observability | OpenTelemetry | spring-boot-starter-opentelemetry |
| Metrics | Micrometer + Prometheus | Via Boot Actuator |
| Tracing | Jaeger / Zipkin | Via OTel exporter |
| Build System | Gradle | 8.x |
| Containerization | Docker + Kubernetes + Helm | Cloud-native |

### 5.2 Key Technology Decisions

- **Virtual Threads:** Enabled by default (spring.threads.virtual.enabled: true) for I/O-heavy paths. Validated under real load, not assumed universal.
- **Spring Cloud:** Selective use only. Pinned to 2025.1.1 (Oakwood) for Boot 4.0.1+ compatibility. Gateway 5.0.1 for external API gateway. Config Server not needed.
- **Spring AI:** On hold until 2.0 GA. AIDecisionTask uses @HttpExchange clients behind the StepExecutor SPI. No direct Spring AI dependency in core engine.
- **gRPC:** Spring gRPC 1.0 (GA Dec 2025) reserved as optional future optimization for engine-worker communication. Not a current dependency.
- **Undertow:** Removed in Boot 4. Using Tomcat 11 or Jetty 12.
- **Classic Starters:** Excluded. Boot 4 modular starters only (e.g., spring-boot-starter-webmvc, not spring-boot-starter-web).

---

## 6. Sphynx JSON DSL Specification

### 6.1 Overview

The Sphynx JSON DSL is the primary workflow definition format. It is extensible, enterprise-grade, and designed for both low-code UI rendering and programmatic authoring. Workflows defined in YAML or BPMN 2.0 are auto-converted to the internal JSON model at runtime.

### 6.2 Supported Node Types

| Node Type | Purpose | Key Attributes |
|-----------|---------|---------------|
| StartEvent | Explicit workflow entry point | Optional; implicit if omitted |
| UserTask | Human-in-the-loop task | role, form, sla, retries, outcomes, assignment |
| ServiceTask | External service invocation | service, inputMapping, outputMapping, customObject, outputType |
| RuleTask | Decision logic execution | rule, ruleType (decisionTable/externalService), onTrue/onFalse, onDecision |
| AIDecisionTask | AI/ML model invocation | model, threshold, inputMapping, outputMapping, inlineModel |
| ParallelGateway | Concurrent branch execution | branches, join (waitAll/waitAny/count(n)) |
| ExclusiveGateway | Conditional routing | conditions, default branch |
| TimerTask | Delays, reminders, triggers | duration, cron, fireAt |
| ScriptTask | Embedded script execution | language (Groovy/JS/Python), code |
| NotificationTask | Notification dispatch | channel (email/SMS/Slack), template |
| SignalTask | External event wait | signalName, correlation, timeout |
| ConnectorTask | System integration | connector (Kafka/SAP/Salesforce), config |
| CompensationTask | Saga rollback logic | compensateFor, rollbackAction |
| AuditTask | Structured audit event | auditType, metadata |
| SubProcess | Nested workflow invocation | workflowRef, inputMapping |
| EndEvent | Workflow completion | status (success/rejected/escalated) |

### 6.3 Advanced DSL Features

**Outcome Mapping:** Each node supports multiple named outcomes directing workflow routing: `outcomes: { "approved": "nextNode", "rejected": "endRejected", "timeout": "escalation" }`

**Input/Output Mapping:** Dynamic data flow between workflow variables and node execution using `${input.field}` and `${result.field}` syntax. Supports JSONPath and JMESPath expressions for nested object and array access.

**Retry Logic & SLA:** Per-task business SLAs (ISO 8601 duration, e.g., "P2D") and configurable retry attempts with backoff policies for resilience and compliance.

**Workflow Variables:** Global workflow variables with type safety: `{ "timesheetId": "string", "totalHours": "number", "approved": "boolean" }`

**Custom Object Schemas:** Inline JSON Schema for runtime validation and optional dynamic POJO instantiation. Each ServiceTask can declare a customObject with field definitions.

**Metadata:** Per-node metadata for audit, compliance, UI rendering, analytics, and operational insights: audit, compliance, ui, analytics, escalationPolicy fields.

**Runtime Configuration:** DSL-level concurrency controls (maxParallelNodes, workflowConcurrency), queue assignment, scaling strategy (vertical-first / horizontal-first), and rate limiting.

---

## 7. Execution Model & Runtime Internals

### 7.1 Core Runtime Components

| Component | Responsibility |
|-----------|---------------|
| DefinitionService | Manages versioned DSL definitions; supports hot-reload without restart |
| InstanceService | Creates and manages workflow instances and variables |
| Dispatcher | Computes next runnable steps; token-based execution flow |
| ExecutorRegistry | Maps node type to StepExecutor implementation |
| StepExecutor (SPI) | Abstract interface for all node executors |
| Retry/Backoff Engine | Per-node retry logic with exponential/linear/fixed backoff |
| Timer Engine | Schedules and wakes due tasks via Quartz + timer_wheel table |
| Signal Bus | Correlates external events to parked workflow tokens |
| Saga/Compensation Manager | Handles rollback chains on failure/cancellation |
| Human Task Service | Manages inbox, SLAs, escalations, delegation |
| AuditPublisher | Writes immutable audit events to DB + Kafka stream |

### 7.2 Execution Flow

1. DSL uploaded via REST API or Platform UI, validated against JSON Schema, stored in workflow_definition table
2. Workflow instance started with initial variables; execution_token created for the start node
3. Dispatcher identifies runnable nodes (tokens in READY state), enqueues steps to work queues
4. Worker pod claims a step via FOR UPDATE SKIP LOCKED, executes via the registered StepExecutor
5. Executor writes node_result, applies outcome-based transitions, creates new tokens for next nodes
6. Tokens park for waits: UserTask (human input), TimerTask (deadline), SignalTask (external event)
7. Parallel branches: ParallelGateway spawns branch tokens; join conditions (waitAll/waitAny/count(n)) gate merge
8. Audit events published to Kafka and appended to audit_log with hash-chain integrity
9. On failure: retry engine schedules per retry policy; on exhaustion: saga manager runs compensation
10. Workflow reaches EndEvent; instance status set to COMPLETED/REJECTED/ESCALATED

### 7.3 Deterministic Execution Contract

The DSL interpreter is deterministic with no side effects. All executors are idempotent. This enables safe replay for crash recovery and debugging. Key rules:

- No randomness, no current-time calls, no non-deterministic I/O in the interpreter path
- All side effects (API calls, DB writes, notifications) happen inside StepExecutors, not the dispatcher
- Idempotency keys on all external calls to prevent duplicate effects on retry

### 7.4 Timeout Taxonomy

| Timeout Type | Scope | Purpose |
|-------------|-------|---------|
| Schedule-To-Start | Task queue | Time between task enqueue and worker claim |
| Start-To-Close | Executor | Max execution time for a single attempt |
| Schedule-To-Close | End-to-end | Total time from enqueue to completion (all retries) |
| Heartbeat | Long-running | Periodic liveness signal from executor to engine |

---

## 8. Persistence Model & State Management

### 8.1 Database Schema

| Table | Purpose | Key Indexes |
|-------|---------|-------------|
| workflow_definition | Versioned DSL storage | id, version, tenant_id |
| workflow_instance | Running instance state + variables | inst_id, status, tenant_id |
| execution_token | Per-node execution tracking | inst_id, node_id, status |
| node_result | Completed step outcomes and outputs | inst_id, node_id, seq |
| timer_wheel | Scheduled timers and deadlines | fire_at, status |
| signal_subscription | External event subscriptions | correlation_key, signal_name |
| human_task | User task inbox entries | assignee, status, tenant_id |
| audit_log | Immutable audit events with hash chain | inst_id, seq, tenant_id |
| migration_marker | Instance version migration tracking | inst_id, from_version, to_version |

### 8.2 Storage Strategy

- **PostgreSQL (Primary):** All coordination state, instance data, definitions, human tasks, audit log. ACID transactions ensure atomicity between state writes and audit events. Optimistic locking via version column for concurrent updates.
- **Redis (Cache + Fast Locks):** Hot cache for active instance state, fast distributed locks for dispatcher contention, queue claim acceleration.
- **Kafka (Streams + Events):** Audit event streaming to S3/data warehouse, signal delivery, DLQ for failed executions, cross-service event bus.
- **S3/Cassandra (Cold Storage):** Tiered storage for execution history. Hot data in Postgres/Redis, cold history in S3 or Cassandra for long-term retention.

---

## 9. Integration Layer (Apache Camel)

### 9.1 Camel's Role — Hard Boundary

Apache Camel is the one legitimate external engine dependency. It belongs in the connector and mediation plane only. Camel never touches workflow state, execution logic, or the token model. Its responsibilities are:

- 350+ pre-built connectors (REST, SOAP, Kafka, JMS, S3, Salesforce, SAP, Slack, databases)
- Enterprise Integration Patterns (content-based routing, enrichers, splitters, aggregators, DLQ)
- Dynamic route generation from DSL ServiceTask and ConnectorTask node definitions
- Request building (inputMapping), API invocation, response parsing, output mapping
- Custom object instantiation when outputType is specified

### 9.2 Runtime Route Generation

At workflow deployment or DSL change, the engine parses ServiceTask nodes and programmatically builds Camel routes:

- Each route is parameterized by node ID, input/output mappings, and custom object schema
- Routes register dynamically — no restart required for new integrations (hot reload)
- A generic DslOutputMappingProcessor handles field extraction, renaming, type conversion
- Supports Map, List, and POJO output types with JSONPath/JMESPath extraction

### 9.3 Ultra-Low-Latency Bypass

For microsecond-SLA critical paths (e.g., real-time fraud scoring), tasks execute as in-process method calls via Spring Boot and Drools, bypassing Camel and the orchestration layer entirely. Results are optionally reported to the orchestrator after-the-fact for audit compliance.

---

## 10. AI/ML & Rules Integration

### 10.1 AIDecisionTask Patterns

Sphynx treats AI/ML as a first-class workflow node type, not an afterthought:

- **External Model Invocation:** REST calls to TensorFlow Serving, PyTorch Serve, or any ML endpoint with model versioning and A/B testing.
- **Inline Model Code:** Embedded Python/JS/Groovy scoring functions defined directly in the DSL for lightweight rule-based ML.
- **Threshold-Based Routing:** Model confidence scores compared against configurable thresholds to route pass/fail outcomes.
- **Explainability:** LIME/SHAP integration metadata for audit-ready model explanations.
- **NLP Automation:** Unstructured task inputs (emails, chat, scanned docs) processed via NLP models for classification and extraction.

### 10.2 RuleTask Patterns

- **Decision Tables:** Multi-condition rule evaluation with ordered conditions and fallback defaults.
- **SpEL / Drools / OPA:** Pluggable rule engines for different complexity levels.
- **External Rule Services:** REST-based decision engines (e.g., RegTech platforms) with full audit of request/response schemas.
- **Conditional Routing:** onTrue/onFalse for binary rules; onDecision map for multi-outcome routing.

### 10.3 Future AI Roadmap

- Predictive orchestration (next-best-action, auto-pathing)
- Anomaly detection (SLA breach prediction, drift alerts)
- Adaptive process optimization (policy-safe dynamic tuning)
- RAG-based next-best-action suggestions
- Self-healing workflows via AI failure prediction

---

## 11. Human Task Management & Governance

### 11.1 Human Task Service

- Role-based task inbox with configurable assignment strategies (static, dynamic expression-based)
- SLA enforcement with configurable escalation policies (notifyL2After2H, auto-reassign)
- Task delegation, substitution, and reminders
- Form rendering support for low-code UI
- Webhook/REST callbacks for external task completion
- Multi-instance tasks: collection-based iteration with elementVar binding

### 11.2 Governance & Lifecycle Management

- Workflow versioning with Dev → QA → Prod promotion pipelines
- Approval workflows for publishing definition changes
- Full change history and rollback capabilities
- Policy enforcement via OPA WASM at API and node level
- JWT + OPA for fine-grained RBAC/ABAC security

### 11.3 Compliance Framework

| Capability | Implementation |
|-----------|---------------|
| Encryption at Rest | AES-256 for all persisted data |
| Encryption in Transit | TLS 1.3 for all communication |
| Audit Logging | Append-only with hash-chain integrity, WORM-compliant |
| Access Control | RBAC/ABAC tied to SSO, LDAP, OAuth2 |
| Regulatory Templates | Pre-built for SOX, HIPAA, GDPR, PCI DSS |
| Digital Signatures | Workflow non-repudiation support |
| Secrets Management | Vault, AWS Secrets Manager, Azure Key Vault integration |

---

## 12. Multi-Tenancy, Scaling & Deployment

### 12.1 Multi-Tenancy

- Logical tenant isolation at data, execution, and connector levels
- Per-tenant quotas, scaling policies, and SLA definitions
- Tenant-scoped admin dashboards and analytics views
- Safe connector sandboxing to prevent cross-tenant data leakage

### 12.2 Scaling Architecture

- **Vertical-First:** Sphynx scales vertically by default. Virtual threads enable high concurrency on single nodes.
- **Horizontal Scale-Out:** Automatic threshold-triggered horizontal scaling via Kubernetes autoscaling.
- **Worker Sharding:** Workflows distributed by ID/type for load balancing across worker pods.
- **Queue-Based Triggers:** Scale decisions driven by backlog size and schedule-to-start latency metrics.

### 12.3 Performance Benchmarks

| Metric | Target |
|--------|--------|
| Task Execution Throughput | 100K+ tasks/hour |
| DSL Parsing Latency (95th) | < 50ms |
| State Recovery Time (cold) | < 100ms |
| Human Task Cycle Time (90th) | < 1 hour |
| Cluster Rebalance (10K instances) | < 500ms |
| Tenant Isolation | Zero cross-tenant leaks |

### 12.4 Deployment Model

- **Cloud-Native:** Kubernetes + Helm for all orchestration. Docker images for all core services.
- **Hybrid/On-Premise:** Full on-prem support with Infrastructure-as-Code templates (Terraform, Ansible).
- **Disaster Recovery:** Active-active or active-passive clustering. Geographic redundancy with cross-region replication. Point-in-time restore of workflow states.
- **Rollout Strategies:** Blue/green and canary deployments. Worker controller manages safe version transitions.

---

## 13. Observability & Operations

### 13.1 Monitoring Stack

- **Metrics:** Micrometer + Prometheus for workflow execution stats, queue sizes, task durations, error rates.
- **Tracing:** OpenTelemetry spans per workflow/node with distributed tracing via Jaeger/Zipkin.
- **Logging:** Structured JSON logs with traceId and spanId. ELK stack (Elasticsearch, Logstash, Kibana) for search and analysis.
- **Dashboards:** Grafana dashboards for workflow health, parallel execution bottlenecks, and SLA tracking.
- **Alerting:** SLA breach alerts via email/SMS/Slack. PagerDuty integration for critical failures.

### 13.2 Audit & Replay

- Append-only audit_log with cryptographic hash chain for tamper detection
- Kafka audit stream to S3/data warehouse for long-term retention
- Full workflow replay capability for debugging and compliance review
- Health checks, auto-healing, and readiness/liveness probes for all services

---

## 14. Tooling Ecosystem

| Tool | Purpose | Technology |
|------|---------|-----------|
| Modeler (Web/Desktop) | BPMN, DMN, and DSL visual workflow design | React/Next.js |
| Console | Admin, deployment, and tenant management UI | React/Next.js |
| Operate | Runtime workflow monitoring and instance management | React + Grafana |
| Tasklist | Human task inbox and form management | React/Next.js |
| Optimize | SLA analytics and AI-driven recommendations | React + ML endpoints |
| Marketplace | Prebuilt connectors and workflow templates | React + Registry API |
| CLI | Workflow creation, validation, deployment | Java CLI |
| SDKs | Java, Go, TypeScript development kits | Multi-language |

---

## 15. Competitive Positioning

| Capability | Sphynx | Camunda | Temporal |
|-----------|--------|---------|----------|
| Architecture | Native runtime, Spring-first | BPMN-focused, Java-centric | Code-first, Go server |
| Workflow Definition | JSON DSL + YAML + BPMN | BPMN XML | Code only (Java/Go/TS) |
| Low-Code Support | Full drag-and-drop + DSL | BPMN modeling only | None |
| AI/ML Integration | First-class AIDecisionTask | Via external code | Via activities only |
| Rule Engine | Native (SpEL/Drools/OPA/DMN) | DMN only | None built-in |
| Connector Ecosystem | 350+ via Apache Camel | 80+ connectors | None built-in |
| Human Task Inbox | Native with SLA/escalation | Native (Tasklist) | None built-in |
| Audit / Compliance | Hash-chain, WORM-compliant | Basic audit logging | Event history only |
| Multi-Tenancy | Native tenant isolation | Available in SaaS | Namespace-based |
| Licensing | Proprietary (full control) | Source-available / SaaS | MIT + Enterprise |

---

## 16. Implementation Roadmap

| Phase | Focus | Duration | Key Deliverables |
|-------|-------|----------|-----------------|
| 0 | Foundation | Weeks 1–2 | Multi-module project, CI/CD, compliance scaffolding, Micrometer integration |
| 1 | Core Engine | Weeks 3–6 | JSON DSL parser/validator, dispatcher, executor registry, state persistence, REST API |
| 2 | Event-Driven Orchestration | Weeks 7–12 | Kafka integration, timers/schedulers, parallel gateway, audit logging, metrics |
| 3 | Human-in-the-Loop & UI | Weeks 13–16 | Human task support, admin console, RBAC/ABAC, basic Operate dashboard |
| 4 | Advanced Features | Weeks 17–24 | Saga/compensation, workflow versioning/migration, sharding, rule engine adapters, alerting |
| 5 | Ecosystem & Marketplace | Weeks 25+ | Visual modeler, template marketplace, multi-tenancy SaaS, SDKs, documentation |

### 16.1 Build Sequence (Locked)

The confirmed build sequence prioritizes the execution runtime and testing infrastructure:

1. sphynx-engine-core — The execution runtime (dispatcher, token model, state machine)
2. sphynx-testing — Embedded test server + history fixture harness (built alongside the engine, not after)
3. sphynx-spring-boot-starter — Spring wiring, worker autoconfigure
4. sphynx replay CLI + history fixture test suite — Compatibility gate before any release
5. sphynx-ui + operator console — After runtime contracts are stable

### 16.2 Risk Mitigation

| Risk | Mitigation |
|------|-----------|
| DSL Flexibility | Versioned schema, backward compatibility, comprehensive tests |
| Persistence Bottlenecks | Redis sharding, JPA batching, tiered storage |
| Human Task SPOF | Stateless task service, Kafka consumer groups |
| Replay Divergence | Three-check CI pipeline as primary quality gate |
| Spring Cloud Version Trap | Pinned to 2025.1.1, explicit compatibility testing |

---

## 17. Target Industries & Use Cases

| Industry | Primary Use Cases |
|----------|------------------|
| Financial Services | Loan approvals, fraud detection, compliance workflows, credit risk scoring |
| Healthcare | Patient intake, claims processing, HIPAA-compliant automation, clinical workflows |
| Insurance | Underwriting, claims adjudication, regulatory reporting, policy lifecycle |
| Government | Permitting, benefits processing, audit-ready workflows, case management |
| Logistics | Shipment orchestration, tracking, exception handling, supply chain automation |
| Technology / SaaS | Workflow orchestration, developer self-service portals, billing automation |

---

## 18. Conclusion

Sphynx, as the workflow orchestration engine of the SPHUTA platform by Pommala LLC, represents a comprehensive, ground-up approach to enterprise workflow orchestration. By building a native runtime engine that incorporates the best proven patterns from Temporal and Camunda without depending on either, Sphynx delivers a differentiated engine with full control, zero vendor lock-in, and a genuinely unique capability set.

The combination of a powerful JSON DSL, first-class AI/ML integration, Apache Camel's connector ecosystem, compliance-first audit architecture, and Spring Boot 4's modern runtime provides a foundation capable of serving the most demanding enterprise environments across regulated industries.

**The hard design work is done.** The schema is defined, the dispatcher model is designed, the audit architecture is specified, the executor registry is mapped, and the token model is complete. Building from here is engineering, not research.

---

> *Sphynx — Part of the SPHUTA Platform | Pommala LLC*
> *Intelligent, Compliant, and Scalable Enterprise Automation*
