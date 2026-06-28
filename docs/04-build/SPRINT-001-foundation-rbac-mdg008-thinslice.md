---
spec_id: SPRINT-001
spec_name: "Sprint 1 — Foundation + RBAC + MDG-008 NVL Thin-Slice"
spec_version: "1.0.0"
status: planning
tier: LITE
stage: "04"
category: sprint-plan
owner: architect-itadmin-ceo-joint
created: 2026-06-28
last_updated: 2026-06-28
framework: SDLC 6.4.0
duration: 1 week (NQH Sat→Fri per project_weekly_cycle)
sprint_format: per Planning-Hierarchy-SPRINT-TEMPLATE
---

# Sprint 1 — Foundation + RBAC + MDG-008 NVL Thin-Slice

## Sprint Schedule

| Attribute | Value |
|---|---|
| Sprint Number | 1 of N (MVP roadmap 6 sprints) |
| Duration | 1 week (Sat → Fri, NQH weekly cycle) |
| **Conditional start** | T+0 days post: COGS-PASS ✅ AND engine-review checklist PASS ✅ AND ETA-Q4+ ✅ (per Decision Log `785ffd07`) |
| **Stretch target start** | 2026-07-15 (Wednesday, allow ramp-up after BFlow 2 ETA review) — if conditional gates clear earlier, start earlier |
| Team Size | 1 (Ms. Quỳnh @coder) + advisory (em @itadmin + CEO @architect joint, NOT design participant for Ms. Quỳnh) |
| Capacity | ~30 hrs effective (1 person × 5 days × 6 productive hrs) |
| Sprint Goal | Foundation infrastructure + RBAC enforced + MDG-008 NVL CRUD vertical slice (thin-slice prove pipeline end-to-end) |

## Sprint Goal

**Prove pipeline end-to-end on 1 vertical slice** (MDG-008 NVL CRUD) — Foundation infrastructure (DB + Auth + RBAC + Audit) PLUS 1 working MDG = de-risk full 7-MDG buildout before scaling.

Per @mis L2 evidence-of-need: avoid Sprint 2-6 spending without Sprint 1 thin-slice PROVEN.

## Committed Work

### User Stories

| ID | Story | Pts | Acceptance Gherkin | Status |
|----|-------|-----|---|---|
| US-001 | Ms. Quỳnh applies SPEC-002 baseline DDL (Alembic 001) + 002_seed_rbac.sql | 3 | GIVEN clean PostgreSQL `mdm` database WHEN `alembic upgrade head` THEN all 18 tables created per SPEC-002 §2-14; seed has 14 users + 18 roles per SPEC-004 §2 | TODO |
| US-002 | Bootstrap FastAPI scaffold + .env loading + healthcheck endpoint | 2 | GIVEN .env populated per .env.example WHEN service starts THEN `GET /api/v1/health` returns 200 + `GET /api/v1/metrics` exposes Prometheus | TODO |
| US-003 | JWT auth + login/logout/me endpoints (SPEC-003 §2.1) | 5 | GIVEN seeded user `dvhiep` WHEN POST `/auth/login` with credentials THEN JWT returned (HS256 dev); `GET /auth/me` returns user + roles; `POST /auth/logout` revokes | TODO |
| US-004 | RBAC middleware enforcing role-required per endpoint (ADR-002 §2.4 + SPEC-004 §9) | 5 | GIVEN user WITHOUT ROLE_STEWARD_MDG-008 WHEN POST `/api/v1/mdg-008` THEN HTTP 403; audit_log row 'CREATE attempted, RBAC_DENY'. Test cases TC-RBAC-001 to TC-RBAC-008 from SPEC-004 §9 all PASS | TODO |
| US-005 | Record-lock implementation (Redis SETNX + heartbeat + force-release per ADR-002 §2.5) | 5 | GIVEN steward A acquires lock WHEN steward B attempts same record THEN HTTP 423 LOCKED with holder info + remaining TTL; CIO force-release works + audit-logged | TODO |
| US-006 | Audit log append-only (SPEC-002 §3 + REVOKE UPDATE/DELETE enforcement) | 2 | GIVEN any CRUD action WHEN persisted THEN mdm_audit_log row written; attempt UPDATE or DELETE on audit table → HTTP 405 or PostgreSQL permission error | TODO |
| US-007 | MDG-008 NVL CRUD endpoints full lifecycle (SPEC-003 §2.4 + SPEC-002 §9) | 8 | GIVEN steward seeded WHEN POST/GET/PUT/DELETE/APPROVE per SPEC-003 pattern THEN happy-path works + validation errors blocked + audit-logged + state DRAFT→APPROVED on approver action | TODO |

**Total Points**: 30 / 30 capacity (1 person-week)

### Technical Tasks

| ID | Task | Est | Assignee | Status |
|----|------|-----|----------|--------|
| TT-001 | Docker compose stack (FastAPI + PostgreSQL + Redis local dev) | 2h | Quỳnh | TODO |
| TT-002 | requirements.txt + pyproject.toml (FastAPI + SQLAlchemy + Pydantic + alembic + redis + python-jose + openpyxl) | 1h | Quỳnh | TODO |
| TT-003 | Alembic init + migration 001_baseline.sql per SPEC-002 (full DDL) | 4h | Quỳnh | TODO |
| TT-004 | Seed migrations 002_seed_rbac + 003_seed_sites per SPEC-002 §15 | 2h | Quỳnh | TODO |
| TT-005 | Pytest setup + fixture conftest.py + first 3 test files | 3h | Quỳnh | TODO |
| TT-006 | GitHub Actions CI: lint (ruff) + test (pytest) + Alembic dry-run on PR | 2h | Quỳnh | TODO |
| TT-007 | README contributor guide + local-dev quickstart | 1h | Quỳnh | TODO |

### Out of scope Sprint 1

- ❌ MDG-002/003/004/005/009/011/012 CRUD (Sprint 2-3)
- ❌ Excel V2 import/export (Sprint 4)
- ❌ Costing engine (Sprint 5)
- ❌ Vue frontend (Sprint 6 — backend-first MVP)
- ❌ Production deploy (Sprint 7+ post engine-review PASS)

## Acceptance Gates (Definition of Done for Sprint 1)

- [ ] All 7 US PASS acceptance Gherkin (Ms. Quỳnh self-test + em spot-check)
- [ ] All 7 TT completed + committed
- [ ] CI green on main branch
- [ ] Audit log append-only verified empirically (DELETE attempt rejected)
- [ ] RBAC middleware enforces all 8 TC-RBAC-* test cases (SPEC-004 §9)
- [ ] Record-lock concurrency test passes (2 stewards same record → 1 wins)
- [ ] MDG-008 NVL vertical slice demoed: create draft → approve → list → update → soft delete
- [ ] Audit log captures all actions with before/after JSON state
- [ ] @mis sprint review session: scope completion + L3 anti-spiral check (gov:code ratio acceptable for foundation sprint)
- [ ] @cto-CC sprint sign-off

## Risks + Mitigations

| Risk | Likelihood | Mitigation |
|---|---|---|
| Alembic migration drift between local + S2 LXC | Medium | Migration applied first via dry-run on S2 LXC clone; production = exact same migration file |
| RBAC complexity vs 1-week capacity | Medium | Scope reduced to MDG-008 ONLY (1 MDG vertical); other 6 MDGs follow same pattern Sprint 2-3 |
| Redis lock TTL tuning issues | Low | Default 15min + 5min heartbeat per ADR-002 §2.5; observe real-world Sprint 2-3 + adjust |
| Ms. Quỳnh ramp-up on FastAPI + SDLC LITE | Medium | Em + CEO available for Q&A; pair-debug encouraged early; design specs comprehensive (8 docs at 02-design covering full surface) |
| §R-37 gate test authoring complexity | Low | LOGIC-SPEC + fixture SQL ready (SPEC-006 §6.1 + SPEC-007 §2) — Ms. Quỳnh code-converts, no spec ambiguity |

## Sprint Ceremonies (NQH weekly cycle Sat→Fri)

| Day | Event | Duration | Participants |
|---|---|---|---|
| Saturday | Sprint Planning (kickoff) | 1h | Ms. Quỳnh + em + CEO |
| Mon-Thu daily | Standup async (Telegram NQH-Ops-Team) | 5min | Ms. Quỳnh post · em + CEO observe |
| Friday | Sprint Review + Demo | 30min | + @mis + @cto-CC |
| Friday | Sprint Retro | 15min | Same |

## Dependencies (BLOCKERS for Sprint 1 start)

| Dependency | Owner | Status |
|---|---|---|
| COGS-PASS spot-check ✅ | dtdanh (separate stream) | ⏳ |
| Engine-review checklist PASS ✅ | @mis ratify SPEC-001 + session | ⏳ |
| ETA-Q4+ confirm | CEO (BFlow 2 trajectory review) | ⏳ |
| .env vault values | em + CIO Hiệp | ⏳ |
| PostgreSQL `mdm` database created on S2 LXC 210 | dtdanh | ⏳ |
| Redis DB 10 reservation | dtdanh | ⏳ |

## References

- SDLC Sprint Template: `/home/nqh/shared/.sdlc-framework/05-Templates-Tools/08-Project-Templates/Planning-Hierarchy-SPRINT-TEMPLATE.md`
- ROADMAP-001: `04-build/ROADMAP-001-mvp-lean-sprints.md`
- All 8 design docs at `02-design/` (Sprint 1 implements SPEC-002 + ADR-002 + SPEC-003 §2.1-2.4 subset + SPEC-004 §9 RBAC tests)
- Decision Log `785ffd07`: build-sequence gate
- Memory: `project_weekly_cycle` (Sat→Fri) · `feedback_pull_forward_preaudit_optimization`
