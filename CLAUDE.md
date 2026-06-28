# MDM - AI Context (CLAUDE.md)

**Version**: 0.1.0
**Status**: 00 - FOUNDATION (Design Phase)
**Updated**: 2026-06-28
**Framework**: SDLC 6.4.0 LITE

---

## Project Overview

NQH Master Data Management (MDM) — app-bridge MVP-lean consolidating 7 core MDG domains (002 Supplier · 003 Product · 004 Site · 005 Customer · 008 Food · 011 Cost · 012 Equipment) with RBAC + record-lock + validation + Excel V2 round-trip + durable costing-engine.

**Core Value**: Kill multi-owner master-data integrity risk (5-owner-no-RBAC + 60M-bug evidence) NOW with portable engine that ports to BFlow 2 at sunset with 0-waste.

---

## Current Status

| Metric | Value |
|--------|-------|
| Stage | 00 - FOUNDATION |
| Sprint | 0 - scaffolding |
| Progress | 5% |
| Next Gate | G0 ratify post engine-review checklist |
| Build trigger | 🔴 GATE on COGS-PASS + engine-review PASS |

---

## Tech Stack (proposed — to ratify in 02-design)

```yaml
Backend:
  - Python 3.12+ (FastAPI or Flask — em recommend FastAPI for OpenAPI + async)
  - ClickHouse NQH (192.168.2.10:8123) for read-side via existing dim_* tables
  - PostgreSQL on S2 LXC 210 for write-side master-data (transactional + RBAC)
  - Redis (192.168.2.10) for record-lock + cache

Frontend:
  - Vue 3 or React (em recommend Vue 3 — Quỳnh team Vue familiarity per prior)
  - Vuetify or Tailwind UI

DevOps:
  - Docker Compose (S2 LXC alongside VatDownload + ClickHouse)
  - cron for daily Excel V2 sync/snapshot
  - Telegram /v1/send for alerts (NQH-Ops-Team channel)
```

**Final stack chốt** in ADR-001 by @architect (CEO + em joint). Ms. Quỳnh implements per ratified design (NOT design participant per CEO 28/06).

---

## Architecture (proposed — to ratify ADR-001)

```
[Excel V2 (Thy/Quỳnh)] ──poll──▶ [MDM Web App (FastAPI + Vue)] ──▶ [PostgreSQL write]
                                          │                              │
                                          │ RBAC + record-lock           │
                                          │ + validation                 │
                                          ▼                              ▼
                                  [Costing Engine] ◀───── [ClickHouse NQH read]
                                          │
                                          ▼
                                  [Excel V2 round-trip export]
                                  [DWH feed (dim_nvl_master / dim_dish_master / etc.)]
                                          │
                                          ▼
                                  [Dashboards 22/19 · @bod queries]
```

**Sunset path**: PostgreSQL master-data + costing-engine port to BFlow 2 with 0-waste (per durable-engine §E portability — ADR-005 of clickhouse/docs ratified 28/06).

---

## Critical Files (post-scaffold; em + @architect author next)

| Path | Purpose |
|------|---------|
| `docs/00-foundation/00-vision-mdm.md` | Vision + Decision Log cite |
| `docs/01-planning/REQ-001-mvp-lean-prd.md` | PRD MVP-lean 7-MDG CRUD + RBAC + Excel round-trip |
| `docs/02-design/ADR-001-stack-choice.md` | Tech stack ratify (FastAPI/Vue 3 vs alternatives) |
| `docs/02-design/ADR-002-rbac-record-lock-design.md` | RBAC + record-lock spec |
| `docs/02-design/ADR-003-durable-costing-engine-port-path.md` | Sunset portability to BFlow 2 |
| `docs/02-design/SPEC-001-engine-review-checklist.md` | Engine-review gate checklist (em draft post @mis green-light) |

---

## Cross-References

- **SDLC Framework canonical**: `/home/nqh/shared/.sdlc-framework/` (symlink → SDLC-Orchestrator/SDLC-Enterprise-Framework)
- **@mis Decision Log**: `/home/nqh/shared/models/core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Operations/DECISION-MDM-BUILD-WHERE-28062026.md` (commit `785ffd07`)
- **@mis MDM-WEBAPP-SPEC**: same dir/HANDOFF-ITADMIN-MDM-WEBAPP-SPEC-28062026.md
- **COGS pipeline design**: `/home/nqh/shared/models/apps/clickhouse/docs/` (16 specs)
- **VatDownload→DWH design**: `/home/nqh/shared/models/apps/VatDownload/docs/` (7 docs)
- **Excel V2 SSOT**: same RAG dir/`MASTER-DATA-V2.xlsx`

---

## Discipline Anchors (em CC binding)

- **§R-83 substrate-owner**: Ms. Quỳnh implements (per CEO 28/06); em design + handoff; @mis governance; Ms.Hồng + KTT crosswalk content. Em NOT cross-substrate modify.
- **§R-37 invariant**: cost lookup uses transaction_datetime, NEVER now() (extended from COGS pipeline ADR-001)
- **§R-71 SA discipline**: service account credentials rotation ≤90d, registry entry
- **§R-82 ext #15**: verify data-presence empirical, NOT assume from column-existence (per `feedback_verify_data_presence_not_just_columns`)
- **§R-85 refuse-to-fabricate**: em NOT predict crosswalk content / build outcome / @mis ratify

---

## Build Sequence (post Decision Log)

```
1. DESIGN (current): @architect (CEO + em joint) — Ms. Quỳnh NOT participate design (engine-review checklist draft post @mis green-light)
2. Engine-review (gate): @mis + @cto-CC review checklist + ratify
3. GATE COGS-PASS + ETA-Q4+ confirm: triggers MVP-lean BUILD
4. BUILD: Ms. Quỳnh implements per ratified design (Phase 1 = 7-MDG CRUD + RBAC + costing-engine)
5. TEST: 05-test verification per acceptance criteria
6. DEPLOY: 06-deploy ops runbook + RBAC config
7. Sunset to BFlow 2 (post-BFlow-2 GA): port costing-engine + carve Phase 3 to BFlow 2
```

---

## Owner / Roles (per CEO 28/06 final allocation)

| Role | Person/Agent | SE4A/SE4H | Scope |
|------|---|---|-------|
| Sponsor | CEO Đặng Thế Tài | SE4H | Authority |
| **@architect + SE4H** | **CEO Tài** | SE4H | Design + human-engineering authority (per CEO 28/06) |
| **@architect (joint)** | @itadmin (em) | SE4A | Co-architect with CEO + orchestrator + handoff coord |
| **@fullstack** | **Ms. Quỳnh** | SE4A leader | LITE-tier consolidation: @coder + @tester + @devops merged (per CEO LITE doctrine 28/06) |
| **@coder agent** | CC (Claude Code) | SE4A | Parallel code stream — deep context + LOGIC-SPEC interpretation + tests |
| **@coder agent** | Kimi | SE4A | Parallel code stream — bulk mechanical (DDL→ORM gen + Excel parsing + Vue scaffolding) |
| **DE (on-demand)** | dtdanh | SE4A | DWH owner — coordinate when MDM↔ClickHouse interop needed (per CEO 28/06) |
| Reviewer | @cto MTClaw | SE4A | PR review + design verify |
| Governance | @mis | SE4A | Decision Log + scope discipline + Empirical-verify (RATIFY SPEC-001 28/06) |
| Data Stewards | Per MDP SOP roles: Thy (MDG-003) · Ms.Hồng (MDG-002) · KTT + Kế toán Kho (MDG-008/011) · Bếp trưởng (MDG-005) · Hào (MDG-012) · CFO Office (MDG-009) | Human | per SPEC-004 RBAC matrix |
| Cross-team | @cto BFlow | SE4A | Sunset coordination |

### SE4A multi-agent execution model (per CEO 28/06)

For implementation phases (04-build), Ms. Quỳnh leads as @fullstack while orchestrating CC + Kimi parallel agent streams:

```
[CEO @architect SE4H] ──ratify design──▶
    │
    ▼
[em @architect joint SE4A] ──coordinate + clarify spec──▶
    │
    ▼
[Ms. Quỳnh @fullstack SE4A leader]
    │ Daily PR review + integrate + acceptance
    │
    ├──▶ [CC @coder agent stream A] (foundation + RBAC + engine)
    ├──▶ [CC @coder agent stream B] (MDG CRUD batch)
    ├──▶ [Kimi @coder agent stream C] (Excel parser + Vue scaffolding)
    └──▶ [Kimi @coder agent stream D] (DDL→ORM gen + migrations)
            │
            ▼ when DWH interop needed
    [dtdanh DE on-demand] (apps/clickhouse coordination)
            │
            ▼
[Demo + @cto-CC + @mis sign-off]
```

LITE tier consolidation: Ms. Quỳnh holds @coder + @tester + @devops as single @fullstack (no role-separation overhead for 1-2 person team).

---

## What NOT to do (anti-patterns)

- ❌ Predict BFlow 2 ETA (CEO-override Q3 OR Q4+ — em + @architect respect timeline as-is)
- ❌ Add Phase 2/3 scope to MVP (per Decision Log defer — only build when MVP-prove)
- ❌ Replace CukCuk sales OR VatDownload HĐ thuế (non-goal per Decision Log)
- ❌ Build cash-consolidation PROCESS in MDM (carve to BFlow 2)
- ❌ Skip engine-review checklist gate (durable-engine PASS = trigger 1 of 2)

---

*CLAUDE.md per SDLC 6.4.0 AI-ONBOARDING-TEMPLATE standard*
