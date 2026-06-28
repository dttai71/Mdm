# MDM — NQH Master Data Management

**Framework**: SDLC 6.4.0 LITE Tier (universal AI+Human methodology — see `/home/nqh/shared/.sdlc-framework` symlink (shared across NQH projects))
**Status**: 00 - FOUNDATION (Design Phase)
**Updated**: 2026-06-28
**Sponsor**: CEO Đặng Thế Tài
**Owners**: @itadmin (orchestrate) · @architect (design) · @cto MTClaw (review) · @coder Ms. Quỳnh (per CEO 28/06 — NOT design participant) (implement) · @mis (governance) · Thy/Quỳnh/Ms.Hồng (data stewards)

---

## Project Overview

NQH Master Data Management (MDM) is an **app-bridge MVP-lean** that consolidates 7 core master-data domains (Supplier/Product/Site/Customer/Food/Cost/Equipment) with RBAC + record-lock + validation + Excel V2 round-trip + durable costing-engine. Bridges 4-5 month gap until BFlow 2 financial-engine GA.

**Core Value**: Kill multi-owner master-data integrity risk (5-owner-no-RBAC + 60M-bug evidence) NOW with portable engine that ports to BFlow 2 with 0-waste at sunset.

---

## Current Status

| Metric | Value |
|--------|-------|
| Stage | 00 - FOUNDATION |
| Sprint | 0 - Project scaffolding |
| Progress | 5% (folder scaffold + initial docs) |
| Next Gate | G0 (Foundation) → @cto-CC ratify post engine-review checklist |
| Build trigger | 🔴 GATE on COGS-PASS + engine-review PASS |

---

## SDLC 6.4.0 LITE Tier Compliance

Per [SDLC-Enterprise-Framework README §4-Tier Classification](/home/nqh/shared/.sdlc-framework/README.md):

| LITE Required | Status |
|---|---|
| Required Stages: 00, 01, 02, 04 | 🟢 scaffolded |
| README | ✅ this file |
| .env.example | ✅ `.env.example` |
| Team 1-2 | ✅ Ms. Quỳnh primary (per CEO 28/06) + em design-coordinator |

Voluntary additions (per em prior project pattern):
- `CLAUDE.md` — AI-onboarding (STANDARD+ feature, em include for AI context velocity)
- Additional stage dirs (05/06/08) — kept as ON-DEMAND markers per Mental Model #9

---

## SDLC 6.4.0 Discipline Adherence

**Per AmendmentC L1-L5 drift-lessons (June 25, 2026)**:

- **L1 Govern-Proportional + Complexity-Budget**: MDM LITE = minimal governance overhead; expand only if USE/RISK scales
- **L2 Evidence-of-Need before BUILD/GOVERN**: BUILD trigger ON-DEMAND — gates on COGS-PASS + engine-review PASS (NOT speculative)
- **L3 Governance-Spiral Anti-Pattern**: gov:code ratio monitored; current = 100% gov (design phase) — flip when BUILD triggered
- **L4 Drift Circuit-Breaker**: outcome-not-volume; KPI = "master-data corruption events / month → 0" post-deploy
- **L5 Dogfood as Adoption-Test**: em (@itadmin) eat-own-dogfood for MDG-003 Product (em's own daily updates routed through MDM post-MVP)

**Mental Model #9 — Demand Before Surface**:
- Daily user: Thy/Quỳnh/Ms.Hồng (master-data stewards)
- Daily job: CRUD master-data without concurrent-edit corruption
- Re-eval trigger: 2026-12-31 — if BFlow 2 GA, MDM sunset begins; if BFlow 2 slip Q1 2027+, MDM MVP-lean continues operation
- ON-DEMAND markers: Phase 2 + Phase 3 scope (per Decision Log defer)

---

## Problem

Multi-owner master-data integrity risk (5-owner-no-RBAC + 60M-bug evidence) acute NOW. BFlow 2 financial-engine ETA Q4+ 2026 (branch chưa-merge + 9-migration + G-AgOS-4 chưa-pass; CEO-override only if Q3). Gap ≥ 4-5 months until BFlow 2 ready.

## Solution

**app-bridge MVP-lean** (robust-to-ETA) per @mis Decision Log ratify `785ffd07`:
- 7 core MDG domains CRUD (002 Supplier · 003 Product · 004 Site · 005 Customer · 008 Food · 011 Cost · 012 Equipment)
- RBAC + record-lock (kill concurrent-corruption)
- Validation + error-detection
- Excel V2 round-trip import/export (born-clean compatible)
- **Durable costing_engine** (port to BFlow 2 with 0-waste at sunset)

## Scope per Decision Log 28/06 (`785ffd07`)

| Phase | Scope | Trigger |
|---|---|---|
| **MVP** (Phase 1) | 7-MDG CRUD + RBAC/record-lock + validation + Excel round-trip + durable costing_engine | Post COGS-PASS + ETA Q4+ ✅ |
| **Phase 2** ON-DEMAND | Online dashboard · MDG-009 online · price-loop | Only if MVP-prove |
| **Phase 3** CARVE → BFlow 2 | Cash-consolidation PROCESS · virtual-customer-AR · treasury (master-data MDG-009 + Bank-Account stays in MDM feed BFlow 2) | BFlow 2 era |

**Non-goal MVP**: KHÔNG thay CukCuk (sales), KHÔNG thay VatDownload (HĐ thuế). MDM = lớp master-data + costing-method.

## Build sequence

```
durable-costing_engine (on COGS-PASS + engine-review PASS)
       │
       ▼
MVP-lean (on COGS-PASS + ETA-Q4+ ✅)
```

## Folder structure

```
/home/nqh/shared/models/apps/Mdm/
├── README.md                       # this file
├── CLAUDE.md                       # AI-onboarding
├── .env.example                    # SDLC LITE required
├── .gitignore
└── docs/
    ├── 00-foundation/    # WHY?   Vision + Decision Log + sunset clause
    ├── 01-planning/      # WHAT?  PRD MVP-lean + acceptance criteria
    ├── 02-design/        # HOW?   ADRs + schema + engine-review checklist
    ├── 04-build/         # BUILD: impl handoff to Ms. Quỳnh (post engine-review PASS)
    ├── 05-test/          # ON-DEMAND: verification plan when BUILD triggered
    ├── 06-deploy/        # ON-DEMAND: ops runbook when deploy ready
    └── 08-collaborate/   # Handoffs (itadmin → architect/Ms. Quỳnh/Ms.Hồng/Quỳnh/@mis)
```

## Cross-references

- **SDLC Framework canonical**: `/home/nqh/shared/.sdlc-framework/` (symlink → `/home/nqh/shared/SDLC-Orchestrator/SDLC-Enterprise-Framework`)
- **@mis Decision Log**: `core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Operations/DECISION-MDM-BUILD-WHERE-28062026.md` (commit `785ffd07`)
- **@mis MDM-WEBAPP-SPEC**: same dir/HANDOFF-ITADMIN-MDM-WEBAPP-SPEC-28062026.md
- **COGS pipeline design** (em prior): `apps/clickhouse/docs/` (16 specs — ADR-001 freeze + ADR-003 honest-ceiling + ADR-005 VAT interim + SPEC-0004 DDL)
- **VatDownload→DWH design** (em prior 28/06): `apps/VatDownload/docs/` (7 docs — REQ-017 + ADR-017/018 + INT-001/002)
- **Excel V2 SSOT**: `core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Operations/MASTER-DATA-V2.xlsx` (Drive `1K-SL4aeorrZtgvoTiOxarC_sNRewktAi`)
- **Memory anchors**: `project_master_data_ownership` · `project_excel_master_data_ssot_transitional` · `feedback_centralized_nqh_data_ownership`

## Sunset clause

MDM is INTERIM bridge to BFlow 2. Sunset trigger:
- BFlow 2 financial-engine GA (branch merged + 9-migration done + G-AgOS-4 PASS)
- Master-data services (NVL · Recipe · Product · Customer · Site · Bank-Account) ship in BFlow 2 SOP-native
- Costing-engine ported BFlow 2 with 0-waste (per durable-engine §E portability — engine-review checklist)

Per Decision Log: cash-consolidation PROCESS + virtual-customer-AR + treasury CARVE to BFlow 2 (NOT MDM scope). MDM owns master-data layer only.

## Git + Remote

- **Local**: `/home/nqh/shared/models/apps/Mdm/`
- **Remote**: https://github.com/dttai71/Mdm (public)
- **Branch convention**: main = ratified; feature/* = WIP; hotfix/* = urgent fix

## License

Internal NQH project. Not open-source. Code owners: dttai71 (CEO Tài) + ITteam.
