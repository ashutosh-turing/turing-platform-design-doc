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
| `PLATFORM_DESIGN.md` | 0.7 | Through §7 complete. §7.6 splits event plumbing into three layers; §7.8 pins the concurrency envelope (1M registered / 200K peak concurrent) with per-component SLOs against peak burst. §4.2 rewritten to put IH as the integration plane; §15.2 renamed from "Manifest validation" to "Canvas registration (via IH)". Zero references to YAML manifests remain. |
| `COMPONENT_ARCHITECTURE.md` | 0.4 | New §0 Shell Offerings landing. §4.3 reframed as Integration Plane with three shapes (customer/canvas/platform-provider); §4.3.4 adds canvas slug-resolution cache + bus-invalidation for hot-path reads. Two new Zone A components: §4.8 Gateway / API Edge (hot-path discipline — stateless, token-native auth, async audit, per-worker rate limit) and §4.9 Notification Edge (SSE socket-termination tier, 25K sockets/pod, FCM/APNS fallback). §5.1 User Pool adds Scale / hot-path discipline subsection with committed SLOs (claim p99 100 ms, heartbeat p99 10 ms at 50K QPS). 7 of 21 components filled. |
| `CANVAS_SDK.md` | 0.5 | Canvas registration moved to Integration Hub — §3 is now a ~100-line pointer, not a YAML schema. §4.5 adds MCP-shaped runtime capability discovery (`capabilities.list` / `describe` / `use`). §5 split into two credential types (backend client creds + per-worker session JWT); `canvas_id` is platform-assigned, never user-supplied. §6.2 rewritten: onboarding Phase 1 is an IH form, not a `canvas-registry` PR. |
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

- **2026-04-21** — **Concurrency envelope pinned at 1M registered / 200K peak concurrent.** Every hot-path component now carries a numeric SLO against peak burst (not mean). Notification socket-termination split into its own Zone A tier (`COMPONENT_ARCHITECTURE.md §4.9` Notification Edge) distinct from the Zone B notification capability — 25K sockets per pod, bus → client p99 2 s, FCM/APNS fallback for idle workers. Gateway (`§4.8`) now has explicit hot-path discipline: strictly stateless, token-native auth, async audit writes via outbox, per-worker Redis rate limit, circuit-broken downstream pooling. User Pool reference implementation (`§5.1`) upgraded to hybrid Postgres-for-claim + Redis-for-heartbeat with committed SLOs (claim p99 100 ms / heartbeat p99 10 ms at 50K QPS) so Level-0 canvases scale with the platform rather than being forced to Level 2. IH slug resolution (`§4.3.4`) gets a two-tier cache (pod LRU + Redis) invalidated via `Canvas*` events on the bus. New `PLATFORM_DESIGN.md §7.8` ties it all together with the design-load table and per-component SLO matrix. Touches: `COMPONENT_ARCHITECTURE.md §4.3.4`, `§4.8` (new), `§4.9` (new), `§5.1`; `PLATFORM_DESIGN.md §7.8` (new).
- **2026-04-21** — **Canvas registration moved to Integration Hub.** The `canvas-manifest.yaml` design was rejected as an identity-and-state concern mishoused as a deployment artifact (no RBAC, no crypto identity, credentials cannot live there, silent drift from reality, no lifecycle states, no queryability). IH is now the single integration plane for three shapes: **customer integrations**, **canvas integrations**, **platform-provider integrations**. All three share the same data model, admin UI, Secrets Service integration, audit trail, and RBAC. `canvas_id` is platform-assigned (`cvs_<nonce>`), immutable, never user-supplied. Capability adoption is **runtime discovery** (MCP-shaped: `capabilities.list` / `describe` / `use`), not a static declaration. Authorization comes from the session JWT's `scope` claim, which reflects IH grants. Touches: `CANVAS_SDK.md §3` (replaced), `§4.5` (new capability discovery), `§5` (two-credential auth), `§6.2` (new onboarding); `COMPONENT_ARCHITECTURE.md §0` (new Shell Offerings landing), `§4.3` (reframed as Integration Plane + `§4.3.4` canvas + `§4.3.5` platform-provider); `PLATFORM_DESIGN.md §4.2`, `§4.4`, `§4.5`, `§7.1`, `§15.2`.
- **2026-04-21** — Multi-doc split. Content migrated from a single `PLATFORM_DESIGN.md` into the structure above. Terminology renamed seller → canvas. Mermaid diagrams added to architecture sections. Tech decisions committed in `PLATFORM_DESIGN §7.6`.
- **2026-04-21** — Worker-shell architecture decided: iframe + bridge SDK. `CANVAS_SDK.md §3` (manifest), §4 (bridge protocol), §5 (auth handoff), §6 (onboarding playbook) filled.
- **2026-04-21** — Event plumbing re-architected into three explicit layers: Layer 1 outbox (atomic publish at source), Layer 2 hot bus (Redis Streams for online fan-out), Layer 3 durable log (Audit Log service for long-retention history). `PLATFORM_DESIGN §7.6` split; `COMPONENT_ARCHITECTURE §4.1` topology redrawn with swap triggers for Redis→Kafka; `§4.5` Audit Log upgraded from stub to full component spec.
- **2026-04-21** — `CANVAS_SDK.md §2.5` added (formerly §2.8): explicit v1 scope & non-goals. v1 ships flat `bridge.send` / `bridge.on` API, typed envelope, auth handoff, hand-written manifest. Explicitly deferred to v2: scripting-object wrapper (SSF-style `getObject`), interactive events with veto, `@agi-os/canvas-cli`, canvas-publishes-to-shell inversion, double-iframe HITL. Only forward-compat invariant: message names stay `<noun>.<verb>`. Build envelope: ~3 engineer-weeks.
- **2026-04-21** — `CANVAS_SDK.md §2` trimmed. The five-options brainstorm (Option A through E with per-option diagrams, comparison matrix, industry reference table) was removed from the doc body. The section now leads with the decision (iframe + bridge), one topology diagram, five-bullet rationale, explicit risk, reassessment trigger, and commitments. Rationale: a Principal-Engineer design doc presents the path, not a menu of paths. If an ADR is needed for the rejected options, it will live in a separate file rather than in the main architectural doc.
- **2026-04-21** — Worker shell domain renamed from `tasks.turing.com/ppt` → `canvas-agi-os.turing.com`. `/ppt` was a project-namespace collision (PPT = OpenClaw pay-per-task, not the platform). Canvas mount path shortened to `/:canvas_id/*`. Swept across `PLATFORM_DESIGN.md`, `CANVAS_SDK.md`, `README.md`. Also fixed a Mermaid lexical error in `CANVAS_SDK.md §2.2` Option C diagram (dot in `React.lazy` label confused parser; renamed to `lazy import`).
