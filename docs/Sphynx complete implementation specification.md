# SPHYNX — Complete Implementation Specification

**Workflow Orchestration Engine | SPHUTA Platform**

> **Authoritative Architecture Standard & Implementation Reference**
> Version 2.0 | March 2026
> Pommala LLC | SPHUTA Platform
> CONFIDENTIAL | NORMATIVE

*Conformance Language: The key words MUST, MUST NOT, SHALL, SHALL NOT, SHOULD, SHOULD NOT, RECOMMENDED, MAY, and OPTIONAL in this document are to be interpreted as normative requirements.*

---

## PART I — ARCHITECTURE STANDARD

---

## 1. Canonical Views

Sphynx SHALL be described through two complementary and equally valid views:

- **Capability Catalog (130 features):** Used for platform breadth, competitive positioning, external communication, and portfolio-level representation.
- **Tiered Architecture View (38 / 55 / 37):** Used for implementation planning, architecture governance, delivery sequencing, dependency control, and engineering review.

These two views SHALL coexist. Neither view SHALL invalidate the other.

### 1.1 Core Architectural Decision

**Sphynx is a fully native workflow orchestration engine.** It is NOT a wrapper around any external workflow runtime. This decision is final and locked. Apache Camel is the only external engine dependency, used exclusively as the connector and integration layer.

Sphynx incorporates proven industry design patterns — durable execution, event sourcing, deterministic replay, saga compensation, worker versioning, BPMN modeling, human task inbox, DMN decision tables, process instance migration, and compensation events — all implemented entirely in its own codebase.

**Sphynx supports three workflow authoring modes — JSON DSL, BPMN 2.0, and a visual UI designer —** all of which are translated into a common internal workflow definition and executed identically by the runtime. These are co-equal as authoring entry points, not as execution formats. The engine maintains one canonical internal representation regardless of how a workflow was authored.

### 1.2 Naming Standard

- References to the engine SHALL use "SPHYNX Engine" in product documentation.
- References SHALL use "SPHYNX Orchestration Engine" in architecture documentation.
- The term "SPHYNX Platform" SHALL NOT be used. The platform is SPHUTA.
- SPHYNX SHALL NOT be expanded as an acronym.
- SPHUTA always appears as parent/master brand; SPHYNX always appears as sub-brand/engine.

---

## 2. Tier Model

### 2.1 Tier 1 — Core Engine (38 features)

**Engine-correctness features.** If failure, omission, or incompatibility in a Tier 1 feature can compromise workflow progression, dispatch correctness, token integrity, replay safety, state consistency, execution determinism, or audit integrity — it is Tier 1.

If a Tier 1 feature is missing or materially defective, the engine SHALL be considered architecturally incomplete.

### 2.2 Tier 2 — Enterprise Runtime (55 features)

**Enterprise runtime features.** Materially improve production governance, runtime visibility, security, compliance, multi-tenancy, control-plane maturity, operational scalability, enterprise sellability, or advanced orchestration — without redefining the core execution kernel.

The engine MAY execute without some Tier 2 features. Enterprise-grade operation SHOULD assume Tier 2 coverage.

### 2.3 Tier 3 — Ecosystem and Tooling (37 features)

**Ecosystem and tooling features** built around the engine rather than part of the execution kernel. Includes operator products, developer tools, SDKs, packaging, deployment assets, testing harnesses, and documentation tooling.

Tier 3 features SHALL NOT be treated as execution-kernel requirements.

### 2.4 Authoritative Counts

- 130 SHALL be the authoritative total for the master capability catalog.
- 38 / 55 / 37 SHALL be the authoritative engineering-tier distribution.
- 38 + 55 + 37 = 130 SHALL be maintained as a governance invariant.
- Any change to these totals SHALL require explicit architecture revision approval.

---

## 3. Tier 1 — Core Engine (38 Features)

### 3.1 Runtime Contracts and Engine Services (1–24)

| # | Feature | Classification |
|---|---------|---------------|
| 1 | Dispatcher | Engine-correctness |
| 2 | Token state machine | Engine-correctness |
| 3 | ExecutorRegistry | Engine-correctness |
| 4 | Deterministic execution contract | Engine-correctness |
| 5 | SideEffect / controlled non-determinism | Engine-correctness, KERNEL-CRITICAL |
| 6 | Four-timeout taxonomy | Engine-correctness |
| 7 | Retry / backoff engine | Engine-correctness |
| 8 | Timer engine (Quartz + timer_wheel) | Engine-correctness |
| 9 | Signal bus + correlation | Engine-correctness |
| 10 | Saga / compensation manager | Engine-correctness |
| 11 | Audit publisher | Engine-correctness |
| 12 | Hash-chain integrity | Engine-correctness |
| 13 | Optimistic locking | Engine-correctness |
| 14 | Hot-reload definitions | Engine-correctness |
| 15 | Workflow variables (typed) | Engine-correctness |
| 16 | Input / output mapping | Engine-correctness |
| 17 | Outcome-based routing | Engine-correctness |
| 18 | Expression language | Engine-correctness |
| 19 | Idempotency keys | Engine-correctness |
| 20 | Flyway migrations | Engine-correctness |
| 21 | PostgreSQL primary store | Engine-correctness |
| 22 | Versioning policy (pinned / auto-upgrade) | Engine-correctness |
| 23 | State rebuild replay | Engine-correctness |
| 24 | Snapshot-accelerated replay | Engine-correctness |

### 3.2 Foundational Node Executors (25–38)

| # | Node Type | Executor |
|---|-----------|----------|
| 25 | StartEvent | StartEventExecutor |
| 26 | UserTask | UserTaskExecutor |
| 27 | ServiceTask | HttpExecutor / CamelConnector |
| 28 | RuleTask | RuleExecutor |
| 29 | AIDecisionTask | AIExecutor |
| 30 | ParallelGateway | ParallelExecutor |
| 31 | ExclusiveGateway | ExclusiveExecutor |
| 32 | TimerTask | TimerExecutor |
| 33 | ScriptTask | ScriptExecutor |
| 34 | SignalTask | SignalWaitExecutor |
| 35 | CompensationTask | CompensationExecutor |
| 36 | SubProcess | SubprocessExecutor |
| 37 | NotificationTask | NotifyExecutor |
| 38 | EndEvent | EndEventExecutor |

A conforming implementation SHALL NOT claim Tier 1 completeness if any foundational executor is absent or materially non-functional.

---

## 4. Tier 2 — Enterprise Runtime (55 Features)

### 4.1 Advanced Runtime and Orchestration (1–23)

| # | Feature | Delivery Priority |
|---|---------|------------------|
| 1 | Workflow Update API | Standard |
| 2 | Task queue priority and fairness | Standard |
| 3 | Payload codec SPI | Standard |
| 4 | Schedule system | Elevated (kernel-sensitive) |
| 5 | Cancellation scopes | Elevated (kernel-sensitive) |
| 6 | Search attributes / visibility query | Prerequisite (kernel-critical) |
| 7 | Cross-tenant invocation | Standard |
| 8 | Per-node reset | Elevated (kernel-sensitive) |
| 9 | Batch operations | Standard |
| 10 | Process instance migration | Elevated (kernel-sensitive) |
| 11 | Child workflow multi-instance | Standard |
| 12 | Global user task listeners | Standard |
| 13 | Form schema | Standard |
| 14 | BPMN compensation events | Standard |
| 15 | Escalation events | Standard |
| 16 | Execution listeners | Standard |
| 17 | Cluster / tenant variables | Standard |
| 18 | Document store | Standard |
| 19 | Ad-hoc subprocess | Standard |
| 20 | Process analytics | Standard |
| 21 | Simulation / play mode | Standard |
| 22 | Human task delegation | Standard |
| 23 | SLA escalation policies | Standard |

### 4.2 Additional Node Types (24–32)

ConnectorTask (Camel), ManualTask, ExternalTask, AuditTask, CrossTenantTask, AdHocSubprocess, EscalationEvent, DSL YAML conversion, BPMN 2.0 authoring/import/export and execution mapping.

### 4.3–4.7 Runtime, Security, Tenancy, Observability, Storage

**Runtime:** per-node-type thread pools, backpressure, rate limiting.

**Security:** JWT/OAuth2, RBAC/ABAC, OPA WASM, encryption, secrets, digital signatures, compliance templates (SOX/HIPAA/GDPR/PCI), WORM storage.

**Multi-tenancy:** tenant isolation, per-tenant quotas, connector sandboxing.

**Observability:** Prometheus, OpenTelemetry, structured logging, Grafana, ELK, SLA alerting, health checks.

**Storage extensions:** Redis cache, S3 cold storage.

> *Note: Kafka streaming capabilities are governed by the Message Bus Boundary Standard (Section 7) and delivered through the message bus SPI. Kafka is NOT counted as a standalone Tier 2 feature.*

---

## 5. Tier 3 — Ecosystem and Tooling (37 Features)

### 5.1 Products

Modeler (Web/Desktop) — visual UI workflow authoring surface and drag-and-drop designer that emits canonical JSON DSL. Console, Operate, Tasklist, Optimize, Marketplace.

### 5.2 Developer Tooling

CLI, Java SDK, Go SDK, TypeScript SDK, IDE debugger plugin, Terraform provider, sphynx-bench, process testing framework, DSL Copilot.

### 5.3 Deployment

Kubernetes + Helm, Docker, on-premise, hybrid/multi-cloud, Terraform/Ansible IaC, active-active clustering, blue/green rollouts, canary rollouts, geographic redundancy, point-in-time restore.

### 5.4 Kubernetes Operator

sphynx-worker-controller, SphynxConnection CRD, SphynxWorkerDeployment CRD, three rollout strategies, six version lifecycle states.

### 5.5 API and Validation

OpenAPI 3.1 spec, Swagger UI, gRPC (reserved), Spring REST Docs, contract tests, sphynx-features replay harness, shadow execution divergence detection.

---

## 6. Kernel-Critical Features

The following capabilities SHALL be treated as kernel-critical or kernel-sensitive, regardless of implementation phasing. They directly affect replay correctness, history compatibility, execution safety, or recoverability and MUST be governed with stricter contracts than ordinary backlog items.

| Feature | Impact | Delivery Classification |
|---------|--------|------------------------|
| SideEffect / non-determinism | Replay breaks on non-deterministic ops | Prerequisite to first production release |
| Search attributes / visibility | Foundation for batch ops, monitoring | Prerequisite to first production release |
| Cancellation scopes | Branch cleanup, saga correctness | Prerequisite to enterprise runtime cert |
| Schedule system / entity | Overlap policy needed from first deploy | Prerequisite to enterprise runtime cert |
| Per-node reset | Token model contract change | Design-for day one; build when ready |
| Process instance migration | Token remapping contract change | Design-for day one; build when ready |

**A kernel-critical feature SHALL NOT be treated as optional polish.**

---

## 7. Message Bus Boundary Standard

### 7.1 Core Dependency Rule

**The core execution kernel MUST NOT depend on Kafka availability for token dispatch, token progression, replay, or state persistence.**

### 7.2 Abstraction Rule

All message bus interaction SHALL be placed behind an SPI boundary. That SPI MUST permit pluggable implementations: Kafka, RabbitMQ, in-memory transport, and future adapters.

### 7.3 Preferred Profile

Kafka SHOULD be treated as the preferred reference implementation for: audit streaming, signal distribution, asynchronous fan-out, dead-letter handling, and streaming integrations.

### 7.4 Non-Mandatory Rule

Kafka SHALL NOT be a mandatory dependency of the core execution loop. A PostgreSQL-only deployment profile MUST remain valid.

**The engine MUST NOT fail token dispatch or workflow progression solely because Kafka is unavailable.**

### 7.5 SPI Interface

MessageBusSPI: `publish(topic, AuditEvent)`, `publish(topic, SignalEvent)`, `subscribe(topic, group, handler)`, `publishToDLQ(topic, FailedEvent)`. Implementations: KafkaMessageBus, InMemoryMessageBus, RabbitMQMessageBus.

---

## 8. Delivery Priority Standard

### 8.1 Prerequisite to First Production Release

All 38 Tier 1 features + SideEffect + search attributes + versioning policy + unified API.

### 8.2 Prerequisite to Enterprise Runtime Certification

Cancellation scopes + schedule system / schedule entity.

### 8.3 Design-For Requirements

Per-node reset and process instance migration MUST be designed-for in the token model and persistence schema even if implementation is deferred.

---

## 9. Change Control

Any of the following changes SHALL require formal architecture review:

- Reclassification of a feature between tiers
- Change to the authoritative counts (38 / 55 / 37 / 130)
- Removal of a Tier 1 feature
- Conversion of a preferred dependency into a mandatory core dependency
- Weakening of replay, audit, or deterministic execution guarantees
- Changes to kernel-critical compatibility rules

Architecture change approval SHOULD be recorded in the canonical design baseline.

---

## PART II — TECHNOLOGY STACK & REPOSITORY

---

## 10. Technology Stack (Locked)

| Component | Technology | Version / Policy |
|-----------|-----------|-----------------|
| Language | Java | 21 LTS |
| Framework | Spring Boot | 4.0.4 (no Boot 3 compat) |
| Core Framework | Spring Framework | 7.x |
| Module Boundaries | Spring Modulith | 2.0 |
| Primary Database | PostgreSQL | 16+ |
| Cache / Fast Locks | Redis | 7+ |
| Message Bus | Kafka (via SPI, not mandatory) | 3.x |
| Integration | Apache Camel | 4.x |
| Timer / Scheduler | Quartz | 2.3+ |
| DB Migrations | Flyway | 11.x |
| Serialization | Jackson | 3 (Boot 4 default) |
| Null Safety | JSpecify | 1.0.0 |
| Security | Spring Security | 7 + OAuth2 |
| HTTP Clients | @HttpExchange | Typed declarative |
| Observability | OpenTelemetry | spring-boot-starter-opentelemetry |
| Metrics | Micrometer + Prometheus | Via Boot Actuator |
| Build | Gradle | 8.x |
| Containers | Docker + K8s + Helm | Cloud-native |

Policies: Virtual threads default-on; Spring Cloud 2025.1.1; Spring AI on hold; gRPC reserved; Undertow excluded; classic starters excluded; Jackson 3 only.

---

## 11. Repository Structure

### 11.1 Four Repositories

- **sphynx-engine —** Native runtime. Modules: schema/postgres, gateway, engine, dispatch, signals, visibility, system, executors, connectors, codec, messagebus.
- **sphynx-api —** Contracts. Protobuf + OpenAPI + buf. No runtime.
- **sphynx-features —** Replay harness. Three-check CI pipeline.
- **sphynx-worker-controller —** K8s operator. Java Operator SDK + Spring Boot 4.

### 11.2 Build Sequence (Locked)

1. sphynx-engine-core — execution runtime
2. sphynx-testing — embedded test server + history fixtures (alongside engine)
3. sphynx-spring-boot-starter — Spring wiring, worker autoconfigure
4. sphynx replay CLI + history fixture suite — compatibility gate
5. sphynx-ui + operator console — after contracts stable

---

## PART III — CORE ENGINE IMPLEMENTATION

---

## 12. Dispatcher & Token Model

### 12.1 Token States

```
READY → CLAIMED → EXECUTING → COMPLETED → (create next tokens)
                                         | FAILED → RETRY_SCHEDULED → READY
                                         | WAITING (user/timer/signal/update)
                                         | CANCELLED
```

### 12.2 Claim Query

```sql
SELECT * FROM execution_token
WHERE status = 'READY'
  AND queue = :queue
  AND tenant_id = :tenant
ORDER BY priority ASC, enqueued_at ASC
FOR UPDATE SKIP LOCKED
LIMIT :batch
```

Priority 1 = highest; Priority 5 = lowest (default 3). Fairness: weighted round-robin across fairnessKey values.

### 12.3 Execution Flow

DSL upload → validate → store → start instance → create token → dispatcher claims → executor runs → write result → create next tokens → park for waits → audit publish → retry/compensate on failure → EndEvent completes.

### 12.4 Deterministic Contract

No side effects in interpreter. All IO in executors. SideEffect records non-deterministic outputs. Idempotency keys on external calls.

### 12.5 Timeout Taxonomy

Schedule-To-Start (queue wait), Start-To-Close (execution), Schedule-To-Close (total), Heartbeat (liveness).

---

## 13. Foundational Node Executors

- **UserTask:** human_task table, WAITING park, role/dynamic/round-robin assignment, SLA timer, form schema (Tier 2), multi-instance, named outcomes.
- **ServiceTask:** Camel route generation, input→invoke→parse→output pipeline, custom object schema, codec, retries, circuit breaker.
- **RuleTask:** Decision tables, SpEL/Drools/OPA, external REST, onTrue/onFalse or onDecision routing.
- **AIDecisionTask:** External model REST, inline Python/JS/Groovy, threshold routing, LIME/SHAP explainability, model versioning, A/B testing. Behind StepExecutor SPI.
- **ParallelGateway:** Branch token spawning, waitAll/waitAny/count(N) join, cancellation policy, maxParallelNodes.
- **ExclusiveGateway:** Ordered condition evaluation, first match, default branch.
- **TimerTask:** timer_wheel entry, duration/cron/absolute, 1s scan, WAITING→READY on fire.
- **ScriptTask:** Groovy/JS/Python, SideEffect mode for replay safety.
- **SignalTask:** signal_subscription, WAITING park, REST/message bus wake, SLA timeout.
- **CompensationTask:** Rollback logic, reverse-order invocation, idempotent.
- **SubProcess:** Child workflow, parent close policy (TERMINATE/REQUEST_CANCEL/ABANDON), multi-instance (Tier 2).
- **NotificationTask:** email/SMS/Slack/webhook, template rendering, fire-and-forget.
- **Start/EndEvent:** Entry/exit; EndEvent sets terminal status, triggers compensation.

---

## 14. Enterprise Node Executors (Tier 2)

- **ConnectorTask:** Dynamic Camel, 350+ connectors, hot reload, EIP patterns.
- **CrossTenantTask:** Endpoint allowlist, sync/async dispatch, isolated audit per tenant.
- **AdHocSubprocess:** Dynamic task selection, AI agent integration, activation conditions.
- **EscalationEvent:** Non-interrupting scope escalation, caught by nearest parent.
- **AuditTask:** Structured audit event, regulatory checkpoints.
- **ManualTask:** Action tracking without form system.
- **ExternalTask:** External worker poll pattern.

---

## 15. Workflow Definition Specification

### 15.1 Authoring Modes

Sphynx supports three co-equal workflow authoring modes, all of which are translated into a common internal WorkflowDefinition and executed identically by the runtime:

- **JSON DSL (v1.1) —** Canonical developer-facing definition format. Full feature coverage. Primary for developers, CI/CD pipelines, and programmatic workflow generation. This is the canonical format: all other authoring modes produce JSON DSL or the equivalent internal model.
- **BPMN 2.0 —** Visual standards-based authoring mode. Primary for business analysts, process architects, and organizations with existing BPMN investments. BPMN diagrams are parsed by the BPMN XML parser into the same internal WorkflowDefinition model. Supports authoring, import/export, and execution mapping via the Sphynx execution model.
- **UI Workflow Designer —** Low-code visual authoring mode via the Sphynx Modeler (Web/Desktop). Primary for business users, low-code teams, and rapid prototyping. The UI Designer emits canonical JSON DSL. Draft state may be persisted in the Modeler, but deployed definitions are always canonical JSON DSL.

### 15.2 Ingestion Architecture

The engine gateway contains two parsers producing the same internal WorkflowDefinition: a JSON DSL parser (primary) and a BPMN XML parser. The UI Designer emits canonical JSON DSL directly, so no third parser is required. The runtime is authoring-agnostic — it never distinguishes how a workflow was authored.

### 15.3 JSON DSL Envelope (v1.1)

Envelope: `sphynxDslVersion`, `engine`, `workflow { id, name, version, owner, tenant, tags, timezone, expressionLang, determinism, versioningPolicy, concurrency, runtime, start, triggers, signals, variables, searchAttributes, updateHandlers, globalListeners, security, audit, nodes, globalErrorHandlers, metadata }`.

### 15.4 Universal Node Attributes

Every node supports: `id`, `type`, `sla`, `retries`, `retry` (policy), `timeouts` (four-taxonomy), `inputMapping`, `outputMapping`, `priority`, `queue`, `metadata`, `codec`, `listeners`, `outcomes`, `sideEffect`, `runtime`.

### 15.5 Supported Definition Formats

JSON DSL (canonical), YAML (auto-converted to JSON at runtime), BPMN 2.0 (parsed to internal model).

---

## 16. Persistence (15 Tables)

| Table | Purpose |
|-------|---------|
| workflow_definition | Versioned DSL storage |
| workflow_instance | Instance state + variables + pinned_version |
| execution_token | Per-node execution tracking |
| node_result | Completed step outcomes |
| timer_wheel | Scheduled timers/deadlines |
| signal_subscription | Event subscriptions |
| human_task | User task inbox |
| audit_log | Immutable events + hash chain |
| migration_marker | Version migration tracking |
| instance_snapshot | Replay acceleration |
| schedule_definition | Schedule entities |
| batch_job | Batch operation tracking |
| search_attribute | Custom visibility index |
| endpoint_registry | Cross-tenant endpoints |
| config_variable | Cluster/tenant configuration |

Storage tiers: PostgreSQL (MUST standalone), Redis (cache/locks), MessageBusSPI (streaming), S3/Cassandra (cold).

---

## 17. Replay & Event Sourcing

20 audit event types: INSTANCE_STARTED through INSTANCE_CANCELLED. Hash chain: SHA-256(prev_hash + event_type + payload + seq). WORM-compliant. Replay modes: state rebuild, shadow execution, full re-run, snapshot-accelerated. SideEffect: record once, return cached.

---

## 18. Timer, Retry, SLA, Signals, Parallelism, Compensation

- **Timers:** Quartz + timer_wheel, 1s scan, duration/cron/absolute.
- **Retries:** Per-node policy, exponential/linear/fixed backoff.
- **SLA:** ISO 8601 per node, escalation on breach, priority boost.
- **Signals:** Subscription table, REST/message bus delivery, correlation key matching.
- **Parallelism:** Fork/join (waitAll/waitAny/count), cancellation scopes, maxParallelNodes.
- **Compensation:** Tier 1 forward saga → Tier 2 BPMN-standard events.

---

## 19. Human Tasks, AI/ML, Camel, Security, Scaling

- **Human Tasks:** Inbox, assignment, SLA, delegation, listeners (Tier 2), forms (Tier 2).
- **AI/ML:** External models, inline code, threshold, explainability, NLP.
- **Camel:** Connector plane only, runtime route generation, hot reload, 350+ connectors, ultra-low-latency bypass.
- **Security:** JWT/OAuth2, RBAC/ABAC, OPA, AES-256, TLS 1.3, secrets, signatures.
- **Compliance:** SOX/HIPAA/GDPR/PCI templates, WORM audit, hash-chain.
- **Multi-tenancy:** Isolation, quotas, sandboxing.
- **Scaling:** Vertical-first, horizontal K8s, sharding, 100K+ tasks/hour.
- **Deployment:** K8s/Helm, Docker, on-prem/hybrid/cloud, blue/green/canary, geo-redundancy.

---

## PART IV — API SURFACE

---

## 20. Unified REST API

### Workflow Definitions

```
POST  /api/v1/definitions              # Deploy
GET   /api/v1/definitions              # List
GET   /api/v1/definitions/{id}:{version}  # Get
```

### Workflow Instances

```
POST  /api/v1/instances                # Start
GET   /api/v1/instances/{id}           # Status
POST  /api/v1/instances/{id}/signal    # Signal
GET   /api/v1/instances/{id}/query     # Query
POST  /api/v1/instances/{id}/update    # Update (Tier 2)
POST  /api/v1/instances/{id}/cancel    # Cancel
POST  /api/v1/instances/{id}/terminate # Terminate
POST  /api/v1/instances/{id}/reset     # Reset (Tier 2)
GET   /api/v1/instances/{id}/history   # History
```

### Human Tasks

```
GET   /api/v1/tasks                    # Inbox
POST  /api/v1/tasks/{id}/claim         # Claim
POST  /api/v1/tasks/{id}/complete      # Complete
POST  /api/v1/tasks/{id}/delegate      # Delegate
```

### Schedules (Tier 2)

```
POST  /api/v1/schedules                # Create
PUT   /api/v1/schedules/{id}           # Update
POST  /api/v1/schedules/{id}/pause     # Pause
POST  /api/v1/schedules/{id}/resume    # Resume
POST  /api/v1/schedules/{id}/trigger   # Trigger
```

### Batch Operations (Tier 2)

```
POST  /api/v1/batch/start              # Start batch
GET   /api/v1/batch/{id}/status        # Status
DELETE /api/v1/batch/{id}              # Cancel
```

### Admin

```
PUT   /admin/queue/{name}/config       # Queue config
PUT   /admin/config/variables          # Config variables
POST  /admin/endpoints                 # Register endpoint
POST  /admin/migrate                   # Trigger migration
POST  /admin/versions                  # Register version
```

### Docs

```
GET   /api/v1/docs                     # Swagger UI
GET   /api/v1/openapi.json             # OpenAPI 3.1
```

---

## PART V — IMPLEMENTATION ROADMAP

---

## 21. Delivery Phases

| Phase | Scope | Duration |
|-------|-------|----------|
| 1 | All 38 Tier 1 + SideEffect + search attributes + versioning + unified API | Weeks 1–6 |
| 1.5 | Cancellation scopes + schedule system + MessageBusSPI | Weeks 7–8 |
| 2 | Kafka, priority/fairness, update API, codec, multi-instance, bench, testing | Weeks 9–14 |
| 3 | Human task enhancements, admin, RBAC, Operate, cross-tenant, reset, batch, migration, listeners, forms, compensation, variables | Weeks 15–20 |
| 4 | Ad-hoc subprocess, analytics, document store, escalation, simulation | Weeks 21–28 |
| 5 | Modeler, marketplace, SaaS, SDKs, Copilot, IDE debugger, Terraform | Weeks 29+ |

---

## PART VI — FINAL NORMATIVE STATEMENT

---

## 22. Closing Statement

Sphynx SHALL be governed as a workflow platform with:

- a 130-feature capability catalog for breadth,
- a three-tier architecture model (38 / 55 / 37) for implementation rigor,
- a replay-safe execution kernel,
- an audit-first design,
- a policy-governed runtime, and
- a message-bus-agnostic core.

Tier 1 SHALL contain engine-correctness features. Tier 2 SHALL contain enterprise runtime features. Tier 3 SHALL contain ecosystem and tooling features.

Kafka SHOULD be used as a preferred integration profile, but SHALL NOT be required for core execution.

**This specification SHALL serve as the authoritative basis for architecture review, implementation planning, roadmap sequencing, conformance assessment, and external positioning.**

---

> *SPHYNX — Part of the SPHUTA Platform | Pommala LLC*
> *Intelligent, Compliant, and Scalable Enterprise Automation*
