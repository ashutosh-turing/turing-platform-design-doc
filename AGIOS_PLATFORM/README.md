# AGI-OS Platform Design

> Design documents for AGI-OS as a platform: the shell, the Task Execution Canvases it hosts, the services it provides, and the contracts between them.

## What this is

AGI-OS is the **platform** that hosts multiple **Task Execution Canvases** (TB, GDPVal, OpenClaw, and ~20 more). This directory defines what the platform provides, what canvases own, where the contracts live, and how the pieces fit.

Terminology used throughout:

- **Task Execution Canvas (TEC)** — formal name. "Canvas" in prose.
- **`ExecutionSurface`** — technical/code term (see `services/task-management/src/task_management/models/execution_surface.py`).
- The three surfaces: **AGI-OS Admin Shell**, **Worker Shell** (`tasks.turing.com/ppt`), and **Canvas** (rendered inside the worker shell).

## Documents in reading order

| # | Doc | What it covers |
|---|---|---|
| 1 | [PLATFORM_DESIGN.md](./PLATFORM_DESIGN.md) | **Start here.** Vision, operating principles, the three zones (A/B/C), the three-surface model, high-level architecture with diagrams, committed tech decisions, rollout, governance. |
| 2 | [COMPONENT_ARCHITECTURE.md](./COMPONENT_ARCHITECTURE.md) | Per-component deep-dive. Purpose, contract, events, SDK, storage, tech choices, diagrams. Reference doc — jump to the component you need. |
| 3 | [CANVAS_SDK.md](./CANVAS_SDK.md) | How canvases plug into AGI-OS. Worker-shell architecture (open decision), bridge protocol, auth handoff, onboarding playbook. |
| 4 | [EVENT_CATALOG.md](./EVENT_CATALOG.md) | Canonical event envelope + v1 event catalog. Grep target. |
| 5 | [IH_GAP_ANALYSIS.md](./IH_GAP_ANALYSIS.md) | Current-state vs target-state for Integration Hub, grounded in the AGI-OS codebase. Scoped to IH only by explicit choice. |

## Reading by audience

| Who | Read | Can skip |
|---|---|---|
| Leadership | `PLATFORM_DESIGN §2` (TL;DR) | Everything else |
| Platform engineers | `PLATFORM_DESIGN` + `COMPONENT_ARCHITECTURE` | — |
| Canvas engineers (TB, GDPVal, OpenClaw, …) | `CANVAS_SDK` first, then `PLATFORM_DESIGN §4–§6` | `§14–§17` |
| IH team | `IH_GAP_ANALYSIS` + `COMPONENT_ARCHITECTURE §4.3`, `§5.8` | — |
| SRE / infra | `PLATFORM_DESIGN §7.6` (tech decisions) + `COMPONENT_ARCHITECTURE` topology diagrams | — |

## Status dashboard

| Doc | Version | Status |
|---|---|---|
| `PLATFORM_DESIGN.md` | 0.5 | Through §7 complete; §7.6 split event plumbing into three layers (outbox / hot bus / durable log); §8–§18 filled or pointing at sibling docs |
| `COMPONENT_ARCHITECTURE.md` | 0.2 | 4 of 19 components filled (Event Bus with three-layer topology + swap triggers, IH Outbound, Audit Log, User Pool); 15 stubs with current-state notes |
| `CANVAS_SDK.md` | 0.3 | Shell architecture **decided — iframe + bridge SDK**. §3 manifest, §4 bridge protocol, §5 auth handoff, §6 onboarding playbook all filled. **§2.8 adds explicit v1 scope & non-goals** — flat message API only; scripting-objects / CLI / veto-events deferred to v2. |
| `EVENT_CATALOG.md` | 0.0 | Envelope defined; v1 catalog pending Pass 3 |
| `IH_GAP_ANALYSIS.md` | 0.0 | Outline + current-state table; target & migration plan pending Pass 4 |

## Open decisions

Three remaining items gate further fill-in. The fourth (worker-shell architecture) closed 2026-04-21.

1. ~~**Worker-shell architecture.**~~ **Closed 2026-04-21 — iframe + bridge SDK.** Rationale in `CANVAS_SDK.md §2`; committed in `PLATFORM_DESIGN.md §7.6`. ADR to be written as the permanent record.
2. **Event bus transport confirmation**. Redis Streams is the recommended pick; NATS and Kafka are real alternatives. Affects `COMPONENT_ARCHITECTURE §4.1`.
3. **`ProjectIntegration` vs `ConnectorConfigModel` unification**. Real duplication bug in the current codebase. Proposed resolution in `IH_GAP_ANALYSIS`.
4. **Prism naming**. Is "Prism" a rename of the existing `quality-control` service, or a new layered service composed of it? Affects `COMPONENT_ARCHITECTURE §5.3`.

## Related documents (siblings)

| Doc | Purpose |
|---|---|
| `../TB_AGI_OS/PLATFORM_DESIGN_AGIOS.md` | TB canvas in AGIOS-federated mode |
| `../TB_AGI_OS/PLATFORM_DESIGN.md` | TB standalone reference |
| `../GDPVAL/PLATFORM_DESIGN.md` | GDPVal talent evaluation canvas |
| `../Notification Service/PLATFORM_DESIGN.md` | Platform notification service |

## Change log

- **2026-04-21** — Multi-doc split. Content migrated from a single `PLATFORM_DESIGN.md` into the structure above. Terminology renamed seller → canvas. Mermaid diagrams added to architecture sections. Tech decisions committed in `PLATFORM_DESIGN §7.6`.
- **2026-04-21** — Worker-shell architecture decided: iframe + bridge SDK. `CANVAS_SDK.md §3` (manifest), §4 (bridge protocol), §5 (auth handoff), §6 (onboarding playbook) filled.
- **2026-04-21** — Event plumbing re-architected into three explicit layers: Layer 1 outbox (atomic publish at source), Layer 2 hot bus (Redis Streams for online fan-out), Layer 3 durable log (Audit Log service for long-retention history). `PLATFORM_DESIGN §7.6` split; `COMPONENT_ARCHITECTURE §4.1` topology redrawn with swap triggers for Redis→Kafka; `§4.5` Audit Log upgraded from stub to full component spec.
- **2026-04-21** — `CANVAS_SDK.md §2.8` added: explicit v1 scope & non-goals. v1 ships flat `bridge.send` / `bridge.on` API, typed envelope, auth handoff, hand-written manifest. Explicitly deferred to v2: scripting-object wrapper (SSF-style `getObject`), interactive events with veto, `@agi-os/canvas-cli`, canvas-publishes-to-shell inversion, double-iframe HITL. Only forward-compat invariant: message names stay `<noun>.<verb>`. Build envelope: ~3 engineer-weeks.
