# SPHYNX

**Intelligent, Compliant, and Scalable Enterprise Automation**

SPHYNX is the orchestration and workflow engine that powers intelligent automation across the [SPHUTA](https://github.com/pommala) platform.

> **SPHUTA** — Smart Platform for High-Performance Unified Transaction and Accounting
> *One Platform, Seamless Unified Transactions.*

---

## What is SPHYNX?

SPHYNX is a fully native workflow orchestration engine built for long-running, stateful, human-and-system workflows in compliance-sensitive domains. It is designed for finance, healthcare, insurance, government, logistics, and any industry that demands deterministic execution, immutable audit, and policy-governed automation.

SPHYNX is not a wrapper around any external workflow runtime. It implements proven orchestration patterns entirely in its own codebase, with Apache Camel as the only external engine dependency for the connector and integration layer.

---

## Three Authoring Modes

SPHYNX supports three co-equal workflow authoring modes, all translated into a common internal workflow definition and executed identically by the runtime:

| Mode | Primary Audience | Description |
|------|-----------------|-------------|
| **JSON DSL** | Developers, CI/CD | Canonical definition format with full feature coverage |
| **BPMN 2.0** | Business analysts, architects | Visual standards-based modeling with import/export |
| **UI Designer** | Business users, low-code teams | Drag-and-drop visual builder via Sphynx Modeler |

YAML is supported as an equivalent authoring format for the JSON DSL.

---

## Key Capabilities

- **Durable Execution** — Workflows run from milliseconds to months, surviving restarts, deploys, and worker loss
- **Deterministic Replay** — Workflow state is reproducible from immutable, hash-chain-verified audit logs
- **Human-in-the-Loop** — Role-based task inbox with SLA enforcement, delegation, escalation, forms, and comments
- **AI/ML Decisions** — First-class AIDecisionTask with model versioning, threshold routing, and explainability
- **Rules & Policy** — Pluggable rule engines (SpEL, Drools, OPA), decision tables, and external rule services
- **Parallel Orchestration** — Fork/join with waitAll, waitAny, and k-of-n semantics; cancellation scopes
- **Saga & Compensation** — Forward compensation chains and BPMN-standard compensation events
- **350+ Connectors** — Apache Camel integration layer with runtime route generation from DSL
- **Event-Driven** — Signals, webhooks, Kafka events with durable correlation and subscription
- **Compliance-First** — SOX, HIPAA, GDPR, PCI templates; WORM-compliant audit; digital signatures
- **Multi-Tenant** — Logical isolation at data, execution, and connector levels with per-tenant quotas
- **Scalable** — Vertical-first with automatic horizontal scale-out; 100K+ tasks/hour target

---

## Architecture

SPHYNX is organized into nine logical layers:

```
┌─────────────────────────────────────────────────────┐
│                   Tooling Layer                      │
│   Modeler · Console · Operate · Tasklist · CLI       │
├─────────────────────────────────────────────────────┤
│                  Control / API Layer                  │
│            REST · gRPC · OpenAPI · OAuth2             │
├───────────────┬─────────────────┬───────────────────┤
│  Human Task   │  Orchestration  │   Decision Layer   │
│    Layer      │     Core        │  Rules · OPA · AI  │
│  Inbox · SLA  │ Dispatcher ·    │  DMN · ML Models   │
│  Escalation   │ Tokens · Retry  │                    │
│               │ Timers · Saga   │                    │
├───────────────┴─────────────────┴───────────────────┤
│               Execution Runtime                      │
│         Workers · Executors · Queues                 │
├─────────────────────────────────────────────────────┤
│              Integration Layer                       │
│     Apache Camel · Connectors · EIP Patterns         │
├─────────────────────────────────────────────────────┤
│           Persistence & Audit Layer                  │
│    PostgreSQL · Redis · Kafka (SPI) · S3 · Audit     │
└─────────────────────────────────────────────────────┘
```

---

## Technology Stack

| Component | Technology |
|-----------|-----------|
| Language | Java 21 LTS |
| Framework | Spring Boot 4.0.4, Spring Framework 7 |
| Module Boundaries | Spring Modulith 2.0 |
| Primary Database | PostgreSQL 16+ |
| Cache | Redis 7+ |
| Message Bus | Kafka 3.x (via SPI, not mandatory) |
| Integration | Apache Camel 4.x |
| Scheduler | Quartz 2.3+ |
| Migrations | Flyway 11.x |
| Serialization | Jackson 3 |
| Security | Spring Security 7, OAuth2 |
| Observability | OpenTelemetry, Micrometer, Prometheus |
| Build | Gradle 8.x |
| Deployment | Docker, Kubernetes, Helm |

---

## Repository Structure

```
sphynx-engine/          # Native execution runtime
├── schema/postgres/     # Flyway migrations
├── gateway/             # REST/gRPC API, auth, rate limiting
├── engine/              # State, history, timers, queuing
├── dispatch/            # Task queue dispatch, priority, fairness
├── signals/             # Signal correlation, event waking
├── visibility/          # Search attributes, query index
├── system/              # Schedules, batch jobs
├── executors/           # All node executor implementations
├── connectors/          # Apache Camel integration
├── codec/               # Payload encryption, compression
└── messagebus/          # SPI: Kafka, InMemory, RabbitMQ

sphynx-api/              # Protobuf + OpenAPI contracts
sphynx-features/         # History compatibility test harness
sphynx-worker-controller/  # Kubernetes operator (Java Operator SDK)
```

---

## Node Types

| Node | Purpose |
|------|---------|
| StartEvent | Workflow entry point |
| UserTask | Human approval with roles, forms, SLA, escalation |
| ServiceTask | REST, gRPC, DB, queue invocation |
| ConnectorTask | Camel-backed integration with 350+ connectors |
| RuleTask | Decision tables, OPA, SpEL, Drools |
| AIDecisionTask | ML/LLM invocation with threshold and explainability |
| ParallelGateway | Fork/join with waitAll, waitAny, count(N) |
| ExclusiveGateway | Condition-based branching |
| TimerTask | Delays, reminders, deadlines, cron |
| ScriptTask | Embedded Groovy, JavaScript, Python |
| SignalTask | External event wait with correlation |
| NotificationTask | Email, SMS, Slack, webhook dispatch |
| SubProcess | Child workflow with parent close policy |
| CompensationTask | Saga rollback and BPMN compensation |
| CrossTenantTask | Cross-namespace RPC via endpoint registry |
| AdHocSubprocess | Dynamic task selection for AI agents |
| AuditTask | Structured audit event recording |
| EndEvent | Workflow completion |

---

## Quick Start

```bash
# Clone
git clone https://github.com/pommala/sphynx-engine.git
cd sphynx-engine

# Build
./gradlew build

# Run with embedded Postgres + Redis (dev profile)
./gradlew bootRun --args='--spring.profiles.active=dev'

# Run tests
./gradlew test
```

---

## API

SPHYNX exposes a unified REST API:

```
POST   /api/v1/definitions          # Deploy workflow definition
POST   /api/v1/instances            # Start workflow instance
GET    /api/v1/instances/{id}       # Get instance status
POST   /api/v1/instances/{id}/signal   # Send signal
POST   /api/v1/instances/{id}/update   # Sync request-response
GET    /api/v1/tasks                # List task inbox
POST   /api/v1/tasks/{id}/complete  # Complete task
GET    /api/v1/docs                 # Swagger UI
GET    /api/v1/openapi.json         # OpenAPI 3.1 spec
```

---

## Workflow Example (JSON DSL)

```json
{
  "sphynxDslVersion": "1.1",
  "engine": "sphynx",
  "workflow": {
    "id": "loan-approval",
    "name": "Loan Approval",
    "version": 1,
    "start": "collectApplication",
    "variables": {
      "customerId": "string",
      "loanAmount": "number"
    },
    "nodes": [
      {
        "id": "collectApplication",
        "type": "UserTask",
        "role": "loan_officer",
        "outcomes": { "submitted": "creditCheck" }
      },
      {
        "id": "creditCheck",
        "type": "ServiceTask",
        "service": "CreditScoreAPI",
        "inputMapping": { "customerId": "${input.customerId}" },
        "outputMapping": { "creditScore": "${result.score}" },
        "outcomes": { "success": "decisionGate" }
      },
      {
        "id": "decisionGate",
        "type": "RuleTask",
        "rule": "creditScore > 700",
        "onTrue": "autoApprove",
        "onFalse": "managerReview"
      },
      {
        "id": "autoApprove",
        "type": "NotificationTask",
        "channel": "email",
        "template": "loan-approved",
        "outcomes": { "sent": "endApproved" }
      },
      {
        "id": "managerReview",
        "type": "UserTask",
        "role": "manager",
        "sla": "P2D",
        "outcomes": {
          "approved": "endApproved",
          "rejected": "endRejected"
        }
      },
      {
        "id": "endApproved",
        "type": "EndEvent",
        "status": "success"
      },
      {
        "id": "endRejected",
        "type": "EndEvent",
        "status": "rejected"
      }
    ]
  }
}
```

---

## Documentation

| Document | Purpose |
|----------|---------|
| [Complete Implementation Specification](docs/Sphynx_Complete_Implementation_Specification.md) | Normative architecture standard (38/55/37 tiered model, 130 features) |
| [Finalized Consolidated Spec](docs/Sphynx_Workflow_Engine_Finalized_Consolidated_Spec.md) | Narrative architecture reference |
| [New Features Deep Dive](docs/Sphynx_New_Features_Deep_Dive.md) | Detailed specs for advanced features |
| [DSL Schema](schemas/sphynx-dsl-schema-v1_0.json) | JSON Schema for DSL validation |
| [DSL Example](schemas/sphynx-dsl-example-v1_0.json) | Full loan approval example |

---

## Target Industries

| Industry | Use Cases |
|----------|----------|
| Financial Services | Loan approvals, fraud detection, compliance workflows, credit risk |
| Healthcare | Patient intake, claims processing, HIPAA-compliant automation |
| Insurance | Underwriting, claims adjudication, regulatory reporting |
| Government | Permitting, benefits processing, audit-ready workflows |
| Logistics | Shipment orchestration, tracking, exception handling |
| Technology / SaaS | Workflow orchestration, billing automation, developer portals |

---

## Design Principles

- **Durable by default** — workflows survive restarts, deploys, and worker loss
- **Deterministic by contract** — state reproducible from persisted execution facts
- **Separation of concerns** — core, connectors, rules, tasks, and UI evolve independently
- **Policy-first governance** — RBAC, ABAC, OPA guards, deployment approvals
- **Business-friendly authoring** — three modes ensure accessibility beyond developers
- **Code-extensible, not code-dependent** — custom handlers without code changes for standard workflows
- **Observability as product** — audit, traces, metrics, SLA visibility, and replay built-in
- **Pluggable ecosystem** — connectors, node types, rule providers, and message bus extensible via SPI

---

## License

Proprietary. Copyright © 2026 Pommala LLC. All rights reserved.

---

> *SPHYNX is SPHUTA's intelligent workflow, decisioning, and orchestration engine.*
> *Pommala LLC*
