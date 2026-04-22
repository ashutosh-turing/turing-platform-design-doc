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
| 6 | [PRD_ALIGNMENT.md](./PRD_ALIGNMENT.md) | Maps the product-side PRD (`Central Task-Based Platform`) onto the engineering design. Terminology map, gap analysis, cross-reference table. Start here if you're coming from the PRD side. |
| 7 | [KICKOFF.md](./KICKOFF.md) | **Implementation plan.** Dependency graph, phase ordering (sequential vs parallel), per-item BOE sizing (S/M/L/XL). No calendar — EM owns that. Read after the design is approved. |

## Reading by audience

| Who | Read | Can skip |
|---|---|---|
| Leadership | `PRD_ALIGNMENT §1` + `PLATFORM_DESIGN §2` (TL;DR) + `§7.8` (scale) | Everything else |
| Product team (PRD-side readers) | `PRD_ALIGNMENT.md` (full) | — |
| Platform engineers | `PLATFORM_DESIGN` + `COMPONENT_ARCHITECTURE` | — |
| Canvas engineers (TB, GDPVal, OpenClaw, …) | `CANVAS_SDK` first, then `PLATFORM_DESIGN §4–§6` | `§14–§17` |
| IH team | `IH_GAP_ANALYSIS` + `COMPONENT_ARCHITECTURE §4.3`, `§5.8` | — |
| SRE / infra | `PLATFORM_DESIGN §7.6` (tech decisions) + `COMPONENT_ARCHITECTURE` topology diagrams | — |
| EM / staff eng scoping implementation | `KICKOFF.md` (dependencies + BOE) + `PLATFORM_DESIGN §14` (rollout) | — |

## Status dashboard

| Doc | Version | Status |
|---|---|---|
| `PLATFORM_DESIGN.md` | 0.9 | Through §7 complete. §7.6 splits event plumbing into three layers; §7.8 pins the concurrency envelope (1M registered / 200K peak concurrent) with per-component SLOs against peak burst. §4.2 puts IH as the integration plane; §15.2 is now "Canvas registration (via IH)". §10 filled with the reconciled v1 catalog (**18 events** in three groups) aligned to the §11 state machine. §11 state machine adds `Escalated` + `PermanentlyRejected` per PRD §11.8.1. Zero YAML-manifest references. |
| `COMPONENT_ARCHITECTURE.md` | 0.5 | §0 Shell Offerings landing. §4.3 reframed as Integration Plane (three shapes: customer/canvas/platform-provider); §4.3.4 adds slug-cache + bus-invalidation. New Zone A: §4.8 Gateway (hot-path discipline) + §4.9 Notification Edge (SSE tier, 25K sockets/pod). §5.1 User Pool — hot-path discipline + new `ConcurrentClaimLimit` and `VettingRequired` eligibility policies (PRD alignment). Three new stubs from PRD: §5.13 Canvas Template, §5.14 Vetting, §5.15 Task Marketplace. 10 of 24 components filled. |
| `CANVAS_SDK.md` | 0.8 | Canvas registration owned by IH (§3 is a pointer, not a schema). §2.6 enumerates first-party worker-shell views (Marketplace, Earnings, Vetting, Settings) vs canvas-owned right pane (PRD alignment). §4.5 MCP-shaped runtime capability discovery. §5 two credential types (backend client creds + per-worker session JWT). `canvas_id` platform-assigned. **§7 SDK Repository**: separate repo `turing-canvas-sdk`, TypeScript-only. **Two public SDKs by role: `@turing/canvas-host` (shell) + `@turing/canvas-guest` (canvas)**, plus `@turing/canvas-schemas` (JSON Schema only, language-agnostic) for polyglot backends. Shared internals in private `_internal/bridge-core` bundled at build time (not published). pnpm + turbo + changesets, bidirectional Playwright contract tests. Kickoff ordering + BOE in §7.9. |
| `EVENT_CATALOG.md` | 0.2 | Envelope frozen (11 fields). V1 catalog names + groupings frozen: **18 events in three groups** (10 task-lifecycle including `TaskEscalated` + `TaskPermanentlyRejected` for Senior QA path, 5 unit-lifecycle pay-per-task-specific, 3 canvas-lifecycle). Payload notes locked for escalation-relevant events. Payload schemas + idempotency formulas pending Phase F1 implementation. |
| `IH_GAP_ANALYSIS.md` | 0.0 | Outline + current-state table; target & migration plan pending. |
| `PRD_ALIGNMENT.md` | 0.1 | Initial alignment pass mapping `PRD_ Central Task-based Platform.docx` (April 2026) onto the engineering design. Terminology map, gap analysis, cross-reference table. Three real gaps surfaced and closed (escalation states, vetting, task marketplace). |
| `KICKOFF.md` | 0.3 | Implementation kickoff — **dependencies + BOE sizing only, no timelines**. Phase 0 = ADR-001 (day-1 gate, ~1 day). Two streams run in parallel from day 1: **Stream 1 (Foundation)** F1 → F2 → F3 → F4 serial (~31–53 eng-days, F1 ships all **18** canonical events); **Stream 2 (Shell + SDK)** S1–S4 event-independent (~31–47 eng-days) → S5/S6 converge post-F3/F1 (~18–30 eng-days). Phase P1 = 3 parallel tracks after F4 (A: IH refactor, C: User Pool, D: gateway discipline). Phase 2 = OpenClaw canary. Total ~193–308 eng-days; EM computes calendar from team size. |

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

- **2026-04-21** — **Canvas SDK renamed to role-based pair: `@turing/canvas-host` + `@turing/canvas-guest`.** Previous names (`canvas-bridge` / `canvas-shell-host`) mixed metaphors — "bridge" could mean either side. Host/guest is unambiguous (standard term in iframe, VM, and browser-extension ecosystems). Shared internal code (envelope, correlation, error taxonomy) demoted from its own published package (`bridge-core`) to a **private workspace package** (`packages/_internal/bridge-core`, `"private": true`) that is bundled into both host and guest at build time — consumers see exactly two SDKs on npm plus `@turing/canvas-schemas` (the language-agnostic contract artifact for polyglot backends). Net effect: three public packages (was four), clearer naming matching consumer mental model, zero change in wire protocol or capability set. `CANVAS_SDK.md` §7.2 repo structure, §7.3 published-artifacts table, §7.9 kickoff steps, §4.4 code examples, and §2.1 topology diagram all refreshed. `KICKOFF.md` S1 deliverables updated. `PLATFORM_DESIGN.md §7.6` tech-decisions row updated. `CANVAS_SDK.md` bumped to v0.8.
- **2026-04-21** — **Canvas SDK repository decided — separate repo `turing-canvas-sdk`, TypeScript-only.** After reading the PRD, confirmed Canvas SDK is the right starting point (PRD §13.1 Template System + §13.2 Unified Integration Layer + §14.3 Shell/Canvas Model + §14.4 Auth — the MVP literally cannot ship without the bridge contract). Committed to a separate repo (not inside `agi-os`) to enforce boundary discipline and independent semver. Client code is TypeScript-only; no Python client SDK is maintained by platform. The contract stays language-agnostic via a separate `@turing/canvas-schemas` package containing only JSON Schema files — polyglot canvas backends (Python, Go, Node) consume the schemas via native validators (`jsonschema`, etc.) with ~15 lines of code. Four published packages: `@turing/bridge-core` (internal), `@turing/canvas-bridge` (canvas guest), `@turing/canvas-shell-host` (shell host), `@turing/canvas-schemas` (wire format source of truth). Tooling: pnpm + turbo + changesets + bidirectional Playwright contract tests. Full spec in `CANVAS_SDK.md §7` with repo layout, published artifacts, versioning policy, CI pipeline, consumers, scope fence, and BOE-sized kickoff ordering (~2–3 eng-weeks for `0.1.0-alpha.1`). `CANVAS_SDK.md` bumped to v0.7.
- **2026-04-21** — **PRD alignment pass.** Read product-side PRD (`PRD_ Central Task-based Platform.docx`, April 2026) and reconciled with engineering design. Three real gaps surfaced and closed: (1) State machine missing `Escalated` + `PermanentlyRejected` per PRD §11.8.1 — added to `PLATFORM_DESIGN §11` with four new transitions covering max-rework-exhaustion / integrity-flag / contractor-appeal triggers. (2) Vetting / pre-production pipeline missing — added as `COMPONENT_ARCHITECTURE §5.14 Vetting (Zone B)` with eligibility-filter contract via new `VettingRequired` policy. (3) Task Marketplace was unspec'd — added as `COMPONENT_ARCHITECTURE §5.15` first-party worker-shell view, with `CANVAS_SDK §2.6` enumerating all first-party views vs canvas-owned right pane. Additional additions: `ConcurrentClaimLimit` eligibility policy matching PRD §11.3.1 tier ladder (3/7/15); `§5.13 Canvas Template` Zone B stub for the reference containerized canvas PRD §13.1 describes. V1 event catalog bumped 16 → 18 (`TaskEscalated` + `TaskPermanentlyRejected`). New doc `PRD_ALIGNMENT.md` maps every PRD section to design location. No architectural conflicts found; PRD and engineering design describe the same platform at different altitudes.
- **2026-04-21** — **Canonical event catalog reconciled across all three docs.** Earlier drafts drifted: `KICKOFF.md` said "12 events," `EVENT_CATALOG.md` proposed 14, and neither matched `PLATFORM_DESIGN §11`'s state machine (which needs the three QC-emitted task events `TaskValidated` / `TaskReworked` / `TaskRejected`). Reconciled to **16 events in three groups**: 8 task-lifecycle (customer contract, state-machine-aligned), 5 unit-lifecycle (pay-per-task-pool-specific, feeds billing), 3 canvas-lifecycle (IH registration, consumed by shell slug cache). Names and groupings now frozen; payload schemas + idempotency formulas still pending Phase F1 implementation. `PLATFORM_DESIGN §10` filled with the authoritative list (was a pointer); `EVENT_CATALOG §3` replaced the "proposed list" with the locked catalog plus a `§3.4 Deferred from v1` table calling out the operational / HITL / batching event families that intentionally miss v1.
- **2026-04-21** — **`KICKOFF.md` v0.3 — parallelized starting point.** Shell + SDK work is ~70% event-independent; serializing it behind Phase F was overcautious. Rewrote around two streams running in parallel from day 1: **Stream 1 (Foundation)** stays strictly serial (F1 envelope → F2 outbox → F3 Redis Streams → F4 Audit Log); **Stream 2 (Shell + SDK)** runs alongside — S1 canvas-sdk bridge + S2 worker-experience-service + S3 shell app + S4 staging E2E demo all start day 1, then S5 notification-edge converges at F3 and S6 `task.event` wiring converges at F1. Only hard day-1 gate is `ADR-001 Event envelope v2` (~1 day). Phase P1 after F4 shrinks to 3 tracks (A: IH refactor, C: User Pool, D: gateway discipline) since most of old Track B is absorbed into Stream 2. Phase 2 still OpenClaw canary. Total BOE ~193–308 eng-days; 2-engineer critical path dominated by Stream 1 (31–53 days) then Track A (40–60 days).
- **2026-04-21** — **`KICKOFF.md` v0.2 — rewrote without timelines.** Reframed around dependency ordering (sequential vs parallel) and BOE sizing (S/M/L/XL, vibe-coding assumed). Dropped all week assignments — EM owns calendar math from total BOE + team size + focus factor.
- **2026-04-21** — **Concurrency envelope pinned at 1M registered / 200K peak concurrent.** Every hot-path component now carries a numeric SLO against peak burst. Notification socket-termination split into Zone A `§4.9 Notification Edge` (25K sockets/pod, FCM/APNS fallback). Gateway `§4.8` has explicit hot-path discipline: stateless, token-native auth, async audit via outbox, per-worker Redis rate limit, circuit-broken downstream pools. User Pool `§5.1` upgraded to hybrid Postgres-for-claim + Redis-for-heartbeat with committed SLOs (claim p99 100 ms, heartbeat p99 10 ms at 50K QPS). IH slug resolution `§4.3.4` gets two-tier cache invalidated via `Canvas*` events. New `PLATFORM_DESIGN.md §7.8` ties it together.
- **2026-04-21** — **Canvas registration moved to Integration Hub.** The `canvas-manifest.yaml` design was rejected as identity-and-state mishoused as a deployment artifact. IH is now the single integration plane for three shapes: customer, canvas, platform-provider integrations. All share the same data model, admin UI, Secrets Service integration, audit trail, RBAC. `canvas_id` is platform-assigned (`cvs_<nonce>`), immutable. Capability adoption is runtime discovery (MCP-shaped: `capabilities.list` / `describe` / `use`). Authorization via session JWT `scope` claim, derived from IH grants.
- **2026-04-21** — Event plumbing re-architected into three explicit layers: Layer 1 outbox (atomic publish at source), Layer 2 hot bus (Redis Streams for online fan-out), Layer 3 durable log (Audit Log service for long-retention history). `§4.1` topology redrawn with swap triggers for Redis → Kafka; `§4.5` Audit Log upgraded from stub to full component spec.
- **2026-04-21** — Worker-shell architecture decided: iframe + bridge SDK. `CANVAS_SDK.md §2` (topology), §4 (bridge protocol), §5 (auth handoff), §6 (onboarding playbook) filled. Worker shell domain: `canvas-agi-os.turing.com`. Canvas mount path: `/<slug>/*`.
- **2026-04-21** — `CANVAS_SDK.md §2.5` added: explicit v1 scope & non-goals. v1 ships flat `bridge.send` / `bridge.on` API, typed envelope, auth handoff. Explicitly deferred to v2: scripting-object wrapper (SSF-style `getObject`), interactive events with veto, `@agi-os/canvas-cli`, canvas-publishes-to-shell inversion, double-iframe HITL. Only forward-compat invariant: message names stay `<noun>.<verb>`.
- **2026-04-21** — Multi-doc split. Content migrated from a single `PLATFORM_DESIGN.md` into the structure above. Terminology renamed seller → canvas. Mermaid diagrams added. Tech decisions committed in `PLATFORM_DESIGN §7.6`.
