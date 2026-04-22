# PRD Alignment — Central Task-Based Platform → AGI-OS Platform Design

> **Purpose.** One page that maps the product-side PRD (`PRD_ Central Task-based Platform.docx`, April 2026) onto the engineering-side design (`PLATFORM_DESIGN.md`, `COMPONENT_ARCHITECTURE.md`, `CANVAS_SDK.md`, `EVENT_CATALOG.md`, `KICKOFF.md`). Shows where the two documents describe the same thing with different words, where the engineering design extends the PRD, and where the PRD surfaced real gaps we've now closed.
>
> **Audience.** Leadership, product, platform engineering, canvas engineering.
>
> **Status.** v0.1 — initial alignment pass after reading the PRD.

---

## 1. TL;DR

| | |
|---|---|
| **Same platform, two altitudes** | The PRD and our design describe the same system. PRD is **what** and **why**; our design is **how**. No architectural conflicts. |
| **Three real gaps the PRD surfaced** | (1) Escalation states in the task state machine. (2) Vetting as a first-class capability. (3) Task Marketplace as a first-party view. All three now closed — see §4. |
| **Two events added to v1 catalog** | `TaskEscalated` + `TaskPermanentlyRejected`. Total bumped from 16 → 18. |
| **Three new component stubs** | `§5.13 Canvas Template`, `§5.14 Vetting`, `§5.15 Task Marketplace` in `COMPONENT_ARCHITECTURE.md`. |
| **What our design adds over PRD** | Event plumbing (outbox/Redis Streams/Audit Log), concurrency envelope to 200K concurrent, Integration Hub as canvas-registration plane, hot-path discipline, three-zone governance. The PRD correctly delegates these to engineering. |

---

## 2. Terminology map

The PRD and this design use different words for the same things. Use this table when reading across documents.

| PRD term | Design term | Where |
|---|---|---|
| Central Task-Based Platform | AGI-OS platform | `PLATFORM_DESIGN §1` |
| AGIOS | AGI-OS admin shell (`agi-os.turing.com`) | `PLATFORM_DESIGN §6` |
| Admin Platform | AGI-OS admin shell | — |
| Trainer / Annotator Platform | Worker Shell (`canvas-agi-os.turing.com`) | `CANVAS_SDK §2` |
| Central task canvas / Task pane | Task Execution Canvas (Canvas) | `PLATFORM_DESIGN §6.2`, `CANVAS_SDK §2` |
| Task marketplace | First-party worker-shell view (`/tasks`) | `CANVAS_SDK §2.6`, `COMPONENT_ARCHITECTURE §5.15` |
| Queue Management / Routing | User Pool + claim strategies | `COMPONENT_ARCHITECTURE §5.1` |
| Quality Management | QC / Prism (Zone B) | `COMPONENT_ARCHITECTURE §5.3` |
| Pay-per-task (PPT) | PPT flow + billing contract (Zone A emit) | `PLATFORM_DESIGN §10.2`, `§4.2` |
| Rate modifiers | Billing consumer's concern (PRD rate-modifier table) | Billing team owns; platform emits `rework_attempt_number` etc. |
| Vetting / Pre-production | Vetting service (Zone B) | `COMPONENT_ARCHITECTURE §5.14` |
| Template System | Canvas Template (Zone B) | `COMPONENT_ARCHITECTURE §5.13` |
| Concurrent task limits (3/7/15 by tier) | `ConcurrentClaimLimit` eligibility policy | `COMPONENT_ARCHITECTURE §5.1` |
| Bi-directional roster sync | IH platform-provider integration (AGIOS as provider) | `COMPONENT_ARCHITECTURE §4.3.5` |
| Flex integration / Jibble integration | IH platform-provider integrations | `COMPONENT_ARCHITECTURE §4.3.5` |
| Rework Required / Escalated / Permanently Rejected (task states) | `Reworked` / `Escalated` / `PermanentlyRejected` | `PLATFORM_DESIGN §11` |

**Spelling note.** PRD uses "AGIOS" (no hyphen). Design uses "AGI-OS". Same thing.

---

## 3. What aligns cleanly (no design change needed)

The PRD and our design agreed on these before the PRD landed. Cited for auditability.

| PRD principle / requirement | Design element | Where |
|---|---|---|
| Two-system split (AGIOS = project/roster/financial; Task Platform = execution) | Admin shell vs Worker shell + AGIOS-as-platform-provider | `PLATFORM_DESIGN §4.2`, `§6` |
| Configuration over custom development (§7.1) | Three-zone governance + Canvas Template (§5.13) | `PLATFORM_DESIGN §4` |
| Central shell + flexible task pane (§7.2) | Iframe + Bridge SDK decision | `CANVAS_SDK §2` |
| One task model, multiple execution environments (§7.6) | Canonical state machine + canvas-side mapping | `PLATFORM_DESIGN §11` |
| Quality-gated payouts by default, submission-based override (§7.4) | Billing reads either `TaskAccepted` (QC-gated) or `TaskSubmitted` (submission-based) per `pay_model` config | `PLATFORM_DESIGN §10.1`, `COMPONENT_ARCHITECTURE §4.4` |
| Pay-per-task default + hourly + mixed (§9.6) | `pay_model` on `dos_pm_projects` already has `pay_per_task` / `fixed_aht` / `capped_aht` / `hourly` | Existing AGI-OS schema |
| Stable API contracts (tasks, quality, payment, access) (§14.2) | Canonical event catalog + IH capability contracts | `EVENT_CATALOG.md`, `COMPONENT_ARCHITECTURE §4.3` |
| Audit, governance, controls (§11.11) | Audit Log service (Layer 3) | `COMPONENT_ARCHITECTURE §4.5` |
| Reusable templates at the integration layer (§13.2) | IH as integration plane (customer + canvas + platform-provider shapes) | `COMPONENT_ARCHITECTURE §4.3` |
| Consistent authentication across embedded surfaces (§11.5, §14.4) | Canvas-scoped JWT minted by shell, JWKS verified by canvas backend | `CANVAS_SDK §5` |

---

## 4. Gaps the PRD surfaced — now closed

The PRD caught three things the engineering design had missed. All closed in this same revision.

### 4.1 Escalation in the task state machine

**PRD §9.3 and §11.8.1.** Tasks can be `Escalated` (to Senior QA) and ultimately `PermanentlyRejected`. Three escalation triggers: max rework exhausted, integrity flag, contractor appeal.

**Before:** `PLATFORM_DESIGN §11` went `Validated → Rejected → [*]` as the terminal unhappy path. No escalation, no Senior QA gate.

**Now:** Two new states (`Escalated`, `PermanentlyRejected`) and four new transitions — see `PLATFORM_DESIGN §11`. Two new events (`TaskEscalated`, `TaskPermanentlyRejected`) in `EVENT_CATALOG §3.1`. `TaskReworked.payload` now must carry `rework_attempt_number` so billing can apply the PRD's rework-penalty multipliers.

### 4.2 Vetting / Pre-production

**PRD §11.6 and §12.2.** Quizzes, task trials, golden datasets, pass/fail for production access, separate pre-production vs production states.

**Before:** Onboarding was implicit in IH canvas grants. No gate between "grant issued" and "can claim production work."

**Now:** `COMPONENT_ARCHITECTURE §5.14 Vetting (Zone B)` — platform owns the pipeline, canvases configure the content. `worker.vetting_status[canvas_id]` becomes an eligibility filter (`VettingRequired` added as stock policy in `§5.1`). Explicit v1 scope fence: not shipping with the OpenClaw canary, but the interface is designed now so User Pool doesn't need reshaping when the first vetted canvas lands.

### 4.3 Task Marketplace

**PRD §12.3.** Browsable, filterable list of eligible tasks — the trainer entry point.

**Before:** Worker shell spec had chrome + canvas iframe mount, but first-party views weren't enumerated. The implicit assumption was each canvas renders its own task list. That would have been wrong — a worker spanning 3 canvases would need 3 tabs.

**Now:** `COMPONENT_ARCHITECTURE §5.15 Task Marketplace` — first-party worker-shell view, backed by `worker-experience-service`. `CANVAS_SDK §2.6` enumerates all first-party views (Marketplace, Earnings, Vetting, Settings) vs canvas-owned right pane.

### 4.4 Concurrent claim limits by tier

**PRD §11.3.1.** 3/7/15 active claims per worker based on tier (New / Established / Expert).

**Before:** Gateway had per-worker QPS rate limiting, but no pool-level concurrent-claim ceiling.

**Now:** `ConcurrentClaimLimit` added as stock `EligibilityPolicy` in User Pool (`COMPONENT_ARCHITECTURE §5.1`). Worker tier derives from lifetime stats; default ladder matches PRD; per-canvas override supported.

### 4.5 Canvas Template system

**PRD §13.1.** One containerized, configurable reference canvas as a starting point for new projects.

**Before:** Canvases were treated as greenfield pluggable black boxes. No shared starting point — OpenClaw, TB, GDPVal all arrived from independent origins.

**Now:** `COMPONENT_ARCHITECTURE §5.13 Canvas Template (Zone B)` — `canvas-starter/` repo ships a working Dockerfile, `canvas.config.yaml`, prewired bridge SDK, React scaffold, example event emitters. OpenClaw (our canary) is the first consumer and its de facto functional test. Anti-goal made explicit: template is not a runtime dependency; forked copies diverge independently.

---

## 5. Where the design extends beyond the PRD

The PRD correctly skipped engineering detail. These extensions are design decisions the PRD delegates and we own.

| Design element | Why it extends the PRD | Where |
|---|---|---|
| Three-layer event plumbing (outbox / Redis Streams / Audit Log) | PRD §13.3 says "centralized ingestion." The three-layer split is how it holds up under bursts + reconciliation + 7-year audit. | `PLATFORM_DESIGN §7.6`, `COMPONENT_ARCHITECTURE §4.1` + `§4.5` |
| Concurrency envelope 1M registered / 200K concurrent | PRD §13.4 says "hundreds of concurrent." Our envelope is 1000× larger — deliberate headroom so the infra doesn't need re-architecture when volume grows. | `PLATFORM_DESIGN §7.8` |
| Notification Edge tier (SSE, 25K sockets/pod) | PRD doesn't address real-time delivery mechanics. We need a stateful tier. | `COMPONENT_ARCHITECTURE §4.9` |
| User Pool hot-path (Postgres claim + Redis heartbeat) | PRD doesn't address claim-rate vs heartbeat-rate asymmetry. Hybrid design lets us hold the claim SLO without Postgres saturating on heartbeats. | `COMPONENT_ARCHITECTURE §5.1` |
| Integration Hub as canvas-registration plane | PRD §13.2 addresses external-tool integration but is silent on how canvases register. We use the same IH machinery for all three shapes (customer / canvas / platform-provider). | `COMPONENT_ARCHITECTURE §4.3` |
| Gateway hot-path discipline (token-native auth, async audit, circuit breakers, per-worker rate limit) | PRD silent. Our hot-path discipline is the difference between linear scale and a cliff. | `COMPONENT_ARCHITECTURE §4.8` |
| Three-zone governance (A platform-dictated / B platform-offered / C canvas-owned) | PRD has principles (§7); our zones are the enforcement mechanism. | `PLATFORM_DESIGN §4` |
| Per-component SLO matrix | PRD has `product/operational/user metrics` (§16) but not per-component technical SLOs. | `PLATFORM_DESIGN §7.8` |

---

## 6. Where the PRD is a tighter MVP than our design

The PRD's MVP (§15) is explicitly narrower than our Phase P1. That's fine — we build in layers, and Phase 0 + Phase F + Phase P1 set foundations regardless of whether the first canary fully uses them.

Concretely:

- PRD MVP has "one integrated external task environment or wrapper" — we're building for many canvases from day one, but **OpenClaw is our one-canvas canary** (`KICKOFF §8`). Same shape.
- PRD MVP has "manual payment option via CSV plus one automated pathway with Flex" — our IH outbound covers both patterns; Flex is the first platform-provider integration wired via `§4.3.5`.
- PRD MVP has "basic routing logic" — we're building the full `ClaimStrategy` interface (stock: `FIFO`, `SkillMatch`, `TieredTTL`, `WeightedRandom`) but OpenClaw at Level 0 just picks one default. Extension is cheap.
- PRD MVP has "basic analytics" — we're building Audit Log in Phase F4; dashboards (`§5.12 Observability`) are explicitly Pass 3.

**No divergence.** PRD MVP is a subset our architecture naturally supports at the point the canary ships.

---

## 7. Decisions the PRD leaves open (PRD §17)

The PRD lists open questions. Our design answers some; flags others for product decision.

| PRD open question | Design position |
|---|---|
| Should task setup live in AGIOS, task platform, or hybrid? | **Hybrid, with clear split.** AGIOS owns project creation (existing), task platform owns task ingestion + execution. IH bridges the two via canvas registration. See `COMPONENT_ARCHITECTURE §4.3.4`. |
| Should the platform be single system or multi-instance by project type? | **Single multi-tenant platform; per-canvas isolation via `canvas_id` + RBAC + pool sharding.** Rationale: `PLATFORM_DESIGN §4`. |
| Where should task decomposition live? | **In the canvas.** Platform sees the output (`TaskCreated`), not the decomposition logic. Zone C. |
| How configurable should the standard task canvas be? | **Every UI zone configurable via `canvas.config.yaml`; business logic is code, not config.** See `§5.13 Canvas Template`. |
| How far should metric standardization go? | **Canonical events + canonical state machine are the lowest common denominator.** Beyond that, canvases map their internal metrics to a reporting shape (`PLATFORM_DESIGN §11`). |
| Where should disputes and appeals live operationally? | **Appeal triggers `TaskEscalated`; Senior QA owns adjudication.** Stub component `§5.10 Dispute / Appeal` is placeholder for future formalization. |
| What identity model for mixed internal/external talent? | **One identity service; IH grants scope per-canvas.** Internal vs external distinction is a grant attribute, not a separate identity system. `CANVAS_SDK §5`. |
| Latency/stability for embedded custom canvases? | **Iframe sandbox + bridge protocol; bridge timeouts + version negotiation give graceful degradation.** `CANVAS_SDK §4`. |

---

## 8. Cross-reference table

Use this to find the design answer to any PRD section.

| PRD § | Topic | Design § |
|---|---|---|
| §1 Product Intent | Vision | `PLATFORM_DESIGN §1–§2` |
| §2 Executive Summary | TL;DR | `PLATFORM_DESIGN §2`, this doc §1 |
| §3 Product Vision | Configuration-first platform | `PLATFORM_DESIGN §4` |
| §7 Core Principles | Zones + seams principle | `PLATFORM_DESIGN §4` |
| §8 Product Scope (Admin / Trainer / AGIOS Integration) | Three surfaces | `PLATFORM_DESIGN §6`, `CANVAS_SDK §2` |
| §9.1 Core Work Unit | Task schema | `PLATFORM_DESIGN §10`, `EVENT_CATALOG §2` |
| §9.2 Task Types | Canvas-owned (Zone C) | `PLATFORM_DESIGN §4` |
| §9.3 Task States | Canonical state machine | `PLATFORM_DESIGN §11` |
| §9.4 Routing / Queue Management | User Pool claim strategies | `COMPONENT_ARCHITECTURE §5.1` |
| §9.5 Eligibility | `EligibilityPolicy` | `COMPONENT_ARCHITECTURE §5.1` |
| §9.6 Payment Model | Billing contract (Zone A emit) | `PLATFORM_DESIGN §10.2`, `COMPONENT_ARCHITECTURE §4.4` |
| §9.8 Rework and Escalation | Escalation transitions | `PLATFORM_DESIGN §11`, `EVENT_CATALOG §3.1` |
| §11.1 Project and Task Configuration | Admin shell + IH | `COMPONENT_ARCHITECTURE §4.3` |
| §11.2 Task Injection | Canvas-owned + SDK | `CANVAS_SDK §4`, `EVENT_CATALOG §3.1` (`TaskCreated`) |
| §11.3 Queue Management | User Pool | `COMPONENT_ARCHITECTURE §5.1` |
| §11.3.1 Concurrent Limits | `ConcurrentClaimLimit` policy | `COMPONENT_ARCHITECTURE §5.1` |
| §11.4 Talent / Access Management | Identity + IH platform-provider integration | `COMPONENT_ARCHITECTURE §4.2`, `§4.3.5` |
| §11.5 Authentication | JWT + JWKS | `CANVAS_SDK §5` |
| §11.6 Vetting | Vetting Zone B | `COMPONENT_ARCHITECTURE §5.14` |
| §11.7 Quality Management | QC / Prism | `COMPONENT_ARCHITECTURE §5.3` |
| §11.8 Review Operations / §11.8.1 Escalation | State machine + events | `PLATFORM_DESIGN §11` |
| §11.9 Financial Management / Payout | Billing contract (Zone A) | `COMPONENT_ARCHITECTURE §4.4` |
| §11.10 Performance Analytics | Audit Log + Observability | `COMPONENT_ARCHITECTURE §4.5`, `§5.12` |
| §11.11 Audit, Governance, Controls | Audit Log | `COMPONENT_ARCHITECTURE §4.5` |
| §11.12 Communications and Support | Notification (§5.6) + Notification Edge (§4.9) | `COMPONENT_ARCHITECTURE §5.6`, `§4.9` |
| §12.1 Authentication and Entry | Canvas-scoped JWT handoff | `CANVAS_SDK §5` |
| §12.2 Onboarding and Vetting | Vetting service | `COMPONENT_ARCHITECTURE §5.14` |
| §12.3 Task Marketplace | First-party worker-shell view | `COMPONENT_ARCHITECTURE §5.15`, `CANVAS_SDK §2.6` |
| §12.4 Task Claiming | User Pool claim path | `COMPONENT_ARCHITECTURE §5.1` |
| §12.5 Task Workspace | Canvas iframe + bridge | `CANVAS_SDK §2` + `§4` |
| §12.6 Submission and Feedback | `TaskSubmitted` event + bridge `task.event` | `EVENT_CATALOG §3.1`, `CANVAS_SDK §4` |
| §12.7 Payments, Earnings | Earnings view (first-party) | `CANVAS_SDK §2.6` |
| §12.8 Performance Visibility | Observability + first-party view | `COMPONENT_ARCHITECTURE §5.12`, `CANVAS_SDK §2.6` |
| §13.1 Template System | Canvas Template Zone B | `COMPONENT_ARCHITECTURE §5.13` |
| §13.2 Unified Integration Layer | IH as integration plane | `COMPONENT_ARCHITECTURE §4.3` |
| §13.3 Data Model and Reporting | Event catalog + Audit Log | `EVENT_CATALOG.md`, `COMPONENT_ARCHITECTURE §4.5` |
| §13.4 Scalability and Reliability | Concurrency envelope + per-component SLOs | `PLATFORM_DESIGN §7.8` |
| §14 Technical Architecture | Zones + SDK + auth + shell/canvas separation | All five design docs |
| §15 MVP Recommendation | OpenClaw canary scope | `KICKOFF §7` (Phase 2) |
| §16 Success Metrics | Observability | `COMPONENT_ARCHITECTURE §5.12` |
| §17 Risks / Open Questions | See this doc §7 | — |

---

## 9. Document trail

| Change | Where | Status |
|---|---|---|
| Add `Escalated` + `PermanentlyRejected` states + transitions | `PLATFORM_DESIGN §11` | landed |
| Add `TaskEscalated` + `TaskPermanentlyRejected` events | `PLATFORM_DESIGN §10.1`, `EVENT_CATALOG §3.1` | landed |
| Payload requirement: `rework_attempt_number`, `escalation_reason`, `escalation_chain` | `EVENT_CATALOG §3.1` | landed |
| `§5.13 Canvas Template` (Zone B) | `COMPONENT_ARCHITECTURE.md` | stub landed |
| `§5.14 Vetting` (Zone B) | `COMPONENT_ARCHITECTURE.md` | stub landed |
| `§5.15 Task Marketplace` (first-party view) | `COMPONENT_ARCHITECTURE.md` | stub landed |
| `ConcurrentClaimLimit` + `VettingRequired` in `EligibilityPolicy` | `COMPONENT_ARCHITECTURE §5.1` | landed |
| First-party worker-shell views vs canvas-owned right pane | `CANVAS_SDK §2.6` | landed |
| v1 event catalog 16 → 18 | `PLATFORM_DESIGN §10`, `EVENT_CATALOG §3`, `KICKOFF §5 F1` | landed |

---

## 10. What's still open (needs product input)

Two items the PRD raised that the engineering design should not resolve unilaterally:

1. **Vetting content + policy** — who owns quiz authoring, what constitutes a "pass," when a failed worker can retry, whether failures propagate across canvases. Platform provides the pipeline; product + ops own the policy. **Decision owner: product + ops, per-canvas.**
2. **Appeal window duration + escalation SLAs** — PRD §11.8.1 specifies 48h / 72h review SLA by tier, but not appeal window duration or Senior QA SLA. Need product sign-off before F1 payload schemas lock. **Decision owner: product + QA ops.**

Neither blocks Phase F. Both should be answered before Phase 2 (OpenClaw canary).
