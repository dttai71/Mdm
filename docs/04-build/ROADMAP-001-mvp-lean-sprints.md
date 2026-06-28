---
spec_id: ROADMAP-001
spec_name: "MVP-Lean Sprint Roadmap (Sprint 1-6+ Outline)"
spec_version: "1.0.0"
status: planning
tier: LITE
stage: "04"
category: roadmap
owner: architect-itadmin-ceo-joint
created: 2026-06-28
last_updated: 2026-06-28
sprint_duration: 1 week (Sat→Fri NQH cycle)
---

# ROADMAP-001 — MVP-Lean Sprint Roadmap

**Authority**: Decision Log `785ffd07` (MVP-lean scope) + CEO 28/06 directive @architect (CEO + em joint)

## Roadmap principles

1. **Vertical-slice first** — Sprint 1 = full pipeline on 1 MDG (MDG-008 NVL) prove end-to-end
2. **Pattern-replicate** — Sprint 2-3 = same vertical pattern across remaining 6 MDGs
3. **Cross-cutting features later** — Sprint 4 (Excel V2) + Sprint 5 (engine) + Sprint 6 (UI) AFTER all 7 MDGs CRUD work
4. **Per L1 govern-proportional** — Sprint 7+ ON-DEMAND post MVP-prove (deploy/production-harden + Phase 2 features)

## Sprint schedule (conditional + stretch)

| Sprint | Theme | Duration | Conditional start | Stretch target |
|---|---|---|---|---|
| **1** | Foundation + RBAC + MDG-008 thin-slice | 1 week | T+0 post gates | 2026-07-15 (Wed) |
| **2** | MDG-002/003/004 CRUD (Supplier/Product/Site) | 1 week | T+7 days | 2026-07-22 |
| **3** | MDG-005/009/011/012 CRUD (BTP/Customer/Cost/Equipment) | 1 week | T+14 days | 2026-07-29 |
| **4** | Excel V2 round-trip import/export + sync workflow | 1 week | T+21 days | 2026-08-05 |
| **5** | Costing engine (LOGIC-SPEC impl + §R-37 gate + tests) | 1 week | T+28 days | 2026-08-12 |
| **6** | Vue 3 frontend MVP (7-MDG CRUD UI + Excel sync UI) | 1 week | T+35 days | 2026-08-19 |
| **7+** | ON-DEMAND: production deploy + Phase 2 features (per @mis sprint-by-sprint trigger) | TBD | post Sprint 6 demo + MVP-prove | TBD |

**Conditional gate (BLOCKING all sprints)**: COGS-PASS ✅ + engine-review checklist PASS ✅ + ETA-Q4+ ✅ per Decision Log

Total MVP = **6 weeks** (~42 calendar days from gate clear). Sprint 1 alone proves architecture; remaining 5 sprints = replicate-pattern + cross-cutting features.

## Sprint detailed scopes (high-level)

### Sprint 1 — Foundation + RBAC + MDG-008 NVL thin-slice
**Detail spec**: `SPRINT-001-foundation-rbac-mdg008-thinslice.md`

**Deliverables**: PostgreSQL schema applied · FastAPI scaffold · JWT auth · RBAC middleware · record-lock · audit log · MDG-008 NVL CRUD vertical slice

**Acceptance**: 7 US PASS · 7 TT done · CI green · Ms. Quỳnh demos NVL create→approve→update→soft-delete

### Sprint 2 — MDG-002/003/004 CRUD (3 MDGs)
**Theme**: Replicate MDG-008 pattern for Supplier (MDG-002) + Product (MDG-003) + Site (MDG-004)

**Deliverables**:
- `/api/v1/mdg-002` CRUD per SPEC-003 §2.4
- `/api/v1/mdg-003` CRUD + special `Ten_CukCuk_HoaDon` bridge field
- `/api/v1/mdg-004` CRUD + `is_pc_authoritative` flag per project_bu_pc_structure_27062026
- Recipe endpoint stub `/api/v1/recipes/mon/{ma_mon}` (read-only Sprint 2 → editable Sprint 5)
- Test fixtures + RBAC enforcement per SPEC-004

**Acceptance**: 4 MDGs (002/003/004 + 008 from Sprint 1) full CRUD lifecycle · all RBAC TC pass · ≥ 80% test coverage

### Sprint 3 — MDG-005/009/011/012 CRUD (4 MDGs)
**Theme**: Complete 7-MDG coverage

**Deliverables**:
- `/api/v1/mdg-005` BTP CRUD + yield_qty + yield_uom validation
- `/api/v1/recipes/btp/{ma_btp}` upsert (BTP recipe components per SPEC-005 §3.2)
- `/api/v1/mdg-009` Customer CRUD + PDPL 2025 consent fields
- `/api/v1/mdg-011` Cost master CRUD
- `/api/v1/mdg-012` Equipment CRUD (TSCĐ + CCDC distinction)
- Recipe Mon BOM endpoint editable

**Acceptance**: All 7 MDGs full CRUD · RBAC per MDP SOP doctrine · all 7 owner approval workflows tested

### Sprint 4 — Excel V2 round-trip
**Theme**: Implement SPEC-005 round-trip protocol

**Deliverables**:
- `/api/v1/excel/import` async upload + parse + diff
- `/api/v1/excel/sync/{run_id}/diffs` review + accept/reject/conflict-merge
- `/api/v1/excel/sync/{run_id}/commit` apply accepted
- `/api/v1/excel/export` born-clean v2 generation
- Drive service account integration (read-only Phase 1)
- Daily 06:00 cron poll (with sync_run idempotency)
- Validation rules per SPEC-005 §4 enforced

**Acceptance**: Import golden fixture `MASTER-DATA-V2.xlsx` → all 7 MDGs populated · export round-trip = bit-for-bit identical (modulo timestamps) · conflict detection works · Thy/Quỳnh manual review demo

### Sprint 5 — Costing engine (LOGIC-SPEC impl)
**Theme**: Implement SPEC-006 OP-1/OP-2/OP-3 + SPEC-007 fixture tests

**Deliverables**:
- `compute_recipe_cost` recursive (RAW→PRE/RTU/CON leaf-stop + INT yield divide + PRD/PRD combo per SPEC-006 §2)
- `freeze_bill_line` per-bill point-in-time (§R-37 gate enforced)
- `lookup_nvl_effective_price` multi-source priority (per ADR-005)
- 5-class anomaly emission + Telegram routing per WI-105 tier
- `/api/v1/engine/*` endpoints per SPEC-003 §2.7
- Test fixtures per SPEC-007 §2-7 (§R-37 gate test BLOCKING CI)
- Nightly smoke test vs @mis PROVEN values (PRD-0173/0034/0181/0325/0009)

**Acceptance**: §R-37 gate test PASS · BTP yield test PASS · combo test PASS · @mis PROVEN values match ±0.5% · @mis + @cto engine-review session sign-off (re-confirm SPEC-001 checklist)

### Sprint 6 — Vue 3 frontend MVP
**Theme**: User-facing UI for Thy/Quỳnh/Ms.Hồng/KTT/Hào/CFO data stewards

**Deliverables**:
- Vue 3 + Vuetify 3 SPA scaffold
- Auth UI (login/logout/profile)
- 7-MDG list/detail/edit views (CRUD per RBAC)
- Excel V2 import wizard + diff review UI
- Audit log viewer (per record + global)
- Lock-conflict UI (holder + remaining TTL + "wait" / "contact")
- Costing engine "compute cost" diagnostic tool
- Anomaly dashboard (5-class + severity)

**Acceptance**: Stewards (Thy + Ms.Hồng + KTT + Hào volunteer testers) complete day-in-life CRUD tasks via UI · usability feedback collected · ≥ 1 happy-path screen recording per MDG · UX accessibility check (Vuetify defaults)

### Sprint 7+ — ON-DEMAND
**Theme**: Production deploy + Phase 2 features (triggered post MVP-prove signal)

Candidates (ON-DEMAND per Mental Model #9):
- Production deploy on S2 LXC + Telegram alert wire to NQH-Ops-Team
- Sentry integration
- Performance tuning (load test)
- VAT price-loop integration (per ADR-005 Stage 2 + REQ-017 VatDownload)
- BFlow 2 outbox consumer (Stage 3 sunset prep)
- Phase 2 online dashboard

## Acceptance gates per sprint (Definition of Done universal)

Each sprint MUST meet:
- [ ] All committed US PASS Gherkin acceptance
- [ ] All committed TT completed
- [ ] CI green (lint + test + Alembic dry-run + ≥ 80% coverage where feasible)
- [ ] Sprint review demo (Friday)
- [ ] @mis sprint-end scope check (anti L3 governance-spiral)
- [ ] @cto-CC sign-off
- [ ] Sprint retro filed at `04-build/SPRINT-{N}-RETRO-{date}.md`

## Per L4 drift circuit-breaker — outcome KPIs

| Sprint | KPI | Target |
|---|---|---|
| 1 | RBAC test pass rate | 100% (8/8 TC-RBAC) |
| 2-3 | MDG CRUD coverage | 7/7 MDGs working |
| 4 | Excel V2 round-trip fidelity | 100% structural equality |
| 5 | @mis PROVEN values match | ≥ 5/5 within ±0.5% |
| 6 | Steward UX completion | 4+ stewards complete tasks unaided |
| Post-MVP | Master-data corruption events/month | 0 (vs 60M-bug baseline) |

## Sprint dependency chain

```
Sprint 1 (Foundation + MDG-008)
    │
    ▼ depends on Foundation
Sprint 2 (MDG-002/003/004) ──┐
                              │
Sprint 3 (MDG-005/009/011/012) ─── parallel-OK if multi-coder, sequential if 1-coder
    │
    ▼ all 7 MDGs CRUD done
Sprint 4 (Excel V2)
    │
    ▼ Excel ingest tested
Sprint 5 (Costing engine)  ← engine-review checklist PASS gate already cleared at MVP start
    │
    ▼ engine produces @mis PROVEN values
Sprint 6 (Vue UI)
    │
    ▼ MVP DEMO
Sprint 7+ ON-DEMAND
```

## Em + CEO @architect oversight per sprint

- **Sprint planning** (Saturday): em + CEO joint review backlog, ratify sprint goal + US/TT scope
- **Mid-sprint check** (Wednesday async): em review PR drafts + answer Ms. Quỳnh design Q (if any — design specs should cover)
- **Sprint review** (Friday): em + CEO + @mis + @cto-CC demo + sign-off
- **Sprint retro** (Friday post-review): identify pattern adjustments for next sprint

## Memory anchors

- `project_weekly_cycle` (Sat→Fri NQH cycle)
- `feedback_pull_forward_preaudit_optimization` (pull-forward design done Sprint 0)
- `feedback_pattern_density_compounding_rate` (≥3 new patterns + ≥2 activations per sprint = healthy)
- `feedback_objective_met_defer` (Sprint 5 PROVEN values met = consider deferring Sprint 6 polish vs Sprint 7+ ON-DEMAND)

## References

- SPRINT-001 detail: `04-build/SPRINT-001-foundation-rbac-mdg008-thinslice.md`
- All 8 design docs `02-design/`
- Decision Log `785ffd07`
- SDLC 6.4.0 Sprint template
