# Sphynx Workflow Engine

## Finalized Consolidated Architecture & Specification

> **Document Status**
> Status: Finalized consolidated specification
> Baseline date: March 22, 2026
> Scope: Workflow engine only, with platform context included where it directly affects engine behavior

This document consolidates the Sphynx workflow-engine material into a single final reference. The goal is to remove ambiguity between earlier architecture explorations and provide one canonical direction for design, implementation, review, and future extension.

Where earlier materials explored multiple approaches, this document records the final normalized decision so engineering, product, and architecture work can proceed against one stable baseline.

---

## 1. Final Positioning

Sphynx is the proprietary workflow and orchestration engine for the SPHUTA ecosystem. It is BPMN-inspired, business-friendly, low-code first, code-extensible, audit-heavy, and designed for long-running, stateful, human-and-system workflows in compliance-sensitive domains.

The engine supports three workflow authoring modes: JSON DSL as the canonical execution model, BPMN 2.0 for standards-based visual modeling, and a visual UI designer for low-code authoring. YAML is an equivalent authoring format for the DSL. All three modes are translated into a common internal workflow definition and executed identically by the runtime. Human tasks, service integrations, rules, AI decisions, timers, signals, compensations, and replayable audit are first-class concerns, not add-ons.

The engine is intended to power finance, billing, HR, compliance, approvals, case-processing, notification, and cross-service orchestration scenarios across SPHUTA modules.

---

## 2. Canonical Decisions After Reconciliation

| Topic | Finalized Decision |
|-------|-------------------|
| Runtime architecture | Native Sphynx runtime is the canonical target. Earlier hybrid designs are treated as exploration lineage, not the product definition. |
| Definition model | Three authoring modes: JSON DSL (canonical), BPMN 2.0 (visual standards-based), and UI Workflow Designer (low-code). YAML is equivalent for DSL authoring. All modes compile to one internal model. |
| Integration layer | Apache Camel is the preferred connector and EIP backbone, but it is outside the core state/execution kernel. |
| Rules and policy | OPA, decision tables, and rule-service adapters are first-class extensions; rule evaluation must be pluggable and policy-governed. |
| AI posture | AI is additive, optional, and policy-bound. Core workflow execution must not depend on AI availability. |
| Message bus | Event-bus integration is pluggable. Kafka is the preferred default for audit/event streaming, but the core engine must remain operable without making Kafka a hard execution dependency. |
| Persistence | Postgres is the primary durable store; Redis is an accelerator for locks/cache/hot-path lookups; cold or analytical history can be streamed to S3/Cassandra/warehouse. |
| Scaling stance | Vertical-first scaling is the default operational strategy; horizontal worker scale-out is enabled beyond configured thresholds. |
| Replay model | Deterministic replay from immutable audit is mandatory and is part of the engine contract, not only an operational tool. |
| Governance | Versioned definitions, safe migrations, promotion controls, approval workflows, and change history are mandatory for enterprise usage. |

---

## 3. Design Principles

- **Durable by default:** workflows may run from milliseconds to months, and must survive restarts, deploys, and worker loss.
- **Deterministic by contract:** workflow state must be reproducible from persisted execution facts.
- **Separation of concerns:** orchestration core, connectors, rule/AI services, human-task service, and UI tooling must evolve independently.
- **Policy-first governance:** every workflow can be constrained by RBAC, ABAC, OPA guards, masking, classification, and deployment approval rules.
- **Business-friendly authoring:** three authoring modes (JSON DSL, BPMN, UI designer) ensure definitions remain understandable by analysts and architects, not only developers.
- **Code-extensible, not code-dependent:** custom handlers are supported, but standard business workflows should not require code changes.
- **Observability as product capability:** audit, traces, metrics, queue lag, SLA visibility, and replay are part of the platform.
- **Pluggable ecosystem:** connectors, node types, resolvers, rule providers, and deployment profiles must be extensible without redesigning the DSL.

---

## 4. Engine Scope

In scope are workflow definition management, validation, deployment, execution, routing, human tasks, timer handling, signal correlation, compensation, audit, replay, security enforcement, observability, and operational administration.

Out of scope for the core engine are domain-specific business UIs, bespoke connector logic that should live in connector packages, and ultra-low-latency microsecond paths that are better handled in-process and only reported back to the engine for audit or follow-up orchestration.

---

## 5. Canonical Logical Architecture

The finalized engine architecture is composed of nine logical layers:

- **Definition Layer** — stores workflow packages, versions, schemas, policies, calendars, and deployment metadata.
- **Control/API Layer** — secure REST and optionally gRPC endpoints for definition, deployment, instance, task, signal, and administrative actions.
- **Orchestration Core** — dispatcher, token manager, transition evaluator, retry engine, timer engine, compensation manager, and replay model.
- **Execution Runtime** — workers and executors that run concrete node types against queued runnable work.
- **Human Task Layer** — task inbox, claim/release/delegate/complete flows, comments, attachments, escalations, and reminders.
- **Integration Layer** — connector registry plus Camel-backed routes/adapters for HTTP, gRPC, Kafka, JMS, DB, file, email, webhook, and enterprise systems.
- **Decision Layer** — rule engine adapters, OPA policy checks, DMN-style tables, and AI/ML decision endpoints.
- **Persistence and Audit Layer** — Postgres, optional Redis, append-only audit stream, object storage/warehouse sinks, snapshots, and history views.
- **Tooling Layer** — modeler (visual UI authoring), console, operate/tasklist/optimize style capabilities, CLI, validation tools, migration tools, and test harnesses.

> **Architecture stance:** Sphynx is finalized here as its own workflow runtime with native state, timers, queues, replay, and policy enforcement.

---

## 6. Core Runtime Components

| Component | Responsibility |
|-----------|---------------|
| DefinitionService | Create, validate, version, publish, deprecate, and retrieve workflow packages. |
| DeploymentService | Promote versions across environments, apply approvals, and activate definitions. |
| InstanceService | Start, resume, cancel, retry, migrate, and query workflow instances. |
| Dispatcher | Evaluate runnable tokens, route next work, and honor dependency/join rules. |
| ExecutorRegistry | Map node type to StepExecutor implementation. |
| Retry/Backoff Engine | Compute retry schedules per node/policy and enforce limits. |
| Timer Engine | Schedule deadlines, reminders, delays, and wakeups. |
| Signal Bus / Correlator | Create subscriptions and wake parked tokens when matching events arrive. |
| Saga / Compensation Manager | Track compensable steps and execute reverse actions on failure or cancellation. |
| Human Task Service | Manage inbox, assignments, comments, escalations, and SLA timers for manual work. |
| AuditPublisher | Write immutable execution events to DB and optional event streams. |
| ReplayService | Rebuild state, compare actual vs replayed behavior, and support forensic analysis. |
| Policy Guard | Apply JWT, RBAC, ABAC, OPA, and tenant isolation checks at API and node boundaries. |

---

## 7. Execution Model

The execution model is token-based and event-driven. A deployed workflow definition is compiled into an executable graph. When a workflow instance starts, the engine creates an instance record, materializes initial variables, emits an INSTANCE_STARTED audit event, and places a token on the start node.

1. The dispatcher evaluates active tokens and identifies nodes that are runnable under dependency, join, and policy conditions.
2. Runnable work is enqueued against the appropriate queue, priority, and concurrency policy.
3. Workers pull a unit of work, invoke the matching executor, and persist the node result together with audit events.
4. The engine then applies transitions, creates downstream tokens, or parks execution when waiting for user action, timer fire, external signal, or sub-workflow completion.
5. Terminal completion is reached only when all live tokens are resolved and the workflow reaches a completed, rejected, cancelled, compensated, or failed final state.

This execution model supports long-running workflows, partial progress persistence, safe restart, and deterministic replay because each step is materialized as durable state rather than kept only in memory.

---

## 8. Workflow Instance States

| State | Meaning |
|-------|---------|
| DRAFT | Created but not yet started; mostly used for staging or pre-validation. |
| RUNNING | At least one token is active or parked and the workflow can still advance. |
| WAITING | No runnable work at the moment; waiting on user input, timer, or external signal. |
| SUSPENDED | Paused by operator/policy or maintenance action. |
| COMPLETED | Reached successful terminal outcome. |
| REJECTED | Reached business terminal outcome with negative decision. |
| CANCELLED | Stopped by caller or operator before normal completion. |
| FAILED | Stopped due to unhandled technical or policy failure. |
| COMPENSATED | Forward path failed and all configured compensation steps completed. |
| MIGRATED | Instance state carried to a newer compatible definition version. |

---

## 9. Definition and DSL Model

A workflow package may contain workflow, connectors, calendars, policies, DMN artifacts, and payload schemas. JSON is the canonical machine representation. YAML is the equivalent authoring view. BPMN/DMN are interoperability and modeling formats. The UI Workflow Designer emits canonical JSON DSL.

At minimum, every workflow definition must declare an id, version, start node, and node collection. Most production workflows will also declare variables, execution settings, error policies, security, audit behavior, and connector references.

### 9.1 Top-Level Definition Contract

- **Identity:** workflowId/id, name, version, description, tags, owner, tenant mode, classification.
- **Execution:** mode, queue/affinity, priority, idempotency key, concurrency, rate limits, backpressure, scaling strategy.
- **Variables:** schema/spec, defaults, optional required set, masking rules.
- **Behavioral controls:** retry defaults, timeout defaults, global SLA, business calendar, unhandled error policy.
- **Security:** auth mode, claims mapping, RBAC/ABAC, policy guards, encryption/masking declarations.
- **Audit/observability:** audit level, redaction paths, event-stream sink, logging format, correlation key, metrics set.
- **Graph body:** start node and node list.

### 9.2 Supported Definition Formats

| Format | Role in the Finalized Model |
|--------|---------------------------|
| JSON DSL | Primary, canonical execution format; best for APIs, storage, validation, and tooling interoperability. |
| YAML | Equivalent human-friendly authoring format; converted to canonical internal model at deploy time. |
| BPMN 2.0 | Visual standards-based authoring mode; import/export and execution mapping via the Sphynx execution model. |
| DMN / Decision tables | Decision authoring and import format for rules and routing. |
| UI Workflow Designer | Low-code visual authoring via Sphynx Modeler (Web/Desktop); emits canonical JSON DSL. Primary for business users and rapid prototyping. |
| Code-first SDK | Optional power-user interface for custom workers or specialized handlers, not the primary business authoring path. |

---

## 10. Node Model

Every node has a stable id and type. Most nodes also support name, description, input mapping, output mapping, retry, timeout, runtime overrides, metadata, and transition definitions. Node implementations must remain extensible without breaking previously deployed definitions.

| Node Type | Purpose |
|-----------|---------|
| StartEvent | Explicit workflow start entry point. |
| UserTask | Manual approval or data-entry step with assignment, forms, SLAs, delegation, and escalations. |
| ServiceTask | Call to REST, gRPC, DB, file, queue, or internal service integration. |
| ConnectorTask | Connector-backed integration step using registered connectors and credentials. |
| RuleTask / PolicyCheck | Rule or policy evaluation using decision tables, OPA, or external rule services. |
| AIDecisionTask | AI/ML or scoring decision with thresholding, audit metadata, and policy fences. |
| CodeTask / ScriptTask | Pure deterministic logic step for transformations or advanced routing. |
| ParallelGateway | Fork/join, waitAll, waitAny, or k-of-n merge behavior. |
| ExclusiveGateway / Switch | Condition-based branching. |
| ForEach / MultiInstance | Iterative or bulk execution over collections. |
| Timer / TimerTask / TimerEvent | Delay, reminder, deadline, periodic fire, or SLA wakeup. |
| SignalTask / SignalCatch / EventTask | Wait for or emit asynchronous events. |
| NotificationTask | Email, SMS, Slack, webhook, or in-app notification dispatch. |
| SubProcess / SubWorkflow | Call a child workflow and optionally wait for completion. |
| Compensation / isCompensation | Reverse action linked to a previous forward step. |
| ExternalTask | Delegated work for external workers or RPA-like consumers. |
| EndEvent | Terminal outcome node with status/result payload. |

### 10.1 Mandatory Transition Rules

- The start reference must resolve to an existing node id.
- All transition targets must reference valid node ids.
- No orphan nodes are allowed unless explicitly declared as detached compensation or reusable bodies.
- ParallelGateway k-of-n joins must define a valid threshold within the branch count.
- ForEach and multi-instance collections must resolve to array-like data at runtime.
- Compensation targets must reference a completed compensable step or subprocess.

### 10.2 Determinism Rules

- CodeTask and ScriptTask logic must be deterministic given the same state and inputs.
- Clock, random, thread-local ambient state, and hidden network calls are disallowed unless explicitly provided through engine-controlled inputs or signals.
- External side effects must occur only through ServiceTask, ConnectorTask, NotificationTask, EventTask, or explicit integration adapters.
- External effects must be idempotent or protected by idempotency keys.
- Replay mode must never re-emit unsafe side effects unless explicitly authorized for controlled re-run behavior.

---

## 11. Runtime Controls

The engine supports global defaults and per-node overrides for concurrency, max in-flight limits, queue capacity, thread counts, priority, rate limits, CPU/memory hints, rejection policy, and context propagation. The finalized scaling posture is vertical-first with controlled horizontal expansion.

| Control | Meaning |
|---------|---------|
| maxParallelNodes | Global ceiling for concurrent fan-out branches within one workflow instance or node cluster. |
| workflowConcurrency | Max simultaneously active workflow executions or dispatch slots for a definition. |
| threadPools | Named pools for default and node-type-specific execution lanes. |
| queueCapacity | Bounded queue size per runtime pool or node. |
| rateLimit | RPS and burst control for service-heavy nodes or whole workflows. |
| priority / slaBoost | Bias dispatch toward urgent or breached work. |
| backpressure | buffer, block, or drop strategy when runtime pressure rises. |
| allowNodeOverrides | Lets a node override inherited runtime values when justified. |

---

## 12. Retry, Timeout, Timer, and Signal Semantics

Retries are policy-driven and may be fixed, exponential, or jittered. Timeouts can include execution timeout, heartbeat timeout, schedule-to-start style guardrails, and SLA deadlines. Timers can represent delays, reminders, escalations, and archival/cleanup events. Signals are correlated external events that awaken parked execution.

- Retry policies define max attempts, initial delay, max delay, retryable categories, and non-retryable categories.
- Node timeouts override workflow defaults where necessary, especially for external IO and human-task waiting windows.
- Business calendars can pause or shape SLA computation so weekends and holidays are excluded where required.
- Signal subscriptions are durable and correlated by explicit business keys.
- User tasks can be resumed by explicit task completion or by signal/event-driven completion if the business flow requires it.

---

## 13. Human Task Model

Human tasks are a core engine capability. They are not treated as a thin notification wrapper. Each user task may define candidate roles, candidate groups, direct assignees, auto-assignment, claim/release/delegate rules, substitution policies, comments, attachments, form references, reminders, and multi-level escalation paths.

| Area | Supported Behavior |
|------|-------------------|
| Assignment | static assignee, candidate role/group, dynamic expression, or API-backed routing |
| Actions | claim, release, delegate, reassign, comment, attach, complete, approve, reject, send back |
| Timing | warning SLA, due SLA, escalation SLA, pause/resume conditions, business calendar awareness |
| Visibility | tasklist, mobile/web inbox, tenant-scoped dashboards, audit history |
| Escalation | notify higher role, reassign, create parallel review, raise operator alert, or trigger policy action |

---

## 14. Parallelism, Subprocesses, and Compensation

ParallelGateway, multi-instance execution, and sub-workflows together provide the main composition model for complex orchestration. Compensation is explicit and must be modeled, auditable, and replay-safe.

- ParallelGateway supports fork and merge with all, any, and k-of-n semantics.
- Multi-instance execution supports sequential or parallel processing over collections with merge strategies.
- Sub-workflows are version-aware child definitions and may block the parent or run asynchronously depending on wait settings.
- Compensation chains are declared on forward steps or subprocesses and must run in reverse dependency order on failure or cancellation.
- A workflow may reach a COMPENSATED terminal state once all required reverse actions complete successfully.

---

## 15. Connectors and Integration Model

Sphynx separates orchestration from integration implementation. The engine defines intent in DSL; the integration layer executes that intent through connectors and routes. Apache Camel is the preferred first-class integration backbone because it provides mature EIP support, route composition, and adapter extensibility.

| Connector Family | Typical Usage |
|-----------------|---------------|
| HTTP / REST | Service APIs, partner integrations, AI inference endpoints, webhook callbacks |
| gRPC | Low-latency internal platform services |
| Kafka / JMS / RabbitMQ | Event emit/consume, notification hooks, async triggers, streaming audit |
| Database / JDBC | Reads, writes, reconciliations, reporting hooks, status updates |
| Email / SMS / Slack / Webhooks | Notifications and escalations |
| S3 / File / Batch | Document flows, reports, attachments, imports/exports |
| Enterprise systems | SAP, Salesforce, Jira, ServiceNow, and domain-specific adapters via SDK |

---

## 16. Rule, Policy, and AI Decision Model

Rules, policy, and AI are all first-class execution patterns, but they serve different purposes and must stay separately governable.

- RuleTask resolves deterministic business decisions through expression logic, decision tables, or external rule services.
- PolicyCheck applies compliance or authorization decisions, typically through OPA or an equivalent policy engine.
- AIDecisionTask applies scorecards, ML models, or LLM-backed reasoning where allowed, and must capture model reference, threshold, explainability metadata, and audit details.
- AI-driven branching must always remain overrideable by policy and operator intervention.
- AI failure must degrade gracefully to manual review or deterministic fallback paths rather than blocking the engine core.

---

## 17. Persistence Model

Postgres is the primary durable store for definitions, instances, tokens, node results, timers, signals, tasks, and migration metadata. Redis is optional for fast locks, caches, short-lived indexes, and hot-path acceleration. Audit and long-term analytics may be streamed to Kafka, S3, warehouse storage, or Cassandra-style cold history stores.

| Table | Purpose |
|-------|---------|
| workflow_definition | Definition metadata, version, content, status, approval state |
| workflow_instance | One record per execution instance with high-level status and identity |
| execution_token | Active/parked tokens representing current graph positions |
| node_result | Durable result of each executed node attempt |
| timer_wheel | Scheduled timer/deadline/reminder entries |
| signal_subscription | Correlation registrations for waiting nodes |
| human_task | Manual task state, assignment, SLA, comments/attachment references |
| audit_log | Append-only execution facts with sequence and integrity metadata |
| migration_marker | Definition migration and compatibility markers |
| instance_snapshot | Optional replay-acceleration checkpoints |

### 17.1 Indexing Expectations

- **Instance-centric lookup:** instance id, tenant id, status, business key.
- **Execution-centric lookup:** node id, token state, queue, priority.
- **Time-centric lookup:** fire_at, due_at, escalation_at.
- **Correlation-centric lookup:** signal name plus correlation key.
- **Audit-centric lookup:** (instance_id, sequence), event type, definition version.

---

## 18. Audit and Deterministic Replay

Audit is append-only and immutable. Every meaningful lifecycle change emits an ordered event. Replay reconstructs workflow state by folding the audit stream into a replay context, optionally accelerated by snapshots. Replay serves debugging, evidence, migration validation, drift detection, and operational recovery.

| Event Type | Meaning |
|-----------|---------|
| INSTANCE_STARTED | Instance created with initial variables and version pin |
| TOKEN_ACTIVATED | A token became active on a node |
| NODE_ATTEMPTED | Execution attempt began |
| NODE_COMPLETED | Node finished successfully or with business outcome |
| NODE_RETRY_SCHEDULED | Retry was computed and persisted |
| TIMER_SCHEDULED / TIMER_FIRED | A timer was created or fired |
| SIGNAL_WAITING / SIGNAL_RECEIVED | Execution parked for and later received a correlated signal |
| TOKEN_JOINED | Parallel/join behavior merged tokens |
| INSTANCE_COMPLETED | Instance reached terminal state |
| DEFINITION_MIGRATED | Running execution moved to compatible definition version |

- **Replay invariants:** immutability, causality, integrity, version pinning, and idempotent side-effect control.
- **Replay modes:** state rebuild, shadow execution, divergence detection, and controlled node re-run.
- **Audit integrity** should support hash-chain or equivalent cryptographic verification for high-assurance environments.

---

## 19. API Surface

| API Group | Core Operations |
|-----------|----------------|
| Definitions | create, validate, get, list, publish, deprecate, compare, export, import |
| Deployment | deploy, promote, approve, rollback, activate, archive |
| Instances | start, get, search, cancel, suspend, resume, retry, migrate |
| Tasks | list inbox, claim, release, delegate, comment, attach, complete |
| Signals | send correlated signal, list subscriptions, inspect waiting instances |
| Administration | replay instance, re-run node, inspect queue backlog, trigger migration, list definitions |
| Operations / visibility | history, audit export, metrics, SLA breach views, health and readiness |

---

## 20. Security, Multi-Tenancy, and Compliance

Security is layered across API access, execution authorization, data protection, and operational governance. Multi-tenancy is enforced at definition, instance, connector, queue, and audit boundaries.

- Authentication supports enterprise SSO / OIDC / OAuth2.
- Authorization combines RBAC, ABAC, and policy-engine evaluation where required.
- Sensitive variables may be masked in logs, selectively encrypted at rest, and redacted from audit exports.
- Tenant scoping must prevent cross-tenant execution, connector leakage, queue leakage, and audit visibility leakage.
- Governance controls support model approval chains, promotion pipelines, change history, and policy simulation before release.
- Compliance templates should be available for SOX, GDPR, HIPAA, PCI-style deployments where relevant.

---

## 21. Observability and Operations

Sphynx must expose deep operational visibility for both platform teams and business operators.

| Capability | What It Should Cover |
|-----------|---------------------|
| Metrics | throughput, queue lag, retries, task latency, timer backlog, SLA breaches, instance volume, failure rate |
| Tracing | distributed traces across dispatch, task execution, external calls, and signal resumption |
| Logging | structured JSON logs with correlation key, tenant, instance id, node id, queue, and actor |
| Dashboards | runtime health, business outcome counts, human-task cycle time, escalation heat maps |
| Alerts | stuck workflow, repeated failure, timer backlog, queue pressure, SLA breach, tenant anomaly |
| Deploy safety | blue/green, canary, version pinning, shadow replay, health checks, rollback hooks |

---

## 22. Deployment Topologies

The engine is intended to run cloud-native, on-premise, or hybrid. A standard deployment includes API/orchestrator service, worker pods, human-task service, connector service, optional rule/AI services, Postgres, optional Redis, and audit sinks.

- **Single-region production baseline:** API/orchestrator + workers + Postgres + Redis + audit sink.
- **Regulated enterprise baseline:** add policy service, secrets manager, warehouse/S3 archive, HA Postgres, and stronger audit integrity controls.
- **SaaS baseline:** tenant-aware queues, tenant quotas, tenant dashboards, and safe connector sandboxing.
- **Air-gapped/on-prem profile:** retain native runtime, local Postgres, local object storage, and internal-only connectors.

---

## 23. Performance and Scaling Strategy

The finalized performance stance emphasizes predictable throughput, bounded resource usage, and clear tuning levers rather than opaque automation.

- Scale vertically first by increasing threads, queue capacities, and runtime pool sizing up to safe host thresholds.
- Scale horizontally when CPU, queue pressure, or SLA conditions cross configured thresholds.
- Prefer bounded queues and explicit backpressure over silent overload.
- Use priority and SLA boost to protect urgent work.
- Separate heavy integration nodes from lightweight routing nodes through queue and worker isolation.
- Archive cold history to lower-cost storage while retaining hot operational state in Postgres/Redis.

---

## 24. SPHUTA-Aligned Primary Use Cases

| Use-Case Family | Examples |
|----------------|----------|
| Finance approvals | invoice approval, payment release, reconciliation exception routing, AR/AP review |
| HR and employee workflows | onboarding, document collection, approval chains, compliance attestations |
| Compliance operations | policy checks, audit evidence gathering, exception handling, remediation flows |
| Billing and timesheets | submission, validation, manager/finance review, posting, notification |
| Customer/CRM flows | case routing, escalation, communication orchestration, SLA-driven follow-up |
| Cross-service orchestration | service choreography, notification fan-out, batch-to-event transition, document-driven processes |

---

## 25. Governance and Lifecycle Management

- Definitions follow Dev → QA → UAT → Prod style promotion with approval checkpoints.
- Each deployed workflow version remains immutable once active; edits create a new version.
- Safe migration requires compatibility checks, variable mapping validation, and replay-based confidence tests.
- Retirement or archival must preserve audit discoverability even when a version is no longer deployable.
- Marketplace/template distribution must not bypass tenant, policy, or approval gates.

---

## 26. Final Roadmap View

| Phase | Primary Deliverables |
|-------|---------------------|
| Phase 1 | Native runtime kernel, DSL validator, definition repository, basic executors, Postgres persistence, API baseline |
| Phase 2 | Timers, retries, signals, human-task service, core audit, queue isolation, connector registry |
| Phase 3 | Parallelism, sub-workflows, compensation, replay tooling, policy checks, visibility dashboards |
| Phase 4 | BPMN/DMN interoperability, visual modeler, migration tooling, richer connector packs, advanced HA |
| Phase 5 | Marketplace, SaaS tenant controls, optimization tooling, AI-assisted authoring and recommendations |

---

## 27. Non-Goals and Boundaries

- The engine should not be framed as a thin wrapper around an external workflow product.
- Kafka or any one broker should not be treated as the sole mandatory runtime dependency.
- AI should not be allowed to make opaque irreversible decisions without policy or human-governed fallback.
- The core orchestration model should not be coupled to one industry-specific domain model.
- Ultra-low-latency in-process logic should not be forced through distributed orchestration when a direct in-process path is more appropriate.

---

## 28. Final Recommendation

Proceed with Sphynx as a native, proprietary workflow runtime whose core strengths are deterministic orchestration, three-mode authoring (JSON DSL, BPMN, UI designer), human tasks, policy-aware execution, first-class audit, and pluggable integration. Use Camel as the integration backbone, OPA/decision services as governance layers, JSON/YAML as the canonical definition model, and Postgres-centered persistence with optional Redis acceleration.

Retain compatibility bridges and migration paths for BPMN/DMN — but keep the product identity and implementation baseline anchored in the native Sphynx runtime defined here.

---

## Appendix A. Canonical YAML Skeleton

The following YAML skeleton reflects the finalized engine direction and is intended as the canonical authoring reference shape.

```yaml
sphynxDslVersion: "1.1"
engine: sphynx

workflow:
  id: submit_timesheet
  name: Submit Timesheet
  version: 1
  description: Timesheet submission with approvals and posting
  owner: TMS
  tenant: shared
  tags: [tms, approval]
  timezone: America/Chicago
  expressionLang: jsonata

  concurrency:
    maxParallelNodes: 4
    workflowConcurrency: 500

  runtime:
    scaling:
      strategy: vertical-first
      vertical:
        cpuUtilizationThreshold: 0.75
        allowNodeOverrides: true
        threadPools:
          default:
            threads: 32
            queueCapacity: 1000
            rejectionPolicy: block
            contextPropagation: [trace, auth]
          ServiceTask:
            threads: 48
            queueCapacity: 1500
          UserTask:
            threads: 8
            queueCapacity: 200
      horizontal:
        enabled: true
        scaleOutThreshold: 0.90
        maxReplicas: 20
    rateLimits:
      workflowRps: 200
      burst: 100
    priorities:
      default: 5
      slaBoost: 2
    backpressure:
      strategy: buffer
      highWatermark: 0.85
      lowWatermark: 0.60

  defaults:
    retryPolicy:
      maxAttempts: 5
      backoff:
        type: exponential
        initial: 5s
        max: 5m
        jitter: full
    timeouts:
      workflowRun: P7D
      taskStartToClose: PT2M
      heartbeat: PT30S
    sla:
      deadline: P2D
      breachAction: escalate:manager

  variables:
    schema:
      type: object
      properties:
        employeeId: { type: string }
        period: { type: string }
        hours: { type: number }
      required: [employeeId, period]
    defaults:
      hours: 0

  security:
    auth: oidc
    classification: CONFIDENTIAL
    pii:
      mask: [employee.ssn]
      encryptAtRest: true
    rbac:
      roles:
        employee: [start, submit]
        manager: [approve, reject]
        finance: [reconcile]

  audit:
    export:
      format: S65B
      includeHeaders: true
    eventStream:
      channel: kafka
      topic: audit.sphynx.submit_timesheet
      retentionDays: 3650

  start: submit

  nodes:
    - id: submit
      type: UserTask
      assignment:
        candidateRoles: [employee]
      ui:
        formRef: timesheet-form-v1
      on:
        submitted: validate
        cancelled: end_cancelled

    - id: validate
      type: ServiceTask
      worker:
        type: http
        endpoint:
          url: https://hr/api/timesheets/validate
          method: POST
      retry:
        policyRef: default
      on:
        success: approval_gate
        error: manual_review

    - id: approval_gate
      type: ExclusiveGateway
      branches:
        - when: ${hours < 40}
          to: post_ledger
        - default: true
          to: manager_review

    - id: manager_review
      type: UserTask
      assignment:
        candidateRoles: [manager]
      sla:
        due: PT8H
      on:
        approved: post_ledger
        rejected: end_rejected

    - id: post_ledger
      type: SubWorkflow
      workflowRef: ledger_posting_v2
      waitForCompletion: true
      compensation:
        workflowRef: reverse_ledger_v1
      on:
        completed: notify
        failed: reconcile

    - id: reconcile
      type: UserTask
      assignment:
        candidateRoles: [finance]
      on:
        completed: notify

    - id: notify
      type: NotificationTask
      channel: email
      templateRef: timesheet-submitted
      on:
        sent: end_success

    - id: end_success
      type: EndEvent
      result:
        status: COMPLETED

    - id: end_rejected
      type: EndEvent
      result:
        status: REJECTED

    - id: end_cancelled
      type: EndEvent
      result:
        status: CANCELLED
```

---

## Appendix B. Node-to-Executor Mapping

| Node | Executor |
|------|----------|
| UserTask | UserTaskExecutor |
| ServiceTask / ConnectorTask | HttpExecutor, CamelConnectorExecutor, GrpcExecutor, DbExecutor, etc. |
| RuleTask / PolicyCheck | RuleExecutor, OPAExecutor |
| AIDecisionTask | AIExecutor |
| CodeTask / ScriptTask | CodeExecutor / ScriptExecutor |
| ParallelGateway | ParallelExecutor |
| ExclusiveGateway / Switch | RoutingExecutor |
| ForEach / MultiInstance | LoopExecutor |
| Timer / TimerEvent | TimerExecutor |
| SignalTask / SignalCatch | SignalWaitExecutor |
| NotificationTask | NotifyExecutor |
| SubProcess / SubWorkflow | SubprocessExecutor |
| Compensation | CompensationExecutor |
| EndEvent | EndExecutor |

---

## Appendix C. Source Reconciliation Notes

Primary uploaded materials reviewed for this consolidation included the native orchestrator spec, unified DSL specification, platform specifications, workflow-engine V2/V3 documents, roadmap variants, feature-summary draft, comparison document, advanced JSON DSL pattern documents, JSON schema/example files, and the archived workflow-definition zip contents.

- Native runtime and replay model came primarily from the native orchestrator and overall feature-set material.
- Definition shape and node taxonomy came primarily from the unified DSL specification and JSON DSL pattern documents.
- Scaling, runtime overrides, and vertical-first controls were reconciled with the extended JSON schema and advanced example definition.
- Platform-level capabilities, tool chain, compliance posture, and deployment expectations were reconciled from the platform specification material.
- Earlier hybrid documents were retained as exploration lineage and interoperability reference, but normalized here into one native engine baseline.

---

*End of finalized consolidated Sphynx Workflow Engine specification.*

> *SPHYNX — Intelligent, Compliant, and Scalable Enterprise Automation*
> *Part of the SPHUTA Platform | Pommala LLC*
