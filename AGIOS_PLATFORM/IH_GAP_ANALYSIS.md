# Integration Hub — Current State vs Target State

> Scoped to Integration Hub only, by explicit choice. Every other platform component in `COMPONENT_ARCHITECTURE.md` is target-state only; this document is the one exception where current code is grounded file-by-file.

## Status

**Draft v0.0.** Current-state section is grounded. Target-state and migration plan will be expanded in Pass 4.

## 1. Scope

This document answers three questions:

1. What does Integration Hub have today (in `services/integration-hub/`)?
2. What does `PLATFORM_DESIGN.md` require it to become?
3. What is the migration path, in rough order?

No other component is analyzed at this level of detail. See `COMPONENT_ARCHITECTURE.md` for target-state descriptions of other components.

## 2. Current state (grounded)

### 2.1 What exists


| Area                         | Files                                                        | Status                                                                                                                                                           |
| ---------------------------- | ------------------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Connector interface**      | `src/integration_hub/connectors/base.py`                     | `BaseIntegrationConnector` ABC with `connect`, `disconnect`, `health_check`, `push_data`, `pull_data`, `validate_webhook_signature` (default HMAC-SHA256).       |
| **Registered connectors**    | `connectors/{google_sheets,bigquery,jibble,gcs,tms_sync}.py` | 5 connectors registered at startup via `IntegrationHubFacade.__init__`.                                                                                          |
| **Registry**                 | `registry/connector_registry.py`                             | In-process registry, keyed by `connector_type`.                                                                                                                  |
| **Per-project config**       | `models/connector_config.py`                                 | `ConnectorConfigModel(project_id, connector_type, configuration, webhook_secret, is_active)` with `UNIQUE(project_id, connector_type)`.                          |
| **Inbound webhook endpoint** | `api/webhooks.py`                                            | `POST /webhooks/{connector_type}` + `POST /webhooks/{connector_type}/bulk`. HMAC signature verify, 5-minute timestamp replay protection, idempotent persistence. |
| **Webhook event log**        | `models/webhook_event.py`                                    | `WebhookEventModel(idempotency_key UNIQUE, connector_type, event_type, status, raw_payload, received_at, processed_at)`.                                         |
| **Idempotency**              | `services/webhook_service.py:22-76`                          | Optimistic insert with `IntegrityError` fallback for concurrent duplicates. Race-safe.                                                                           |
| **Facade**                   | `service.py`                                                 | `IntegrationHubFacade` wires registry + services for API layer.                                                                                                  |
| **Admin views**              | `admin/views.py`, `admin/setup.py`                           | SQLAdmin-based views for config and events.                                                                                                                      |


### 2.2 What is stubbed or missing


| Area                                                       | Location                             | Current status                                                                                                                                          |
| ---------------------------------------------------------- | ------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Phase 2 inbound processing**                             | `services/webhook_service.py:78-100` | `process_event` is an explicit stub. Comments describe the intended Phase 2 flow (`received → processing → processed/failed`) but no dispatcher exists. |
| **Project-scoped webhook config lookup**                   | `api/webhooks.py:54-62`              | `_verify_signature` finds config by `connector_type` only. TODO comment: "scope by project_id when multi-tenant webhook routing is needed."             |
| **Outbound delivery — dispatcher**                         | -                                    | **Does not exist.** No subscription to `TaskDelivered`; no dispatcher; no outbound path at all.                                                         |
| **Outbound delivery — endpoint model**                     | -                                    | **Does not exist.** No `OutboundEndpointModel` or equivalent.                                                                                           |
| **Outbound delivery — attempts log**                       | -                                    | **Does not exist.**                                                                                                                                     |
| **Outbound delivery — DLQ**                                | -                                    | **Does not exist.**                                                                                                                                     |
| **Outbound delivery — ack tracking**                       | -                                    | **Does not exist.** No `TaskDeliveryAcked` emission.                                                                                                    |
| **Agent-panel connector**                                  | -                                    | Not a first-class connector type.                                                                                                                       |
| **Event emission (inbound → bus)**                         | -                                    | `WebhookReceived` event exists in `shared/events/types.py` but no IH code publishes it.                                                                 |
| **KMS-wrapped secrets**                                    | `models/connector_config.py:30-34`   | `webhook_secret` stored plaintext in Postgres.                                                                                                          |
| **Connector versioning**                                   | -                                    | No version field on `ConnectorConfigModel` or the registry.                                                                                             |
| **Outbound operator UI (DLQ replay, endpoint management)** | -                                    | Admin views cover inbound only.                                                                                                                         |


### 2.3 The `ProjectIntegration` vs `ConnectorConfigModel` duplication

A real architectural bug, not a polishing item. Two models in two services describe the same concept:


| Field              | `project-management` `ProjectIntegration` | `integration-hub` `ConnectorConfigModel` |
| ------------------ | ----------------------------------------- | ---------------------------------------- |
| Table              | `dos_pm_project_integrations`             | `dos_ih_connector_configs`               |
| Key                | `UNIQUE(project_id, integration_type)`    | `UNIQUE(project_id, connector_type)`     |
| Purpose field      | `integration_type` (enum)                 | `connector_type` (string)                |
| Config field       | `config` (JSON, no secrets)               | `configuration` (JSON)                   |
| External reference | `external_ref_id`                         | —                                        |
| Secrets            | —                                         | `webhook_secret`                         |
| Active flag        | `is_active`                               | `is_active`                              |


Neither is a superset of the other. The project-creation wizard writes to PM; IH reads from its own table. They are not kept in sync.

Proposed resolution in §4.

## 3. Target state

See:

- `PLATFORM_DESIGN.md §6.1` (Zone A: IH outbound is non-negotiable contract).
- `PLATFORM_DESIGN.md §7` (architecture).
- `COMPONENT_ARCHITECTURE.md §4.3` (IH outbound — full contract).
- `COMPONENT_ARCHITECTURE.md §5.8` (IH inbound — target for Phase 2).

Summary: IH must cover both directions with uniform signing, retry, DLQ, idempotency, ack tracking, and event emission. Every outbound customer delivery goes through the dispatcher; no canvas delivers directly.

## 4. Gap analysis


| #   | Gap                                                                                                  | Blast radius                                         | Effort | Phase |
| --- | ---------------------------------------------------------------------------------------------------- | ---------------------------------------------------- | ------ | ----- |
| 1   | `TaskDelivered` event does not exist in `shared/events/types.py`                                     | blocker for outbound                                 | XS     | P0    |
| 2   | Envelope missing `project_id`, `idempotency_key`, `schema_version`, `correlation_id`, `causation_id` | every consumer affected                              | S      | P0    |
| 3   | No `OutboundEndpointModel`, `OutboundDeliveryAttemptModel`, `OutboundDeliveryDLQModel`               | outbound blocked                                     | M      | P0    |
| 4   | No outbound dispatcher service subscribing to `TaskDelivered`                                        | outbound blocked                                     | L      | P0    |
| 5   | No outbound HMAC signing or customer-facing idempotency header                                       | customer trust blocker                               | S      | P0    |
| 6   | No DLQ tables or replay UI                                                                           | operator pain                                        | M      | P0    |
| 7   | `ProjectIntegration` / `ConnectorConfigModel` duplication                                            | correctness bug; PM-UI writes don't reach IH runtime | M      | P0    |
| 8   | `process_event` stub — inbound events do not publish to bus                                          | inbound analytics + canvas callbacks blocked         | S      | P1    |
| 9   | `webhook_secret` stored plaintext                                                                    | security concern                                     | M      | P1    |
| 10  | Webhook config lookup by `connector_type` only, not `(project_id, connector_type)`                   | multi-tenant routing blocker                         | S      | P1    |
| 11  | No first-class `agent_panel` connector                                                               | every canvas reinvents qc-agents callbacks           | M      | P1    |
| 12  | No connector versioning in manifest / registry                                                       | upgrade-in-place hazards                             | M      | P2    |


## 5. Migration plan (proposed)

> Pass 4 detail. One-line sketch per phase:

- **P0 (6 weeks).** Add `TaskDelivered` + upgrade envelope (#1, #2). Build outbound endpoint + attempt + DLQ models (#3). Ship outbound dispatcher with signing + retry (#4, #5, #6). Collapse `ProjectIntegration` into `ConnectorConfigModel` via a one-shot migration; PM reads via IH SDK thereafter (#7).
- **P1 (4 weeks).** Complete Phase 2 inbound dispatch — `process_event` routes to `ConnectorEventHandler` and publishes canonical events (#8). Move `webhook_secret` to KMS-wrapped references (#9). Scope webhook config lookups by project (#10). Add `agent_panel` as a first-class connector type (#11).
- **P2 (2 weeks).** Connector versioning in manifest + registry (#12). Operator console polish.

Detailed ticket breakdown lives in the engineering project plan, not here.