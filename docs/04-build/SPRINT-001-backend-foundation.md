---
spec_id: SPRINT-001
spec_name: "Sprint 1 — Backend Foundation + 7-MDG CRUD + Minimal RBAC"
spec_version: "3.0.0"
status: planning
tier: LITE
stage: "04"
category: sprint-plan
owner: architect-itadmin-ceo-joint
created: 2026-06-28
last_updated: 2026-06-28
framework: SDLC 6.4.0
duration: 1 week (NQH Sat→Fri)
multi_agent: "Ms. Quỳnh (@fullstack LITE leader) + CC + Kimi parallel streams"
phase: MVP Phase 1 of 2 (per CEO 28/06 — full MDM = post-MVP-feedback Phase 2)
---

# Sprint 1 — Backend Foundation + 7-MDG CRUD + Minimal RBAC

## Sprint Goal

Backend ready for Sprint 2 frontend: 7-MDG CRUD via API + JWT auth + 2-role RBAC (steward + staff search) + basic audit log + staff search API + initial Excel V2 seed.

**NO** in Sprint 1: costing engine · full RBAC per-MDG · record-lock · approval workflow · anomaly detection. All Phase 2 ON-DEMAND.

## Schedule

| Attribute | Value |
|---|---|
| Sprint | 1 of 2 (MVP) |
| Duration | 1 week Sat→Fri |
| Conditional start | T+0 post COGS-PASS ✅ + ETA-Q4+ ✅ |
| Stretch target | 2026-07-15 (Wed) |
| Team | Ms. Quỳnh (@fullstack) + CC + Kimi parallel · em + CEO advisory · dtdanh DE on-demand if needed |
| Capacity | ~100 effective dev-hours (4 parallel streams × 5 days × ~5h productive — Quỳnh integration ~25% overhead) |

## Parallel Agent Streams

### Stream A — Foundation (CC, ~24h)
- Docker compose (FastAPI + PostgreSQL + Redis local dev) — TT-001
- pyproject.toml (FastAPI + SQLAlchemy + Pydantic v2 + alembic + redis + python-jose + openpyxl) — TT-002
- Alembic baseline 001 per SPEC-002 (subset — 7 MDG tables + mdm_user/role/user_role + basic mdm_audit_log; **SKIP lock_registry + full audit JSONB columns** = Phase 2)
- FastAPI scaffold + .env loading + healthcheck `/health` + Prometheus `/metrics`
- JWT auth `/auth/login`, `/auth/logout`, `/auth/me` (HS256 dev)
- EOW: service starts; alembic migrate works; login + token works

### Stream B — Minimal RBAC + Basic Audit (CC, ~16h)
- **2 roles only**: `ROLE_STEWARD` (CRUD 7 MDGs) + `ROLE_STAFF_SEARCH` (read-only search/lookup)
- RBAC middleware (FastAPI dependency)
- Basic audit log: timestamp + user_id + action (CREATE/READ/UPDATE/DELETE) + mdg_domain + record_id + ip
- **SKIP full before/after JSONB** (Phase 2)
- Test fixtures TC-RBAC-MVP-001-005 (subset of SPEC-004 §9 — 2-role scenarios)
- EOW: steward can CRUD; staff search returns read-only; audit captures writes

### Stream C — MDG-002/003/004/008 CRUD (CC, ~24h)
4 MDGs uniform pattern per simplified SPEC-003 §2.4:
- MDG-002 Supplier
- MDG-003 Product (+ `Ten_CukCuk_HoaDon` bridge for CukCuk paste)
- MDG-004 Site (+ `is_pc_authoritative` flag)
- MDG-008 NVL (+ `don_gia_sau_so_che` cột-K field)
- POST/GET/PUT/PATCH/DELETE (soft-delete) — NO approve endpoint Sprint 1 (direct steward edit per MVP)
- List pagination + filter DSL
- EOW: 4 MDGs full CRUD lifecycle

### Stream D — MDG-005/009/011/012 CRUD (Kimi, ~24h)
4 MDGs uniform pattern (replicate Stream C):
- MDG-005 BTP (+ `yield_qty` + `yield_uom` + `don_gia_sau_che_bien` — CRITICAL per @cto §C.4 + @mis rider #3 29/06; Excel-authoritative interim, engine yield-divide = VALIDATION pattern only)
- MDG-009 Customer (+ PDPL 2025 consent fields)
- MDG-011 Cost master
- MDG-012 Equipment (+ TSCĐ/CCDC enum)
- Recipe stub endpoints (read-only Sprint 1 — write Phase 2 if engine triggered)
- EOW: 4 MDGs full CRUD

### Stream E — Seed migrations + initial Excel import (Kimi, ~12h)
- Migration 002 seed RBAC (2 roles + initial users per CEO directive — Thy/Quỳnh/Hồng/KTT/Hào/CFO/dvhiep/staff_test)
- Migration 003 seed MDG-004 sites (BKL/TMM/Kupid/AFS/THOM/ADR1/ADR2 per project_bu_pc_structure)
- Migration 004 initial Excel V2 import script (`MASTER-DATA-V2.xlsx` all 7 MDGs read-only seed)
- GitHub Actions CI: ruff + pytest + Alembic dry-run on PR
- README contributor quickstart
- EOW: fresh DB → `alembic upgrade head` → fully seeded; CI green

### Stream F — Staff Search API (CC, ~8h)
- GET `/api/v1/search?mdg=*&q=keyword&filter=*` — read-only across 7 MDGs
- Full-text search on `ten_*` columns + filter by code/category/BU
- Pagination + ranking
- Copy-friendly response (returns `ma_*` + `ten_*` + key fields = paste-ready for CukCuk/Fast/Bflow 1)
- EOW: staff role can search across all MDGs

## Quỳnh @fullstack integration (~30h)
- Daily 09:00 PR triage (review Stream A-F PRs)
- Daily 14:00 merge (post CI green)
- Daily 17:00 standup post NQH-Ops-Team Telegram
- Wed mid-week: integration smoke test 7 MDGs CRUD + search end-to-end
- Friday: Sprint Review demo prep + execution

## Acceptance Gates

- [ ] 7 MDGs CRUD via API (steward role) lifecycle works (create → read → update → soft-delete)
- [ ] Staff search returns filtered MDG records (read-only, 2-role enforced)
- [ ] 2-role RBAC enforced (steward CRUD vs staff read-only) — TC-RBAC-MVP-001-005 PASS
- [ ] Basic audit log captures all writes
- [ ] JWT auth functional (login + token + me + logout)
- [ ] Initial Excel V2 seed populates all 7 MDGs
- [ ] CI green main (ruff + pytest + Alembic dry-run)
- [ ] Sprint Review demo: CRUD on 2 MDGs + search 1 MDG + RBAC denial
- [ ] @cto MDM (dvhiep) + @mis sign-off → Sprint 2 unblocked

## Out of scope Sprint 1 (Phase 2 ON-DEMAND)

- ❌ Costing engine (SPEC-006 — Phase 2)
- ❌ Full RBAC per-MDG 18 roles (SPEC-004 full — Phase 2)
- ❌ Record-lock (ADR-002 §2.5 — Phase 2)
- ❌ Approval workflow (Phase 2)
- ❌ Full audit JSONB before/after (Phase 2)
- ❌ Anomaly detection (Phase 2)
- ❌ Excel V2 round-trip UI (Sprint 2)
- ❌ Vue 3 frontend (Sprint 2)

## CEO doctrine context (28/06)

- **Bflow 1/2 native access**: if MDM live, Bflow 1 + Bflow 2 can both query MDM directly (chúng ta phát triển = NQH-controlled API access). NO Excel-paste workaround needed for Bflow.
- **CukCuk + Fast Accounting Online**: external systems = MUST nhập liệu thủ công OR import qua Excel file (KHÔNG có direct API integration). MDM Phase 1 staff search UI = paste source for these 2 systems specifically.
- → Sprint 2 staff search UI = optimized cho **CukCuk + Fast Accounting Online paste flow** (copy ma_* + ten_* + key fields format-friendly).

## Risks + Mitigations

| Risk | Mitigation |
|---|---|
| Kimi LOGIC drift on Stream D MDG CRUD | Quỳnh attentive review; first MDG in Stream D = pair-review vs CC Stream C reference |
| Alembic migration conflicts cross-stream | Stream E (Kimi) OWNS all migrations; Streams A-D contribute via PR to Stream E branch |
| Staff search query performance | Phase 1 = simple ILIKE; indexes on `ten_*` columns; optimize Phase 2 if needed |
| MVP scope creep | em + CEO veto any Phase 2 feature attempts; Phase 2 trigger = post-MVP-feedback only |

## Sprint Ceremonies

| Day | Event | Duration |
|---|---|---|
| Saturday 09:00 | Sprint Planning kickoff | 1h |
| Mon-Thu 17:00 | Daily standup async (Telegram) | 5min |
| Friday 09:00 | Final stream merge + CI green | 30min |
| Friday 14:00 | Sprint Review + Demo | 30min |
| Friday 14:30 | Sprint Retro | 15min |

## Dependencies (BLOCKERS for Sprint 1 start)

| Dependency | Owner | Status |
|---|---|---|
| COGS-PASS spot-check ✅ | dtdanh | ⏳ |
| ETA-Q4+ confirm | CEO | ⏳ |
| .env vault values populated | em + CIO Hiệp | ⏳ |
| PostgreSQL `mdm` DB on S2 LXC 210 | dtdanh DE | ⏳ |
| Redis DB 10 reservation | dtdanh DE | ⏳ |
| Quỳnh + CC + Kimi agent access | em coordinate | ⏳ |

## References

- ROADMAP-001 v3: `04-build/ROADMAP-001-mvp-lean-sprints.md`
- SPRINT-002: `04-build/SPRINT-002-excel-vue-demo.md`
- Design docs `02-design/` (Phase 1 subsets per ROADMAP §"Design docs status per phase")
- Decision Log `785ffd07`
- CEO directives 28/06 (multi-SE4A · 2-phase · CukCuk/Fast Excel paste flow · Bflow native access)
