# AGI-OS Platform Design

> Design documents for AGI-OS as a platform: the shell, the Task Execution Canvases it hosts, the services it provides, and the contracts between them.

## What this is

AGI-OS is the **platform** that hosts multiple **Task Execution Canvases** (TB, GDPVal, OpenClaw, and ~20 more). This directory defines what the platform provides, what canvases own, where the contracts live, and how the pieces fit.

Terminology used throughout:

- **Task Execution Canvas (TEC)** — formal name. "Canvas" in prose.
- **`ExecutionSurface`** — technical/code term (see `services/task-management/src/task_management/models/execution_surface.py`).
- The three surfaces: **AGI-OS Admin Shell**, **Worker Shell** (`canvas-agi-os.turing.com`), and **Canvas** (rendered inside the worker shell).

## Documents in reading order

| # | Doc | What it covers |
|---|---|---|
| 1 | [PLATFORM_DESIGN.md](./PLATFORM_DESIGN.md) | **Start here.** Vision, operating principles, the three zones (A/B/C), the three-surface model, high-level architecture with diagrams, committed tech decisions, concurrency envelope + SLOs (§7.8), rollout, governance. |
| 2 | [COMPONENT_ARCHITECTURE.md](./COMPONENT_ARCHITECTURE.md) | Per-component deep-dive. Purpose, contract, events, SDK, storage, tech choices, diagrams. Includes §0 Shell Offerings landing; §4.8 Gateway; §4.9 Notification Edge; §5.1 User Pool with hot-path discipline. |
| 3 | [CANVAS_SDK.md](./CANVAS_SDK.md) | How canvases plug into AGI-OS. Worker-shell architecture, bridge protocol, runtime capability discovery (§4.5), two-credential auth (§5), IH-owned registration (§3), onboarding playbook (§6). |
| 4 | [EVENT_CATALOG.md](./EVENT_CATALOG.md) | Canonical event envelope + v1 event catalog. Grep target. |
| 5 | [IH_GAP_ANALYSIS.md](./IH_GAP_ANALYSIS.md) | Current-state vs target-state for Integration Hub, grounded in the AGI-OS codebase. Scoped to IH only. |
| 6 | [KICKOFF.md](./KICKOFF.md) | **Implementation plan.** Dependency graph, phase ordering (sequential vs parallel), per-item BOE sizing (S/M/L/XL). No calendar — EM owns that. Read after the design is approved. |

## Reading by audience

| Who | Read | Can skip |
|---|---|---|
| Leadership | `PLATFORM_DESIGN §2` (TL;DR) + `§7.8` (scale) | Everything else |
| Platform engineers | `PLATFORM_DESIGN` + `COMPONENT_ARCHITECTURE` | — |
| Canvas engineers (TB, GDPVal, OpenClaw, …) | `CANVAS_SDK` first, then `PLATFORM_DESIGN §4–§6` | `§14–§17` |
| IH team | `IH_GAP_ANALYSIS` + `COMPONENT_ARCHITECTURE §4.3`, `§5.8` | — |
| SRE / infra | `PLATFORM_DESIGN §7.6` (tech decisions) + `COMPONENT_ARCHITECTURE` topology diagrams | — |
| EM / staff eng scoping implementation | `KICKOFF.md` (dependencies + BOE) + `PLATFORM_DESIGN §14` (rollout) | — |

## Status dashboard

| Doc | Version | Status |
|---|---|---|
| `PLATFORM_DESIGN.md` | 0.7 | Through §7 complete. §7.6 splits event plumbing into three layers; §7.8 pins the concurrency envelope (1M registered / 200K peak concurrent) with per-component SLOs against peak burst. §4.2 puts IH as the integration plane; §15.2 is now "Canvas registration (via IH)". Zero YAML-manifest references. |
| `COMPONENT_ARCHITECTURE.md` | 0.4 | §0 Shell Offerings landing. §4.3 reframed as Integration Plane (three shapes: customer/canvas/platform-provider); §4.3.4 adds slug-cache + bus-invalidation for hot-path reads. New Zone A: §4.8 Gateway (hot-path discipline) + §4.9 Notification Edge (SSE tier, 25K sockets/pod). §5.1 User Pool adds Scale / hot-path discipline with committed SLOs. 7 of 21 components filled. |
| `CANVAS_SDK.md` | 0.5 | Canvas registration owned by IH (§3 is a pointer, not a schema). §4.5 MCP-shaped runtime capability discovery. §5 two credential types (backend client creds + per-worker session JWT). `canvas_id` platform-assigned. §6.2 onboarding is an IH form. |
| `EVENT_CATALOG.md` | 0.0 | Envelope defined; v1 catalog formalization pending (backed today by `PLATFORM_DESIGN §10`). |
| `IH_GAP_ANALYSIS.md` | 0.0 | Outline + current-state table; target & migration plan pending. |
| `KICKOFF.md` | 0.2 | Implementation kickoff — **dependencies + BOE sizing only, no timelines**. Phase F serial (F1 envelope → F2 outbox → F3 Redis Streams → F4 Audit Log, ~31–53 eng-days). Phase P1 = 4 parallel tracks (IH refactor, worker shell, User Pool, gateway discipline). Phase 2 = OpenClaw canary. Total ~189–301 eng-days; EM computes calendar from team size. |

## Open decisions

Gate items that remain before fill-in completes. Worker-shell architecture closed 2026-04-21.

1. ~~**Worker-shell architecture.**~~ **Closed 2026-04-21 — iframe + bridge SDK.** Rationale in `CANVAS_SDK.md §2`; committed in `PLATFORM_DESIGN.md §7.6`. ADR to be written as permanent record.
2. **Event bus sharding strategy.** Per-project stream vs global with routing key. Affects `COMPONENT_ARCHITECTURE §4.1`. ADR-003 during `KICKOFF F3`.
3. **`ProjectIntegration` vs `ConnectorConfigModel` unification.** Real duplication bug. Resolution proposed in `IH_GAP_ANALYSIS`; implementation is `KICKOFF` Track A4.
4. **Prism naming.** Rename of `quality-control`, or new layered service composed of it? Affects `COMPONENT_ARCHITECTURE §5.3`.

## Related documents (siblings)

| Doc | Purpose |
|---|---|
| `../TB_AGI_OS/PLATFORM_DESIGN_AGIOS.md` | TB canvas in AGIOS-federated mode |
| `../TB_AGI_OS/PLATFORM_DESIGN.md` | TB standalone reference |
| `../GDPVAL/PLATFORM_DESIGN.md` | GDPVal talent evaluation canvas |
| `../Notification Service/PLATFORM_DESIGN.md` | Platform notification service |

## Change log

- **2026-04-21** — **`KICKOFF.md` v0.2 — rewrote without timelines.** Reframed around dependency ordering (sequential vs parallel) and BOE sizing (S/M/L/XL, vibe-coding assumed). Phase F strictly serial (F1 → F2 → F3 → F4, ~31–53 eng-days). Phase P1 is four fully-parallel tracks (A: IH refactor, B: worker shell, C: User Pool, D: gateway discipline) with per-item size + dependency tables. Phase 2 = OpenClaw canary. Dropped all week assignments — EM owns calendar math from total BOE (~189–301 eng-days) + team size + focus factor.
- **2026-04-21** — **Concurrency envelope pinned at 1M registered / 200K peak concurrent.** Every hot-path component now carries a numeric SLO against peak burst. Notification socket-termination split into Zone A `§4.9 Notification Edge` (25K sockets/pod, FCM/APNS fallback). Gateway `§4.8` has explicit hot-path discipline: stateless, token-native auth, async audit via outbox, per-worker Redis rate limit, circuit-broken downstream pools. User Pool `§5.1` upgraded to hybrid Postgres-for-claim + Redis-for-heartbeat with committed SLOs (claim p99 100 ms, heartbeat p99 10 ms at 50K QPS). IH slug resolution `§4.3.4` gets two-tier cache invalidated via `Canvas*` events. New `PLATFORM_DESIGN.md §7.8` ties it together.
- **2026-04-21** — **Canvas registration moved to Integration Hub.** The `canvas-manifest.yaml` design was rejected as identity-and-state mishoused as a deployment artifact. IH is now the single integration plane for three shapes: customer, canvas, platform-provider integrations. All share the same data model, admin UI, Secrets Service integration, audit trail, RBAC. `canvas_id` is platform-assigned (`cvs_<nonce>`), immutable. Capability adoption is runtime discovery (MCP-shaped: `capabilities.list` / `describe` / `use`). Authorization via session JWT `scope` claim, derived from IH grants.
- **2026-04-21** — Event plumbing re-architected into three explicit layers: Layer 1 outbox (atomic publish at source), Layer 2 hot bus (Redis Streams for online fan-out), Layer 3 durable log (Audit Log service for long-retention history). `§4.1` topology redrawn with swap triggers for Redis → Kafka; `§4.5` Audit Log upgraded from stub to full component spec.
- **2026-04-21** — Worker-shell architecture decided: iframe + bridge SDK. `CANVAS_SDK.md §2` (topology), §4 (bridge protocol), §5 (auth handoff), §6 (onboarding playbook) filled. Worker shell domain: `canvas-agi-os.turing.com`. Canvas mount path: `/<slug>/*`.
- **2026-04-21** — `CANVAS_SDK.md §2.5` added: explicit v1 scope & non-goals. v1 ships flat `bridge.send` / `bridge.on` API, typed envelope, auth handoff. Explicitly deferred to v2: scripting-object wrapper (SSF-style `getObject`), interactive events with veto, `@agi-os/canvas-cli`, canvas-publishes-to-shell inversion, double-iframe HITL. Only forward-compat invariant: message names stay `<noun>.<verb>`.
- **2026-04-21** — Multi-doc split. Content migrated from a single `PLATFORM_DESIGN.md` into the structure above. Terminology renamed seller → canvas. Mermaid diagrams added. Tech decisions committed in `PLATFORM_DESIGN §7.6`.
