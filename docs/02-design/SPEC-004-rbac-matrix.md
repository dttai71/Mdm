---
spec_id: SPEC-004
spec_name: "RBAC Matrix — Owner per MDP SOP Authority Doctrine"
spec_version: "1.0.0"
status: draft
tier: LITE
stage: "02"
category: architecture
owner: architect-itadmin-ceo-joint
created: 2026-06-28
last_updated: 2026-06-28
binding_sources:
  - "NQDL-MDP-SOP-000 §Roles (authority chain)"
  - "NQDL-MDP-SOP-002/003/005/008/011/012 + NQH-MDP-SOP-009 (per-MDG owners)"
  - "WI-101 §1.2 (NVL owner — KTT + Bếp trưởng) · WI-105 §5.1 (approval tier)"
  - "feedback_role_based_org_design (ROLE-based, NOT person silo)"
---

# SPEC-004 — RBAC Matrix (per MDP SOP authority)

## 1. Role inventory

| Role ID | Description | Authority source |
|---|---|---|
| `ROLE_VIEW_ALL` | Read-only across all MDGs | Default for authenticated user (transparency baseline) |
| `ROLE_STEWARD_MDG-002` | CRUD MDG-002 Supplier | MDP-SOP-002 — Kế toán Tổng hợp |
| `ROLE_APPROVER_MDG-002` | Approve MDG-002 changes | MDP-SOP-002 — KTT NQDL |
| `ROLE_STEWARD_MDG-003` | CRUD MDG-003 Product | MDP-SOP-003 — Thy / Nhập liệu (Phòng OPS) |
| `ROLE_APPROVER_MDG-003` | Approve MDG-003 changes | MDP-SOP-003 — CEO/CFO (delegate to KTT operational) |
| `ROLE_STEWARD_MDG-005` | CRUD MDG-005 BTP | MDP-SOP-005 — Bếp trưởng (input) |
| `ROLE_APPROVER_MDG-005` | Approve MDG-005 changes | MDP-SOP-005 — KTT NQDL |
| `ROLE_STEWARD_MDG-008` | CRUD MDG-008 NVL | MDP-SOP-008 + WI-101 — Kế toán Kho (input) |
| `ROLE_APPROVER_MDG-008` | Approve MDG-008 changes | MDP-SOP-008 — KTT NQDL |
| `ROLE_PROPOSER_MDG-008` | Propose new NVL (write to draft, NOT direct commit) | WI-101 — Bếp trưởng |
| `ROLE_STEWARD_MDG-009` | CRUD MDG-009 Customer | NQH-MDP-SOP-009 — CFO Office Steward (centralized per §2.6) |
| `ROLE_APPROVER_MDG-009` | Approve MDG-009 changes | NQH-MDP-SOP-009 — CFO NQH (centralized authority) |
| `ROLE_STEWARD_MDG-011` | CRUD MDG-011 Cost master | MDP-SOP-011 — Kế toán Tổng hợp |
| `ROLE_APPROVER_MDG-011` | Approve MDG-011 changes | MDP-SOP-011 — KTT NQDL |
| `ROLE_STEWARD_MDG-012` | CRUD MDG-012 Equipment | MDP-SOP-012 — Hào team (input) |
| `ROLE_APPROVER_MDG-012` | Approve MDG-012 changes | MDP-SOP-012 — KTT + CFO + CIO (3-way per DRAFT) |
| `ROLE_BU_MANAGER` | Approve `Ap_Dung_BU` per record | MDP-SOP-000 §Roles |
| `ROLE_MDT_ADMIN` | Template/schema/tool admin | MDP-SOP-000 §Roles — IT / Master Data Team |
| `ROLE_CIO` | Override per-domain restrictions (audit alert) | CIO authority |
| `ROLE_CFO_NQH` | MDG-009 final + sensitive override | NQH-MDP-SOP-009 |

## 2. Initial user → role seed (per MDP authority + current org per `user_nqh_key_people`)

> **NOTE**: This is the INITIAL seed at MVP launch. Users + role assignments managed via `/rbac/*` endpoints post-launch (per SPEC-003 §2.2). NOT hardcoded in .env (em correct earlier `.env.example` placeholder).

| user_id | full_name | Roles (initial seed) |
|---|---|---|
| `thy` | Nguyễn Ngọc Thy | ROLE_VIEW_ALL · ROLE_STEWARD_MDG-003 · ROLE_PROPOSER_MDG-008 |
| `quynh` | Quỳnh (Ms.) | ROLE_VIEW_ALL · @coder role (implementation, not data steward) · NO MDG-CRUD role at MVP |
| `ms_hong` | Ms. Hồng | ROLE_VIEW_ALL · ROLE_STEWARD_MDG-002 (Mua hàng team) |
| `ktt` | KTT NQDL (role TBD person) | ROLE_VIEW_ALL · ROLE_APPROVER_MDG-002/003/005/008/011/012 (KTT authority across MDGs) |
| `ke_toan_kho` | Kế toán Kho (role TBD person) | ROLE_VIEW_ALL · ROLE_STEWARD_MDG-008 · ROLE_STEWARD_MDG-011 |
| `ke_toan_tong_hop` | Kế toán Tổng hợp (role TBD person) | ROLE_VIEW_ALL · ROLE_STEWARD_MDG-002 · ROLE_STEWARD_MDG-011 |
| `bep_truong_bkl` | Bếp trưởng BKL (role TBD per BU) | ROLE_VIEW_ALL · ROLE_STEWARD_MDG-005 · ROLE_PROPOSER_MDG-008 |
| `bep_truong_thom` | Bếp trưởng Thom | (same as BKL) |
| `bep_truong_kupid` | Bếp trưởng Kupid | (same) |
| `hao` | Hào (Equipment team) | ROLE_VIEW_ALL · ROLE_STEWARD_MDG-012 |
| `cfo_office` | CFO Office Steward (role TBD person) | ROLE_VIEW_ALL · ROLE_STEWARD_MDG-009 |
| `tai_cfo` | CFO NQH (CEO Tài interim per memory) | ROLE_VIEW_ALL · ROLE_APPROVER_MDG-009 · ROLE_CFO_NQH |
| `dvhiep` | Đinh Vũ Hiệp (CIO) | ROLE_CIO · ROLE_MDT_ADMIN · ROLE_VIEW_ALL |
| `bu_mgr_<bu>` | Per BU (Thắng/Kiệt/Vy/etc.) | ROLE_BU_MANAGER · ROLE_VIEW_ALL |
| `itadmin_bot` | @itadmin CC agent | ROLE_MDT_ADMIN (limited, audit-trailed) |

## 3. Multi-role assignment (composition)

User can hold multiple roles (e.g., KTT holds ROLE_APPROVER_MDG-002/003/005/008/011/012 = 6 approver roles). RBAC check = ANY role matches required role for action.

## 4. ROLE_BU_MANAGER scope (per MDP-SOP-000)

BU Manager approves `Ap_Dung_BU` field per record. Implementation:
- Record CRUD requires steward + approver per MDG
- Record commit additionally requires BU Manager confirm `Ap_Dung_BU` per affected BU
- BU Manager scope = own BU only (cannot approve `Ap_Dung_BU` for other BU)

## 5. Approval workflow (per MDP-SOP-000 SLA)

```
T+0  Steward creates draft record  (state='DRAFT')
T+2  Approver reviews + approves   (state='APPROVED' → published downstream)
T+2  Excel V2 reflects approved record (next sync cycle)
```

Per MDP-SOP-000:
- Steward NHẬP T+2 SLA
- Approver REVIEW T+2 SLA
- TOTAL T+4 SLA for new record / change

## 6. Override + escalation

| Scenario | Required role |
|---|---|
| Force-release stuck lock | ROLE_CIO (audit alert) |
| Bypass RBAC for emergency fix | ROLE_CIO (audit alert + reason mandatory) |
| Hard delete (vs soft) | ROLE_CIO (audit alert) |
| Change role assignment | ROLE_CIO + 2nd approver (4-eyes) |
| Schema migration | ROLE_MDT_ADMIN |

## 7. RBAC audit (per ADR-002 §2.6)

Every RBAC enforcement event logged to `mdm_audit_log`:
- Successful action: `action=CREATE/UPDATE/DELETE/APPROVE`, role used
- 403 denial: `action=READ/CREATE/...` (attempted), `note='RBAC_DENY: missing role X'`
- Lock acquire/release: `action=LOCK_ACQUIRE/LOCK_RELEASE`
- Force-release: `action=LOCK_FORCE_RELEASE`, `note='Override reason: ...'`

## 8. RBAC governance

- **Role grant authority**: ROLE_CIO (Hiệp) + 2nd approver
- **Role definition changes**: ROLE_MDT_ADMIN + @mis governance review
- **MVP launch user list**: per §2 initial seed, ratified by KTT + CFO + CIO + @mis

## 9. Test scenarios (acceptance)

```
TC-RBAC-001: User WITHOUT ROLE_STEWARD_MDG-008 attempts POST /mdg-008
             → HTTP 403; audit_log row 'CREATE attempted, RBAC_DENY'
TC-RBAC-002: User WITH ROLE_STEWARD_MDG-008 attempts POST /mdg-008 (no lock needed for CREATE)
             → HTTP 201; audit_log row 'CREATE success'
TC-RBAC-003: User STEWARD attempts PUT /mdg-008/RAW-0088 WITHOUT lock
             → HTTP 423 'No lock held'; audit_log row 'UPDATE attempted, NO_LOCK'
TC-RBAC-004: User A acquires lock; User B (also steward) attempts PUT same record
             → User B HTTP 423 LOCKED with user A info
TC-RBAC-005: KTT (ROLE_APPROVER_MDG-008) approves draft → state DRAFT→APPROVED
             → HTTP 200; audit_log 'APPROVE success'
TC-RBAC-006: CIO force-releases stuck lock
             → HTTP 200; audit_log 'LOCK_FORCE_RELEASE, override reason: ...'
TC-RBAC-007: User attempts to delete from mdm_audit_log via API
             → HTTP 405 / no endpoint exists; PostgreSQL REVOKE enforces
TC-RBAC-008: Multi-role user (KTT) approves across MDGs in single session
             → Each approval audit-logged with role used
```

## 10. References

- ADR-002 RBAC + record-lock: `02-design/ADR-002-rbac-record-lock-design.md`
- SPEC-002 DDL (RBAC tables): `02-design/SPEC-002-postgres-ddl.md` §2
- SPEC-003 OpenAPI (RBAC endpoints): `02-design/SPEC-003-openapi-spec.md` §2.2
- MDP-SOPs: `core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Master_Data_Platform/`
- MDP-SOP-000: `core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Operations/NQDL-MDP-SOP-000_Master_Data_Entry_and_Change_Control.md`
- WI-101 owner doctrine: `core/docs/RAG_Library/01_NQH-SOPs/07_Accounting_Handbook/07-FB-Costing/NQH-HO-ACC-WI-101_Raw_Material_Master.md`
- Memory: `feedback_role_based_org_design` · `feedback_centralized_nqh_data_ownership` · `user_nqh_key_people` · `project_ceo_cfo_dual_hat` · `project_master_data_ownership`
