---
spec_id: VISION-001
spec_name: "MDM Vision — app-bridge MVP-lean to BFlow 2"
spec_version: "1.0.0"
status: draft
tier: LITE
stage: "00"
category: master-data
owner: itadmin-orchestrator
created: 2026-06-28
last_updated: 2026-06-28
---

# MDM Vision — app-bridge MVP-lean

**Authority**: @mis Decision Log `785ffd07` (28/06/2026) + CEO ratify

## 1. Problem

Multi-owner master-data integrity risk **acute NOW**:
- 5 owners (Thy / Quỳnh / Ms.Hồng / KTT / Hào) editing same Excel V2 master-data with NO RBAC + NO record-lock
- Evidence: 60M VND unit-error from concurrent edit (per @mis evidence Sprint 130 incident)
- BFlow 2 financial-engine ETA **Q4+ 2026** (branch chưa-merge + 9-migration + G-AgOS-4 chưa-pass; CEO-override only if Q3)
- Gap ≥ **4-5 months** until BFlow 2 ready → unacceptable to wait

## 2. Outcome

A working app-bridge that:
1. Eliminates concurrent-edit corruption (RBAC + per-record-lock + audit trail)
2. Validates master-data at entry (error-detection prevents bad data → silent downstream errors)
3. Round-trips Excel V2 (Thy/Quỳnh/Ms.Hồng workflow preserved + UI fallback)
4. Hosts **durable costing-engine** (port to BFlow 2 with 0-waste at sunset)
5. Feeds existing DWH (ClickHouse `dim_*`) + dashboards 22/19 + @bod queries

Success measure: **0 master-data corruption events / month** post-deploy (vs 60M-event baseline pre-MDM).

## 3. Strategic anchors

- **CEO Decision Log 28/06** (`785ffd07`): app-bridge MVP-lean robust-to-ETA chosen over wait-for-BFlow-2
- **3 lý do robust**: durable-engine port 0-waste · pain=risk không convenience · gap ≥ 4-5mo ROI
- **@mis governance**: scope MVP-lean ONLY; defer Phase 2/3; carve cash-consol PROCESS to BFlow 2
- **Sunset-first design** (per ADR-005 COGS pipeline + Decision Log): engine + schema portable

## 4. Scope per Decision Log

| Bucket | Items | Rationale |
|---|---|---|
| 🟢 **BUILD MVP-lean** | 7-MDG CRUD (002/003/004/005/008/011/012) + RBAC + record-lock + validation + Excel V2 round-trip + durable costing-engine | Kill acute pain NOW with portable engine |
| 🟡 **DEFER (ON-DEMAND)** | Online dashboard · MDG-009 Customer online · price-loop | Only if MVP-prove |
| 🔴 **CARVE → BFlow 2** | Cash-consol PROCESS · virtual-customer-AR · treasury (master-data MDG-009 + Bank-Account stays MDM feed BFlow 2) | BFlow 2 native scope |
| ❌ **Non-goal MVP** | Replace CukCuk sales · Replace VatDownload HĐ thuế | MDM = master-data + costing-method layer ONLY |

## 5. Sunset clause

MDM is INTERIM bridge. Sunset trigger ALL conditions:
1. BFlow 2 financial-engine GA (branch merged + 9-migration done + G-AgOS-4 PASS)
2. BFlow 2 master-data services ship (NVL · Recipe · Product · Customer · Site · Bank-Account)
3. Costing-engine ported BFlow 2 (per durable-engine §E portability checklist)

Per Mental Model #9 re-eval trigger: 2026-12-31 — confirm BFlow 2 trajectory; if slip Q1 2027+, MDM MVP-lean continues operation; if GA Q4 2026, sunset sprint begins.

## 6. AmendmentC L1-L5 drift-lessons compliance

- **L1 Govern-Proportional**: MDM LITE = minimal governance; expand only if USE/RISK scales
- **L2 Evidence-of-Need before BUILD**: BUILD gated on COGS-PASS + engine-review PASS (NOT speculative)
- **L3 Governance-Spiral Anti-Pattern**: monitor gov:code ratio; design phase = 100% gov is OK (pre-build); flip at BUILD
- **L4 Drift Circuit-Breaker**: outcome KPI = 0 corruption events/month post-deploy
- **L5 Dogfood**: em (@itadmin) dogfood for MDG-003 Product CRUD post-MVP launch

## 7. Out of scope (this vision)

- BFlow 2 internals (separate cross-team coordination @cto BFlow)
- Cash-consolidation PROCESS (CARVE to BFlow 2)
- Sales POS replacement (CukCuk stays)
- HĐ thuế download (VatDownload stays)
- Recipe-gap 114 món fill (rolling Bếp/KTT, non-block)

## 8. References

- Decision Log `785ffd07`: `core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Operations/DECISION-MDM-BUILD-WHERE-28062026.md`
- MDM-WEBAPP-SPEC handoff: same dir/HANDOFF-ITADMIN-MDM-WEBAPP-SPEC-28062026.md
- COGS pipeline ADR-005 (interim price source maturity model): `apps/clickhouse/docs/02-design/ADR-005-VAT-Interim-Price-Source.md`
- SDLC 6.4.0 AmendmentC: `/home/nqh/shared/.sdlc-framework/09-Continuous-Improvement/AmendmentC-Drift-Lessons-Catalog.md`
- Mental Model #9 Demand Before Surface: `/home/nqh/shared/.sdlc-framework/02-Core-Methodology/Ship-Useful-Principle.md` (or equivalent)
- Memory: `project_master_data_ownership` · `project_excel_master_data_ssot_transitional` · `feedback_centralized_nqh_data_ownership`
