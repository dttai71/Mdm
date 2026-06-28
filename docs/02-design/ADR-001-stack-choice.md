---
spec_id: ADR-001
spec_name: "MDM Tech Stack Choice (FastAPI + Vue 3 + PostgreSQL + Redis)"
spec_version: "1.0.0"
status: draft
tier: LITE
stage: "02"
category: architecture
owner: architect-itadmin
created: 2026-06-28
last_updated: 2026-06-28
---

# ADR-001 — MDM Tech Stack Choice

## 1. Context

MDM MVP-lean per REQ-001 needs:
- Web UI (CRUD forms) for 7 MDG domains — data stewards Thy/Quỳnh/Ms.Hồng/KTT/Hào
- Transactional write store (master-data integrity + audit)
- Distributed lock store (record-lock for concurrent edit)
- ClickHouse read integration (existing dim_* tables + costing-engine)
- Excel V2 round-trip (openpyxl or pandas)
- Telegram alerts (NQH-Ops-Team channel)
- Portable to BFlow 2 at sunset (logical model preserved)

## 2. Decision

**Stack (post-@architect ratify, em propose default)**:

```yaml
Backend:
  Language: Python 3.12+
  Framework: FastAPI (OpenAPI built-in, async support, type hints)
  ORM: SQLAlchemy 2.0 + Alembic (migration discipline)
  Validation: Pydantic v2 (per Decision Log validation FR-004)

Write store:
  Engine: PostgreSQL 16 (existing S2 LXC 210 instance)
  Schema: mdm (dedicated database, not co-tenant with bflow)
  Migrations: Alembic versioned

Read integration:
  ClickHouse: existing NQH instance (192.168.2.10:8123)
  ETL: nightly cron + on-demand engine triggers (per ADR-005 portability)

Cache + Lock:
  Redis: existing S2 LXC 210 instance (REDIS_DB=10 MDM dedicated)
  Pattern: `mdm:lock:{mdg}:{record_id}` with 15-min TTL

Frontend:
  Framework: Vue 3 + Composition API
  UI: Vuetify 3 (form-heavy, table-heavy = Vuetify strength)
  State: Pinia
  Build: Vite

Auth:
  Bearer JWT via FastAPI (HS256 dev, RS256 prod)
  Integration with existing OpenWebUI auth OR LDAP (TBD post-architect review)

DevOps:
  Container: Docker Compose (S2 LXC alongside VatDownload + ClickHouse)
  CI: GitHub Actions
  Logging: stdout → systemd journald → log aggregation (optional Sentry post-MVP)
  Monitoring: uptime-kuma (HTTP probe), Telegram alerts via /v1/send

Excel V2:
  Library: openpyxl (read+write Excel V2 born-clean format)
  Drive sync: google-api-python-client + service account (existing nqh-sheet-reader.json pattern)
```

## 3. Rationale

### Why Python + FastAPI (not Node/Java/Go)
- Existing NQH stack: CukCuk scraper + VatDownload + ETL = Python (em prior precedents)
- dtdanh fluency Python (em verified prior sessions)
- openpyxl mature for Excel V2 manipulation
- FastAPI OpenAPI built-in → easy frontend Vue auto-typed via OpenAPI generator
- async I/O for Drive poll + ClickHouse query parallelism

### Why PostgreSQL (not MySQL/MongoDB)
- ACID transactions critical for master-data integrity (FR-007 audit + FR-003 record-lock)
- Existing S2 LXC 210 PostgreSQL instance (no new infrastructure)
- Strong FK constraints + JSON columns (audit_log row state snapshots)
- Sunset port to BFlow 2 = PostgreSQL too (per BFlow 2 master-data services design)

### Why Redis (record-lock)
- Existing S2 LXC 210 Redis instance (no new infrastructure)
- Atomic SETNX with TTL = standard distributed lock pattern
- Per `feedback_centralized_nqh_data_ownership` doctrine — Redis already trusted for cache/lock

### Why Vue 3 (not React/Angular)
- Em note Quỳnh team Vue familiarity per prior MDM-WEBAPP-SPEC context
- Vuetify 3 form/table optimized for data-entry UX
- Composition API → easier for backend-heavy devs (dtdanh) to contribute frontend

### Why Vuetify (not Tailwind alone)
- Form-heavy + table-heavy app = Vuetify components handle 80% out-of-box (Data Tables, Forms, Dialogs, Snackbars)
- Tailwind = utility-first, requires more component-building → slower MVP velocity

## 4. Alternatives considered

| Alt | Rejected because |
|---|---|
| Django + DRF | Heavier ORM coupling, slower API velocity vs FastAPI |
| Node.js + Express | Splits NQH Python stack; dtdanh less fluent |
| MongoDB | NoSQL inconsistent with FK + ACID for master-data |
| React | Quỳnh team prefers Vue; React would add learning curve |
| Tailwind-only | Slower MVP velocity for form-heavy app |
| Streamlit / Retool | Locks data + UI vendor; not portable to BFlow 2 |

## 5. Sunset portability (per ADR-005 COGS pipeline §E criteria)

- PostgreSQL schema → BFlow 2 PostgreSQL native (logical model preserved at sunset)
- FastAPI endpoints → BFlow 2 microservices (RESTful contract preserved)
- Redis lock → BFlow 2 distributed cache (Redis or replacement, same pattern)
- Costing-engine = Python module, callable from any framework (BFlow 2 imports OR rewrites in Go if needed — logic spec preserves)
- Vue UI → BFlow 2 frontend (rewrite OR embed iframe transition; UI = least sunset-critical)

## 6. Consequences

### Positive
- Reuses existing NQH infrastructure (PostgreSQL + Redis + Docker + cron) — no new ops surface
- dtdanh velocity high (Python familiarity + reuse patterns)
- OpenAPI + Pydantic = self-documenting + typed frontend integration
- Sunset path clean (PostgreSQL + Python both BFlow 2 compatible)

### Negative
- Two-language project (Python backend + Vue frontend) = context switch for dtdanh
- Redis lock requires careful TTL tuning (lock-leakage = bad UX)
- openpyxl Excel V2 round-trip = brittle to schema drift (need version pin Excel V2 schema)
- Vuetify 3 = newer, smaller ecosystem vs Material-UI for React

## 7. Acceptance

- [ ] @architect ratify stack choice (post-review)
- [ ] @mis governance no-objection
- [ ] dtdanh confirms Python+Vue OK
- [ ] CEO ratify post engine-review checklist

## 8. References

- REQ-001 PRD: `01-planning/REQ-001-mvp-lean-prd.md`
- VISION-001: `00-foundation/00-vision-mdm.md`
- ADR-005 COGS pipeline sunset portability: `apps/clickhouse/docs/02-design/ADR-005-VAT-Interim-Price-Source.md`
- Existing CukCuk scraper Python pattern: `apps/clickhouse/cukcuk/scraper/scraper.py`
- Existing VatDownload Python: `apps/VatDownload/app.py`
- SDLC 6.4.0 Pillar 5 SASE/SE 3.0: `/home/nqh/shared/.sdlc-framework/01-Overview/`
