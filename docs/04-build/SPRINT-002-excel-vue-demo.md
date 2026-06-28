---
spec_id: SPRINT-002
spec_name: "Sprint 2 — Excel V2 Round-Trip + Vue UI (Steward Admin + Staff Search) + DEMO"
spec_version: "1.0.0"
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
phase: MVP Phase 1 of 2 (final sprint before user feedback gate)
---

# Sprint 2 — Excel V2 Round-Trip + Vue UI + DEMO

## Sprint Goal

MVP DEMO ready: Stewards (Thy/Hồng) manage master-data via web UI (replace raw-Excel pain) + Staff (NV) lookup MD codes via search UI (paste-source for CukCuk + Fast Accounting Online). MVP feedback collection setup.

## Schedule

| Attribute | Value |
|---|---|
| Sprint | 2 of 2 (MVP final) |
| Duration | 1 week Sat→Fri |
| Conditional start | T+0 post Sprint 1 sign-off |
| Stretch target | 2026-07-22 (Wed) |
| Team | Same as Sprint 1 (Quỳnh @fullstack + CC + Kimi + advisory em/CEO/dtdanh) |
| Capacity | ~100 effective dev-hours (multi-agent parallel) |

## Parallel Agent Streams

### Stream G — Excel V2 Round-Trip Backend (CC, ~24h)
- SPEC-005 implementation: parser + diff builder + commit logic
- Drive service account integration (read `1K-SL4ae...` via existing `nqh-sheet-reader.json`)
- POST `/excel/import` async upload + parse
- GET `/excel/sync/{run_id}/diffs` review
- POST `/excel/sync/{run_id}/commit` apply
- GET `/excel/export` generates born-clean V2 .xlsx
- Conflict detection (Excel vs MDM both edited) → manual merge UI
- EOW: import golden fixture → all 7 MDGs reflect; export round-trip bit-equal

### Stream H — Vue 3 Steward Admin UI (Kimi, ~32h)
- Vue 3 + Vuetify 3 + Pinia + Vite scaffold
- Auth UI: login/logout/profile (consumes Stream A JWT API)
- 7-MDG list views (Vuetify Data Table — uniform across MDGs)
- 7-MDG detail/edit forms (Vuetify form — uniform pattern)
- Excel V2 import wizard:
  - Upload .xlsx
  - Diff preview (NEW/UPDATE/CONFLICT per row)
  - Bulk accept/reject + per-row review
  - Commit button + progress
- Excel V2 export download button per MDG OR full workbook
- Audit log viewer (basic: list filter by user/date)
- EOW: Thy/Hồng test-drive 4+ MDGs successfully

### Stream I — Vue 3 Staff Search UI (Kimi, ~16h)
- Top-level search bar (all MDGs OR per-MDG)
- Full-text search + filter dropdowns (category/BU/active)
- Data table: ma_*, ten_*, key fields visible
- **Copy-to-clipboard** button per row → format: `ma_xxx | ten_xxx | dvt | gia | ...` (paste-friendly for CukCuk + Fast Accounting Online manual entry forms)
- Bulk-copy selected rows (CSV-like text for Excel paste into CukCuk Fast)
- Search history (last 10 queries per user)
- EOW: NV volunteer test-drives — finds 5+ MD codes + copies into CukCuk/Fast sample form

### Stream J — Drive Sync + Deploy (CC, ~16h)
- Daily 06:00 cron poll Drive `1K-SL4ae...` modifiedTime; if changed → enqueue sync_run
- Web UI notification banner: "New Excel V2 version available — review diffs"
- Docker compose deploy on S2 LXC alongside VatDownload + ClickHouse
- nginx-proxy-manager route `mdm.nhatquangholding.com` (internal-only Phase 1; hairpin NAT per memory)
- Basic Telegram alert wire (cron failure → NQH-Ops-Team)
- EOW: deployed accessible internal URL; Drive poll cron runs

### Stream K — E2E + UAT + DEMO (Quỳnh, ~20h)
- E2E acceptance scenarios per SPEC-007 §8 ACC-001 (MVP subset)
- Steward UAT (Wed afternoon):
  - Thy: edit MDG-003 Product · import Excel V2 → review diffs → commit
  - Hồng: edit MDG-002 Supplier · search MDG-008 NVL
  - 1 other steward volunteer
- Staff UAT (Thu morning):
  - 2-3 NV volunteer: search MD codes · copy → paste into CukCuk test form
- Demo script (Friday 14:00):
  - 5-min: Thy edits Product via UI vs raw Excel comparison
  - 5-min: Hồng imports Excel V2 → diff review → commit flow
  - 5-min: NV searches NVL by name → copy code → paste CukCuk
  - 5-min: audit log + RBAC denial scenarios
  - 5-min: Q&A
- Screen recordings + feedback form
- EOW: MVP DEMO PASS sign-off

### Stream L — Feedback Collection Infrastructure (Quỳnh, ~8h)
- 3-question feedback form (Google Form OR simple web form):
  1. Easier vs raw Excel? (1-5 scale + comment)
  2. Faster to find/edit MD? (1-5 scale + comment)
  3. What's missing? (free text)
- Send to all stewards + NV volunteers post-DEMO
- ≥2 weeks production use → @mis aggregate → Phase 2 trigger decision

## Acceptance Gates (Sprint 2 + MVP)

- [ ] Excel V2 import → 7 MDGs populated correctly via UI
- [ ] Excel V2 export round-trip bit-equal (modulo timestamps)
- [ ] Stewards (Thy + Hồng + 1) complete CRUD tasks via UI unaided
- [ ] Staff (≥3 NV) find MD codes via search + paste copy into CukCuk/Fast sample form
- [ ] MDM deployed on S2 LXC + internal URL accessible
- [ ] Drive daily poll cron runs cleanly 2+ days
- [ ] Telegram alert wire functional (test failure → NQH-Ops-Team delivered)
- [ ] **MVP DEMO** Friday — CEO + @cto-CC + @mis sign-off
- [ ] Feedback form distributed + first responses collected

## Out of scope Sprint 2 (Phase 2 ON-DEMAND)

- ❌ Costing engine (SPEC-006 Phase 2)
- ❌ Full RBAC per-MDG (Phase 2)
- ❌ Record-lock UI (Phase 2)
- ❌ Approval workflow UI (Phase 2)
- ❌ Anomaly dashboard (Phase 2)
- ❌ BFlow 1/2 native API integration (CEO 28/06: "Bflow native access via MDM API" — Phase 2 expose REST endpoints publicly)
- ❌ Production-harden (Sentry · load test · backup) — Phase 2 / Sprint 7+

## Phase 2 trigger (post-MVP-feedback)

After MVP DEMO PASS + ≥2 weeks production use:
- @mis aggregate feedback form responses
- Identify Phase 2 candidates by demand signal (NOT pre-planned)
- CEO ratify Phase 2 scope per evidence
- Em + CEO @architect re-engage for Phase 2 sprint planning

If feedback = "MVP enough for now" → Phase 2 deferred indefinitely (MM#9 honored).

## Risks + Mitigations

| Risk | Mitigation |
|---|---|
| Vue 3 + Excel UI 1-week tight | Stream H = Kimi bulk Vuetify Data Table pattern (uniform across 7 MDGs = repetition); CC reviews UX critical paths |
| Excel V2 conflict UX complex | Phase 1 = simple "side-by-side accept/reject"; advanced merge = Phase 2 |
| Drive API rate limits | 1 poll/day OK (Drive quota generous for service account); manual import remains available |
| Steward UAT availability conflict | em pre-arrange Thy + Hồng calendars (Wed afternoon block) Sprint 1 Friday |
| Staff search UX too generic | Quỳnh demos to 1 NV early Mon-Tue, iterate copy-format per feedback before Thu UAT |
| Deploy issue S2 LXC | Quỳnh practices deploy Wed; troubleshoot Thu; final Fri |

## CEO doctrine context (28/06)

- **Bflow 1/2 = native access** post-MDM-deploy: chúng ta phát triển = NQH-controlled; Bflow can query MDM REST API directly. NO Excel paste needed for Bflow.
- **CukCuk + Fast Accounting Online**: external SaaS = MUST manual entry OR Excel import. MDM staff search UI = paste source optimized for these 2 systems.
- → Stream I copy-format prioritizes CukCuk + Fast Accounting Online manual-entry workflows.

## Sprint Ceremonies

| Day | Event | Duration |
|---|---|---|
| Saturday 09:00 | Sprint Planning kickoff | 1h |
| Mon-Thu 17:00 | Daily standup async (Telegram) | 5min |
| Wed PM | Steward UAT session | 1.5h |
| Thu AM | Staff UAT session | 1h |
| Friday 14:00 | MVP DEMO (CEO + @cto + @mis attend) | 30min |
| Friday 14:30 | Sprint Retro + Phase 2 trigger discussion | 30min |

## References

- ROADMAP-001 v3
- SPRINT-001
- SPEC-005 Excel V2 round-trip (full implementation Sprint 2)
- ADR-001 stack
- Design docs `02-design/`
- CEO directives 28/06 (CukCuk/Fast paste flow · Bflow native access · 2-phase split)
