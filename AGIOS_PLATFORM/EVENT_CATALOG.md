# AGI-OS Canonical Event Catalog

> The canonical event envelope and v1 event catalog. This is a reference document — grep or link directly.

## Status

**Draft v0.1.** Envelope is frozen (see §2). V1 event names and groupings are frozen (see §3, mirrored in `PLATFORM_DESIGN §10`). **Payload schemas + idempotency formulas are still pending Pass 3** — the §3 table locks names so consumers can subscribe against stable topics while payloads firm up during Phase F1 implementation.

## 1. Purpose

Every cross-cutting platform concern — billing, audit, analytics, customer delivery, observability — consumes the same event stream. This document is the single source of truth for what an event looks like and which events exist.

Rule: **if you are about to emit an event shape that is not in this document, stop and open a PR to this document first**. An event shape becomes permanent the moment a consumer relies on it.

## 2. Canonical envelope (v1)

Every `DomainEvent` on the bus carries this envelope. The envelope is **immutable within a major version**.

| Field | Type | Required | Meaning |
|---|---|---|---|
| `event_id` | UUID | yes | Unique per emission. Consumers dedupe on this when idempotency key alone is insufficient. |
| `event_type` | string | yes | Fully qualified event name. Canvas-emitted events use `canvas.{id}.{event}`; platform events use `{service}.{event}`. |
| `schema_version` | string (semver) | yes | Version of the *payload* schema for this `event_type`. Envelope version is implicit in the bus transport. |
| `occurred_at` | RFC 3339 timestamp | yes | Wall clock of the **business transition**, not the publish call. |
| `project_id` | string(36) | yes | Tenant scope. Every event is project-scoped. |
| `source_service` | string | yes | Emitting microservice name (e.g. `task-management`, `integration-hub`, `canvas.tb`). |
| `idempotency_key` | string(255) | yes | Derived from business identity. Consumers use this to dedupe. Formula documented per event type. |
| `correlation_id` | string | no | Trace a logical task lifecycle across many events. Usually equals `task_id` or `batch_id`. |
| `causation_id` | UUID | no | The `event_id` of the event that caused this one. Used for lineage and replay. |
| `canvas_id` | string | conditional | Required when `source_service` starts with `canvas.`. Matches the canvas manifest ID. |
| `payload` | object | yes | Event-type-specific body, conforming to `schema_version`. |

### Rules

1. **At-least-once delivery.** Every consumer must be idempotent, keyed on `idempotency_key`.
2. **Ordering is per-aggregate, not global.** Events sharing `project_id` + an aggregate key (e.g. `task_id`) arrive in emission order. Across aggregates, no guarantee.
3. **Durability.** Every event is written to the audit log synchronously with publish. The audit log is the source of truth for "did this event happen."
4. **No enrichment in transit.** The bus passes events through unchanged. Consumers do their own joins.
5. **Schema evolution.** Payload schemas follow semver. Minor bumps add optional fields; major bumps are breaking and require deprecation notice.

## 3. Event catalog v1

**16 events in three groups.** Names and groupings are frozen. Payload schemas + idempotency formulas are Phase F1 deliverables (Pass 3 content).

### 3.1 Task lifecycle (8 events)

The customer contract — a billable task traversing the canonical state machine (`PLATFORM_DESIGN §11`). Applies to every canvas regardless of billing model.

| Event | Emitter | Stage | State transition | Idempotency key (preview) |
|---|---|---|---|---|
| `TaskCreated` | canvas | create | `→ Created` | `task_id` |
| `TaskSubmitted` | canvas | submit | `Claimed → Submitted` | `task_id:<attempt>` |
| `TaskValidated` | qc / prism | qc | `Submitted → Validated` | `task_id:<attempt>:validated` |
| `TaskReworked` | qc / prism | qc | `Validated → Reworked` | `task_id:<attempt>:reworked` |
| `TaskRejected` | qc / prism | qc | `Validated → Rejected` | `task_id:<attempt>:rejected` |
| `TaskAccepted` | qc / prism | qc | `Validated → Accepted` | `task_id:<attempt>:accepted` |
| `TaskDelivered` | canvas | deliver | `Accepted → Delivered` | `task_id:delivered` |
| `TaskDeliveryAcked` | integration-hub | deliver | `Delivered → DeliveryAcked` | `task_id:delivery_acked` |

### 3.2 Unit lifecycle (5 events, pay-per-task-specific)

The worker claim/release cycle in the pool. Billing consumers read these to meter pay-per-task. Canvases on other billing models still emit them.

| Event | Emitter | Meaning | Idempotency key (preview) |
|---|---|---|---|
| `UnitPosted` | user-pool | Unit available for claim | `unit_id:posted` |
| `UnitClaimed` | user-pool | Worker holds a lease | `claim_id` |
| `UnitExpired` | user-pool | Lease TTL elapsed | `claim_id:expired` |
| `UnitReleased` | user-pool | Worker voluntarily released | `claim_id:released` |
| `UnitCompleted` | user-pool | Lease closed via submission | `claim_id:completed` |

### 3.3 Canvas lifecycle (3 events)

IH registration changes. Consumed by the worker shell's slug-resolution cache for invalidation (`COMPONENT_ARCHITECTURE §4.3.4`). Not billable-flow.

| Event | Emitter | Meaning | Idempotency key (preview) |
|---|---|---|---|
| `CanvasRegistered` | integration-hub | New canvas record exists | `canvas_id:registered` |
| `CanvasPromoted` | integration-hub | Canvas moved `staging → prod` | `canvas_id:promoted:<revision>` |
| `CanvasDeprecated` | integration-hub | Canvas marked for sunset | `canvas_id:deprecated` |

### 3.4 Deferred from v1

The following event families are acknowledged but deferred to a later minor version. They are not blocking the OpenClaw canary.

| Family | Events (tentative) | Ships with |
|---|---|---|
| Canvas operational | `CanvasArchived`, `CanvasCredentialRotated`, `CanvasScopeChanged` | IH refactor Track A |
| IH Outbound operational | `OutboundDeliveryAttempted`, `WebhookDeliveryFailed` | IH Outbound (Track A5) |
| Gateway audit | `CapabilityInvoked`, `RateLimitExceeded` | Gateway discipline (Track D) |
| HITL | TBD | HITL Zone B capability |
| Batching | TBD | Batching Zone B capability |

## 4. Payload schema conventions

> Pass 3 content. Specifies JSON Schema conventions, required vs optional fields, monetary value representation, worker/customer ID formats.

## 5. Idempotency key formulas

> Pass 3 content. Per-event formula for deriving the `idempotency_key` from business identity.

## 6. Current-state grounding

Existing event types in `services/shared/shared/events/types.py`:

- `ProjectCreated`, `ProjectApproved`, `ProjectStageAdvanced`
- `TeamAllocated`, `TalentOnboarded`
- `TaskProjectCreated`
- `QAReviewCompleted`, `SignOffApproved`
- `WebhookReceived`, `ConnectorInvoked`

**Gap.** These cover project lifecycle and integration mechanics but do not cover the canonical task/unit/canvas lifecycles (`§3.1` – `§3.3`). The v1 catalog adds all 16 events, plus upgrades the envelope to carry `project_id`, `idempotency_key`, `schema_version`, `correlation_id`, `causation_id`, and `canvas_id` — none of which today's `DomainEvent` base class has. See `KICKOFF.md §5 F1` for the implementation plan.
