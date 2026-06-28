---
spec_id: ROADMAP-001
spec_name: "MVP + Phase 2 Roadmap (2 phases per CEO 28/06)"
spec_version: "3.0.0"
status: planning
tier: LITE
stage: "04"
category: roadmap
owner: architect-itadmin-ceo-joint
created: 2026-06-28
last_updated: 2026-06-28
sprint_duration: 1 week (Sat→Fri NQH cycle)
mvp_total: 2 weeks (Sprint 1 + Sprint 2)
phase2_trigger: post-MVP user feedback (Mental Model #9 demand-before-surface)
---

# ROADMAP-001 v3 — 2-Phase MVP

**Authority**: Decision Log `785ffd07` + CEO directives 28/06 (compressed scope per multi-SE4A + 2-phase split per user-feedback gate)

## CEO 28/06 directives folded

1. "2 tuần tối đa với multi SE4A agents CC và Kimi" → Sprint 1 + Sprint 2 = 2 weeks
2. "Web interface cho quản trị template V2 + công cụ tra cứu MD cho NV để paste CukCuk/Fast/Bflow 1" → MVP scope narrowed
3. "Chia làm 2 giai đoạn — full MDM sau khi chạy MVP và có phản hồi từ người dùng" → 2-phase split
4. "Quỳnh = @fullstack (LITE consolidate) cùng CC + Kimi (SE4A); CEO = @architect + SE4H; Danh DE on-demand" → role doctrine

## 2-Phase split

### MVP Phase 1 (Sprint 1 + Sprint 2 = 2 weeks)

**Goal**: Web UI replacing raw-Excel pain for Thy/Quỳnh/Hồng (stewards) + search/lookup tool for NV (staff paste into CukCuk/Fast/Bflow 1).

**Scope (narrow, MM#9 demand-before-surface)**:
- ✅ Web admin UI for 7-MDG CRUD (steward role — Thy/Quỳnh/Hồng/KTT/Bếp trưởng/Hào/CFO)
- ✅ Excel V2 round-trip import/export (preserve steward workflow)
- ✅ Staff search/lookup UI (read-only + filter + copy-to-clipboard)
- ✅ Minimal RBAC (2 roles: `ROLE_STEWARD`, `ROLE_STAFF_SEARCH`)
- ✅ Basic audit log (timestamp + user + action — NO full before/after JSONB)
- ✅ PostgreSQL backend + Vue 3 frontend (per ADR-001 stack)
- ❌ NO costing engine (deferred Phase 2)
- ❌ NO full RBAC per-MDG (deferred Phase 2)
- ❌ NO record-lock (Phase 1 = optimistic concurrency + "last write wins" + audit visibility)
- ❌ NO approval workflow (Phase 1 = direct steward edit)
- ❌ NO anomaly detection (Phase 2)
- ❌ NO multi-source price priority (Phase 2)

### Phase 2 (post-MVP user feedback)

**Trigger** (Mental Model #9 re-eval): MVP DEMO PASS + ≥2 weeks production use + user feedback collected + CEO/CFO ratify Phase 2 scope per actual pain identified.

**Candidate scope** (NOT pre-committed; per feedback signal):
- 🟡 Costing engine LOGIC-SPEC impl per SPEC-006 (RATIFIED 28/06) — if value-truth needed beyond CukCuk/Fast/Bflow 1 paste flow
- 🟡 Full RBAC per-MDG (SPEC-004 18 roles) — if per-domain authority enforcement needed
- 🟡 Record-lock per ADR-002 §2.5 — if concurrent corruption recurs in MVP
- 🟡 Approval workflow (steward draft → KTT approve) — if accountability/audit drives demand
- 🟡 Anomaly detection (5-class) + Telegram WI-105 routing — if data quality drift detected
- 🟡 Multi-source price priority + VAT integration — if pricing accuracy gap surfaces
- 🟡 BFlow 2 sunset prep (Phase 3 carve)

Per L1 govern-proportional: Phase 2 scope decided per @mis trigger + user feedback evidence, NOT pre-planned now.

## Sprint schedule (MVP)

| Sprint | Theme | Duration | Conditional start | Stretch target |
|---|---|---|---|---|
| **1** | Backend + 7-MDG CRUD + minimal RBAC + auth | 1 week | T+0 post gates | 2026-07-15 (Wed) |
| **2** | Excel V2 round-trip + Vue UI (steward admin + staff search) + DEMO | 1 week | T+7 days | 2026-07-22 |

**Conditional gate**: COGS-PASS ✅ (separate stream) + ETA-Q4+ ✅ per Decision Log. **Engine-review session = ON-DEMAND** (deferred Phase 2 trigger).

## SE4A agent allocation

| Person/Agent | SE4A/SE4H | Role | Stream assignment |
|---|---|---|---|
| **CEO Tài** | SE4H | @architect + design authority | Final ratify · stretch goals · scope arbitration |
| **em (@itadmin)** | SE4A | @architect (joint with CEO) | Daily standup observe · respond design Q · spec clarification |
| **Ms. Quỳnh** | **SE4A leader** | **@fullstack** (LITE = coder + tester + devops merged) | Daily PR review + integrate + acceptance + demo prep + deploy ops |
| **CC** (Claude Code) | SE4A | @coder agent stream | Complex logic · LOGIC-SPEC interpretation · test authoring · backend |
| **Kimi** | SE4A | @coder agent stream | Bulk mechanical (DDL→ORM · Excel parsing · Vue scaffolding) |
| **dtdanh** | SE4A | **DE (on-demand)** | DWH coord if Phase 2 engine reads/writes ClickHouse; NOT MVP scope |

## Sprint 1 — Backend foundation + 7-MDG CRUD + minimal RBAC

**Detail spec**: `SPRINT-001-backend-foundation.md` (renamed v3)

**Goal**: Backend complete enough that Sprint 2 frontend can consume. 7-MDG CRUD via API + JWT auth + 2-role RBAC + basic audit log.

### Stream allocation (Sprint 1)

| Stream | Agent | Scope (Phase 1 narrow) |
|---|---|---|
| A. Foundation | CC | Docker + FastAPI + PostgreSQL + Redis + Alembic baseline (subset of SPEC-002) + JWT auth |
| B. Minimal RBAC + basic audit | CC | 2-role middleware (ROLE_STEWARD vs ROLE_STAFF_SEARCH) + basic audit log table + RBAC tests |
| C. MDG-002/003/004/008 CRUD | CC | 4 MDGs uniform CRUD pattern (Supplier/Product/Site/NVL) per simplified SPEC-003 |
| D. MDG-005/009/011/012 CRUD | Kimi | 4 MDGs uniform pattern (BTP/Customer/Cost/Equipment) |
| E. Seed migrations + initial Excel import | Kimi | Alembic seed + initial MASTER-DATA-V2.xlsx import |
| F. Staff search API | CC | Read-only filter + paginate + full-text search across MDG (for staff lookup) |

### Sprint 1 acceptance

- [ ] 7 MDGs full CRUD via API (steward role)
- [ ] Staff search returns filtered MDG records (read-only)
- [ ] 2-role RBAC enforced (steward CRUD vs staff read-only)
- [ ] Basic audit log captures all writes (timestamp + user + action)
- [ ] JWT auth functional
- [ ] Initial Excel V2 seeded
- [ ] CI green

## Sprint 2 — Excel V2 + Vue UI + DEMO

**Detail spec**: `SPRINT-002-excel-vue-demo.md` (renamed v3)

**Goal**: Stewards manage MD via web (vs raw Excel) · staff lookup MD codes via web · MVP DEMO ready.

### Stream allocation (Sprint 2)

| Stream | Agent | Scope |
|---|---|---|
| G. Excel V2 round-trip | CC | SPEC-005 implementation: import + diff review + commit + export born-clean V2 |
| H. Vue 3 steward admin UI | Kimi | Vue 3 + Vuetify 3 SPA scaffold + 7-MDG CRUD forms + Excel import wizard + diff review UI |
| I. Vue 3 staff search UI | Kimi | Read-only search/filter table + copy-to-clipboard for CukCuk/Fast/Bflow 1 paste |
| J. Drive integration | CC | Service account auth + daily 06:00 poll for new V2 versions |
| K. E2E + DEMO prep | Quỳnh | Steward UAT (Thy + Hồng) + staff UAT (volunteer NV) + demo script + screen recordings |
| L. Basic ops + deploy | Quỳnh | Docker compose deploy on S2 LXC + nginx-proxy-manager route + Telegram alert wire |

### Sprint 2 acceptance

- [ ] Excel V2 import → 7 MDGs populated via UI
- [ ] Excel V2 export round-trip bit-equal (modulo timestamps)
- [ ] Stewards (Thy + Hồng + 1 other) complete day-in-life CRUD tasks via UI unaided
- [ ] Staff (≥1 NV volunteer) finds MD codes via search + copies to CukCuk/Fast/Bflow 1
- [ ] MDM deployed on S2 LXC accessible via internal URL
- [ ] **MVP DEMO** Friday Sprint 2 — CEO + @cto-CC + @mis sign-off
- [ ] User feedback form distributed (3-question: easier? faster? missing?)

## Phase 2 trigger criteria (post-MVP)

Em propose explicit criteria for Phase 2 unlock (per L4 drift circuit-breaker outcome-not-volume):

- ≥2 weeks production use of MVP
- User feedback collected from ≥3 stewards + ≥3 staff NV
- @mis governance review: which Phase 2 candidates have demand-signal? (engine? full RBAC? lock? approval?)
- CEO ratify Phase 2 scope per feedback (NOT auto-build per em earlier 8-doc design)

If user feedback = "MVP enough for now" → Phase 2 deferred indefinitely (em respect MM#9 demand-before-surface).

## Per-day execution rhythm (each sprint)

| Day | Focus |
|---|---|
| Saturday | Sprint Planning kickoff 1h (em + CEO + Quỳnh confirm streams) |
| Mon-Thu | Parallel stream execution · Quỳnh daily PR review (morning) + merge (afternoon) · em respond Q (rare) |
| Friday 09:00 | Stream merge final + CI green |
| Friday 14:00 | Sprint Review demo 30min |
| Friday 14:30 | Sprint Retro 15min |

## Design docs status per phase

| Doc | Phase | Status |
|---|---|---|
| ADR-001 stack | MVP | ✅ ratified |
| ADR-002 RBAC + lock | Phase 2 | 🟡 simplified subset for MVP (2 roles + no lock) |
| SPEC-001 engine-review | Phase 2 | ✅ RATIFIED 28/06 (engine = Phase 2) |
| SPEC-002 PostgreSQL DDL | MVP (subset) | 🟡 7 MDG tables + basic audit; lock_registry + full audit JSONB = Phase 2 |
| SPEC-003 OpenAPI | MVP (subset) | 🟡 CRUD + auth + search; engine + anomaly + full RBAC = Phase 2 |
| SPEC-004 RBAC matrix | Phase 2 | 🟡 MVP = 2 roles (steward + staff); full 18-role matrix = Phase 2 |
| SPEC-005 Excel V2 | MVP | ✅ full scope for MVP |
| SPEC-006 Engine LOGIC-SPEC | Phase 2 | ✅ ratified, defer impl |
| SPEC-007 Test fixtures | Both | 🟡 split: MVP CRUD/RBAC/Excel fixtures + Phase 2 engine/lock fixtures |

## L4 drift circuit-breaker — outcome KPIs

| Metric | Target | Phase |
|---|---|---|
| MVP Backend 7-MDG CRUD | 7/7 working | Sprint 1 |
| MVP Excel V2 round-trip | identical bit-fidelity | Sprint 2 |
| MVP Steward UX | ≥3 stewards complete unaided | Sprint 2 |
| MVP Staff search UX | ≥3 NV find + copy MD codes unaided | Sprint 2 |
| MVP DEMO sign-off | CEO + @cto + @mis approve | EOW Sprint 2 |
| Phase 2 trigger | User feedback ≥3+3 + Phase 2 candidate ratify | post-MVP |
| Engine impl (Phase 2) | @mis PROVEN values match ±0.5% | post-MVP if engine triggered |

## Sprint 3+ ON-DEMAND (per Phase 2 trigger)

Triggered per Phase 2 scope ratify. Em NOT pre-plan to avoid L3 governance-spiral.

## References

- SPRINT-001 detail: `04-build/SPRINT-001-backend-foundation.md`
- SPRINT-002 detail: `04-build/SPRINT-002-excel-vue-demo.md`
- All design docs `02-design/` (Phase 1 subset + Phase 2 full)
- Decision Log `785ffd07`
- SPEC-001 RATIFIED 28/06 (engine = Phase 2)
- Memory: `feedback_objective_met_defer` · MM#9 demand-before-surface
