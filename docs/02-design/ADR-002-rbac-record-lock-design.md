---
spec_id: ADR-002
spec_name: "RBAC + Record-Lock Design (Per MDP SOP Owner Doctrine)"
spec_version: "1.0.0"
status: draft
tier: LITE
stage: "02"
category: architecture
owner: architect-itadmin-ceo-joint
created: 2026-06-28
last_updated: 2026-06-28
binding_sources:
  - "core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Master_Data_Platform/ (8 NQDL/NQH MDP SOPs)"
  - "core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Operations/NQDL-MDP-SOP-000_Master_Data_Entry_and_Change_Control.md"
  - "core/docs/RAG_Library/01_NQH-SOPs/07_Accounting_Handbook/07-FB-Costing/ (WI-101 to WI-105)"
---

# ADR-002 — RBAC + Record-Lock Design

## 1. Context

REQ-001 FR-002 + FR-003: kill 5-owner-no-RBAC concurrent corruption (60M-bug evidence). Authority + ownership MUST follow NQH MDP SOP doctrine (BINDING per CEO 28/06).

## 2. Decision — Role-based RBAC per MDP authority chain

### 2.1 Authority chain per MDP SOPs (verbatim)

Per `NQDL-MDP-SOP-000 §Roles`:
- **KTT NQDL** = chủ sở hữu chất lượng dữ liệu cuối cùng + phê duyệt mã mới/thay đổi lớn
- **Kế toán Kho / Tổng hợp** = xác nhận TK, giá, ĐVT; duyệt cấp dưới
- **IT / Master Data Team** = duy trì template, đồng bộ công cụ, giải quyết lỗi kỹ thuật
- **BU Manager** = đề xuất mã mới; xác nhận `Ap_Dung_BU`
- **Bếp trưởng** (per MDP-SOP-008) = đề xuất NVL mới + xác nhận ĐVT/TK chi phí

### 2.2 Per-MDG owner matrix (BINDING per MDP SOPs)

| MDG | Domain | Final Approver | Input Steward | Proposer | Cite |
|---|---|---|---|---|---|
| MDG-002 | Supplier (NCC) | KTT NQDL | Kế toán Tổng hợp | Mua hàng team (Ms.Hồng) | MDP-SOP-002 |
| MDG-003 | Product (Mã món) | CEO/CFO (per SOP header) — operational = KTT delegate | Nhập liệu (Thy) | BU Manager / Bếp trưởng | MDP-SOP-003 |
| MDG-005 | BTP (Bán thành phẩm) | KTT NQDL | Bếp trưởng | Bếp trưởng | MDP-SOP-005 |
| MDG-008 | NVL (Nguyên vật liệu) | KTT NQDL | Kế toán Kho | Bếp trưởng | MDP-SOP-008 + WI-101 |
| MDG-009 | Customer | CFO NQH (centralized per §2.2 ratify 14/06) | CFO Office Steward | Sales/BU | NQH-MDP-SOP-009 |
| MDG-011 | Cost/Financial (TK, journal codes) | KTT NQDL | Kế toán Tổng hợp/Kho | (KTT initiate) | MDP-SOP-011 + WI-105 |
| MDG-012 | Fixed Asset/Equipment | KTT NQDL (+CFO/CIO per draft) | Hào team | BU Manager | MDP-SOP-012 |

Notes:
- KTT NQDL = final-approver-of-record for most MDGs
- "Chủ sở hữu chất lượng dữ liệu" (per MDP-SOP-000) = data-quality custody, NOT silo
- MDG-009 = NQH centralized per `feedback_centralized_nqh_data_ownership` — CFO Office authority, NOT BU silo
- MDP-SOP-012 status: propose-not-ratify (DRAFT 0.1) — em flag pending CFO/CIO/KTT ratify

### 2.3 RBAC roles in MDM app

```
ROLE_VIEW_ALL          — read-only across all MDGs (transparency baseline)
ROLE_STEWARD_<MDG>     — CRUD on assigned MDG (input steward per matrix above)
ROLE_APPROVER_<MDG>    — approve change-records (KTT NQDL / CFO per matrix)
ROLE_MDT_ADMIN         — IT / Master Data Team — template/schema/tool admin
ROLE_CIO               — override per-domain restrictions (audit-trailed)
ROLE_CFO_NQH           — MDG-009 final + sensitive override authority
```

User → role(s) mapped in PostgreSQL `mdm_user_roles` table (NOT .env hardcode — em correct earlier .env.example which had role→person hardcode; that was provisional placeholder).

### 2.4 Action gate matrix

| Action | Required role |
|---|---|
| READ (any MDG) | ROLE_VIEW_ALL (default for authenticated user) |
| CREATE draft (any MDG) | ROLE_STEWARD_<MDG> |
| UPDATE existing record | ROLE_STEWARD_<MDG> + record-lock held |
| APPROVE record (lock → published) | ROLE_APPROVER_<MDG> |
| DELETE (soft delete) | ROLE_APPROVER_<MDG> |
| HARD DELETE | ROLE_CIO (audit alert) |
| OVERRIDE lock force-release | ROLE_CIO (audit alert) |
| EXCEL IMPORT bulk | ROLE_STEWARD_<MDG> + dry-run preview + APPROVER confirm |
| EXCEL EXPORT | ROLE_VIEW_ALL |
| TEMPLATE schema change | ROLE_MDT_ADMIN |

### 2.5 Record-lock semantics

- **Acquire**: `SET NX EX 900` Redis key `mdm:lock:{mdg}:{record_id}` value=`{user_id}:{session_id}:{acquired_at}`
- **Heartbeat**: client refreshes TTL every 5 min via `EXPIRE`; lock auto-expires 15 min idle
- **Release**: explicit on commit OR client navigation away
- **Conflict**: 2nd user sees 423 LOCKED with current holder + remaining TTL; can WAIT (poll) or CONTACT holder
- **Force-release**: ROLE_CIO only, audit logged
- **Lock scope**: per-record (not per-MDG) — multiple users edit different rows in same MDG concurrently OK

### 2.6 Audit log (per FR-007)

Every CRUD action → `mdm_audit_log` table:
```
audit_id            BIGSERIAL PRIMARY KEY
event_at            TIMESTAMP NOT NULL DEFAULT now()
user_id             TEXT NOT NULL
user_role           TEXT NOT NULL
action              TEXT NOT NULL  -- C/R/U/D/APPROVE/REJECT/LOCK_ACQUIRE/LOCK_FORCE_RELEASE
mdg_domain          TEXT NOT NULL  -- 'MDG-002', etc.
record_id           TEXT NOT NULL  -- domain-specific PK
before_state_json   JSONB          -- NULL for CREATE, populated for U/D
after_state_json    JSONB          -- populated for C/U, NULL for D
ip_address          INET
session_id          TEXT
note                TEXT
```

Append-only (no DELETE/UPDATE on audit_log). Retention ≥ 7 years (accounting compliance).

## 3. Alternatives considered

| Alt | Rejected because |
|---|---|
| Per-row owner hardcode | Violates `feedback_role_based_org_design` — must be role-based, not person-based |
| Optimistic locking (CAS) | UX worse — user fills form, hits save, gets conflict. Pessimistic lock surfaces conflict UPFRONT |
| Long-lived locks (no TTL) | Lock leakage if user disconnects = blocks others indefinitely. 15-min TTL with 5-min heartbeat balances UX + leakage |
| LDAP-only auth (no JWT) | Adds infrastructure dependency (existing OpenWebUI JWT pattern preferred) |

## 4. Implementation handoff to @coder Ms. Quỳnh

- PostgreSQL DDL: see SPEC-002 (tables `mdm_user`, `mdm_role`, `mdm_user_roles`, `mdm_audit_log`)
- Redis pattern: see SPEC-003 OpenAPI section §Locks
- Owner matrix initial seed: see SPEC-004 RBAC Matrix
- Test scenarios: see ACC-001

## 5. References

- MDP SOPs (8): `core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Master_Data_Platform/`
- MDP-SOP-000 Master Data Entry: `core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Operations/NQDL-MDP-SOP-000_Master_Data_Entry_and_Change_Control.md`
- WI-101→105: `core/docs/RAG_Library/01_NQH-SOPs/07_Accounting_Handbook/07-FB-Costing/`
- Memory: `feedback_role_based_org_design` · `feedback_centralized_nqh_data_ownership` · `feedback_sop_issuance_authority`
- REQ-001 FR-002/003: `01-planning/REQ-001-mvp-lean-prd.md`
- SPEC-002 DDL: `02-design/SPEC-002-postgres-ddl.md`
- SPEC-003 OpenAPI: `02-design/SPEC-003-openapi-spec.md`
- SPEC-004 RBAC matrix: `02-design/SPEC-004-rbac-matrix.md`
