---
spec_id: REQ-001
spec_name: "MDM MVP-Lean — Product Requirements"
spec_version: "1.0.0"
status: draft
tier: LITE
stage: "01"
category: master-data
owner: itadmin-orchestrator
created: 2026-06-28
last_updated: 2026-06-28
---

# REQ-001 — MDM MVP-Lean Product Requirements

**Authority**: @mis Decision Log `785ffd07` + VISION-001

## 1. Overview

7-MDG CRUD application with RBAC + record-lock + validation + Excel V2 round-trip + durable costing-engine. BUILD trigger gated on COGS-PASS + engine-review PASS + ETA-Q4+ confirm.

## 2. Functional Requirements

#### FR-001: 7-MDG CRUD coverage
**Priority**: P0
**Tier**: LITE

```gherkin
GIVEN authorized data steward logs into MDM web UI
WHEN they navigate to any of 7 MDG domains
THEN they can Create/Read/Update/Delete records for:
  AND MDG-002 Supplier (NCC), owner=Ms.Hồng
  AND MDG-003 Product (Mã món PRD-xxxx), owner=Thy
  AND MDG-004 Site (BU/branch), owner=Ms.Hồng
  AND MDG-005 Customer (khách lẻ + nhóm), owner=Thy
  AND MDG-008 Food (NVL RAW/PRE/INT/CON), owner=Thy
  AND MDG-011 Cost (cost master), owner=KTT
  AND MDG-012 Equipment (TSCĐ + CCDC), owner=Hào
  AND every CRUD action logs to audit trail
```

#### FR-002: RBAC per-domain owner
**Priority**: P0
**Tier**: LITE

```gherkin
GIVEN user role mapped to MDG domain owner (per .env RBAC_OWNER_MDG*)
WHEN user attempts CRUD on a MDG domain
THEN access granted if user IS designated owner of that domain
  AND access DENIED with HTTP 403 if user is NOT owner
  AND read-only view granted for non-owners (consistency + transparency)
  AND admin role (CIO Hiệp) can override per-domain restrictions with audit log
```

#### FR-003: Record-lock (kill concurrent corruption)
**Priority**: P0
**Tier**: LITE

```gherkin
GIVEN user A opens a record for edit
WHEN user B attempts to open the same record for edit
THEN user B sees "Record locked by user A, last activity HH:MM:SS"
  AND user B has read-only view + can wait
  AND lock auto-expires after 15 min idle (configurable LOCK_TIMEOUT_SEC)
  AND user A's commit releases lock
  AND lock store = Redis (REDIS_DB=10) with key pattern `mdm:lock:{mdg}:{record_id}`
```

#### FR-004: Validation + error-detection
**Priority**: P0
**Tier**: LITE

```gherkin
GIVEN user submits create/update form
WHEN MDM validates before commit
THEN required fields MUST be present (ma_*, ten_*, owner-defined required)
  AND foreign key references MUST resolve (e.g. Product.ma_ncc → exists in Supplier)
  AND unit-conversion fields validated (per cột-K validation per @mis Decision Log)
  AND duplicate detection (warn if similar exists)
  AND validation errors displayed inline with field-level red highlight
  AND commit BLOCKED on validation failure
```

#### FR-005: Excel V2 round-trip (Thy/Quỳnh/Ms.Hồng workflow preserved)
**Priority**: P0
**Tier**: LITE

```gherkin
GIVEN data steward prefers Excel editing workflow OR MDM web UI
WHEN they import Excel V2 file (born-clean format per MDG schema)
THEN MDM parses + validates each row
  AND surfaces validation errors per row (preview before commit)
  AND commits valid rows + reports invalid rows for fix
  AND export round-trip produces Excel V2 file matching Thy/Quỳnh format (born-clean compatible)
  AND data stewards can edit in Excel OR MDM UI interchangeably (no lock-in)
```

#### FR-006: Durable costing-engine (sunset-portable)
**Priority**: P0
**Tier**: LITE

```gherkin
GIVEN MDM hosts costing-engine for COGS/món + bill computation
WHEN master-data changes (NVL price, recipe, yield)
THEN engine recomputes on-demand OR per scheduled trigger (NOT realtime online — MVP defer per Decision Log)
  AND engine logic written for portability (per ADR-005 of clickhouse/docs §E criteria)
  AND engine consumes from MDM write tables + ClickHouse read views
  AND engine output feeds dashboard 22/19 + @bod queries (via ClickHouse dim_* updates)
  AND engine port path to BFlow 2 = 0-waste at sunset (logical model preserved)
```

#### FR-007: Audit trail
**Priority**: P0
**Tier**: LITE

```gherkin
GIVEN any CRUD action commits in MDM
WHEN action persists to PostgreSQL
THEN audit row written to `mdm_audit_log` with:
  AND timestamp · user_id · action (C/R/U/D) · mdg_domain · record_id · before_state · after_state · ip_address
  AND audit table append-only (no deletes, no updates)
  AND retention ≥ 7 years (per accounting compliance)
```

#### FR-008: Excel V2 daily sync (poll + snapshot)
**Priority**: P1
**Tier**: LITE

```gherkin
GIVEN Drive Excel V2 (1K-SL4ae...) is sync source
WHEN MDM daily cron runs (06:00 ICT) OR Drive event-hook fires
THEN MDM pulls latest Excel V2 → compares vs MDM PostgreSQL state
  AND surfaces diffs as "pending sync" review queue for data stewards
  AND data stewards review + accept/reject each diff (audit-trailed)
  AND accepted diffs commit to MDM write tables
  AND rejected diffs noted for Thy/Quỳnh fix at source (Excel V2)
```

## 3. Non-Functional Requirements

| Metric | Target | Measurement |
|--------|--------|-------------|
| Concurrent users | ≥ 10 simultaneous edits across MDGs | Load test pre-deploy |
| Record-lock latency | ≤ 100ms acquire / release | Redis ops timing |
| CRUD latency p95 | ≤ 500ms | Server logs |
| Excel V2 import 1000 rows | ≤ 60 sec | Import benchmark |
| Audit retention | ≥ 7 years | Schema constraint |
| Uptime | ≥ 99% (S2 LXC reliability) | Uptime-kuma monitor |
| RBAC enforcement | 100% (zero unauthorized writes) | Pen-test post-MVP |
| Master-data corruption events | 0 / month post-deploy | Monthly audit query |

## 4. Acceptance Criteria

- [ ] All 7 MDG domains support CRUD via web UI
- [ ] RBAC enforced per-domain (denied users see 403)
- [ ] Record-lock prevents concurrent corruption (live test with 2 users)
- [ ] Validation blocks bad-data commit (live test with synthetic errors)
- [ ] Excel V2 import + export round-trip (Thy/Quỳnh sign-off)
- [ ] Costing-engine produces COGS/món matching @mis ground-truth (PRD-0173/0034/0181/0325/0009)
- [ ] Audit trail captures all CRUD actions (verified via query)
- [ ] Daily Excel sync surfaces diffs (live test with edit at source)
- [ ] Engine-review checklist PASS (gate before MVP build)
- [ ] @mis governance sign-off on acceptance evidence

## 5. Out of scope MVP (ON-DEMAND or CARVE per Decision Log)

- ❌ Online realtime dashboard (Phase 2 ON-DEMAND)
- ❌ MDG-009 Customer online (Phase 2 ON-DEMAND)
- ❌ VAT price-loop auto (Phase 3 ON-DEMAND, see VatDownload REQ-017)
- ❌ Cash-consolidation PROCESS (CARVE → BFlow 2)
- ❌ Virtual-customer-AR (CARVE → BFlow 2)
- ❌ Replace CukCuk sales (non-goal)
- ❌ Replace VatDownload HĐ thuế (non-goal)

## 6. References

- VISION-001: `00-foundation/00-vision-mdm.md`
- Decision Log `785ffd07`: `core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Operations/DECISION-MDM-BUILD-WHERE-28062026.md`
- @mis MDM-WEBAPP-SPEC handoff: same dir/HANDOFF-ITADMIN-MDM-WEBAPP-SPEC-28062026.md
- ADR-001 stack choice: `02-design/ADR-001-stack-choice.md`
- SPEC engine-review checklist: `02-design/SPEC-001-engine-review-checklist.md` (em draft post @mis green-light)
- COGS pipeline foundation: `apps/clickhouse/docs/` (ADR-001/003/004/005 + SPEC-0004)
- Excel V2 SSOT: `core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Operations/MASTER-DATA-V2.xlsx`
