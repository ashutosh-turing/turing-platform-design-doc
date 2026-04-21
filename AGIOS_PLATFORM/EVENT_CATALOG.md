# AGI-OS Canonical Event Catalog

> The canonical event envelope and v1 event catalog. This is a reference document — grep or link directly.

## Status

**Draft v0.0.** Envelope is frozen (see §2). V1 event catalog is **pending Pass 3** of the platform design work.

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

> **Pending Pass 3.** The 12 canonical events for the pay-per-task lifecycle will be specified here, with payload schema, idempotency formula, emitter, and example.
>
> Proposed list (to be formalized):
>
> | Event | Emitter | Stage |
> |---|---|---|
> | `TaskCreated` | canvas | create |
> | `UnitPosted` | user-pool | pool |
> | `UnitClaimed` | user-pool | pool |
> | `UnitReleased` | user-pool | pool |
> | `UnitExpired` | user-pool | pool |
> | `TaskSubmitted` | canvas | submit |
> | `TaskValidated` | qc | qc |
> | `TaskRejected` | qc | qc |
> | `TaskReworked` | qc | qc |
> | `TaskAccepted` | qc | qc |
> | `TaskDelivered` | canvas | deliver |
> | `OutboundDeliveryAttempted` | integration-hub | deliver |
> | `TaskDeliveryAcked` | integration-hub | deliver |
> | `WebhookDeliveryFailed` | integration-hub | deliver |

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

**Gap.** These cover project lifecycle and integration mechanics but do not yet cover the pay-per-task lifecycle (create → claim → submit → QC → deliver → ack). The v1 catalog in §3 adds those, plus upgrades the envelope to carry `project_id`, `idempotency_key`, `schema_version`, `correlation_id`, `causation_id`, and `canvas_id` which today's `DomainEvent` base class does not have.
