---
spec_id: SPEC-003
spec_name: "OpenAPI v3 Spec — REST API surface"
spec_version: "1.0.0"
status: draft
tier: LITE
stage: "02"
category: architecture
owner: architect-itadmin-ceo-joint
created: 2026-06-28
last_updated: 2026-06-28
---

# SPEC-003 — OpenAPI v3 REST API Spec

**Implementation**: FastAPI auto-generates `/openapi.json` + Swagger UI at `/docs`. This spec defines the surface @coder Ms. Quỳnh implements.

## 1. Base + Auth

```
Base URL:  https://mdm.nhatquangholding.com/api/v1
Auth:      Bearer JWT (per ADR-001 + OpenWebUI integration TBD)
Content:   application/json (UTF-8)
```

All endpoints return:
- `200/201/204` success
- `400` validation error (JSON: `{detail: [{loc, msg, type}]}`)
- `401` unauthenticated
- `403` unauthorized (RBAC fail)
- `404` resource not found
- `409` business conflict (e.g. duplicate)
- `423` LOCKED (record-lock conflict per ADR-002)
- `500` server error

## 2. Endpoint inventory

### 2.1 Auth + Session
```
POST   /auth/login              → JWT token
POST   /auth/logout             → revoke session
GET    /auth/me                 → current user + roles
```

### 2.2 RBAC management (ROLE_CIO only)
```
GET    /rbac/users              → list users
POST   /rbac/users              → create user
PATCH  /rbac/users/{user_id}    → update user
GET    /rbac/roles              → list roles
POST   /rbac/users/{user_id}/roles    → assign role
DELETE /rbac/users/{user_id}/roles/{role_id}   → revoke role
```

### 2.3 Locks
```
POST   /locks/{mdg}/{record_id}/acquire     → 200 OK with TTL | 423 LOCKED with holder
POST   /locks/{mdg}/{record_id}/heartbeat   → refresh TTL
POST   /locks/{mdg}/{record_id}/release     → release (own)
POST   /locks/{mdg}/{record_id}/force       → CIO force-release (audit)
GET    /locks?held_by=<user_id>             → list user's locks
```

### 2.4 MDG CRUD (uniform pattern per MDG)

Pattern (replace `{mdg}` with `002`/`003`/`004`/`005`/`008`/`009`/`011`/`012`):

```
GET    /mdg-{mdg}                          → list (paginated, filterable)
       Query: ?page=1&size=50&filter=<dsl>&include_deleted=false
GET    /mdg-{mdg}/{record_id}              → get by ID
POST   /mdg-{mdg}                          → create (steward role required)
PUT    /mdg-{mdg}/{record_id}              → update (steward + lock required)
PATCH  /mdg-{mdg}/{record_id}              → partial update (steward + lock)
DELETE /mdg-{mdg}/{record_id}              → soft delete (approver role)
POST   /mdg-{mdg}/{record_id}/approve      → approve draft (approver role)
POST   /mdg-{mdg}/{record_id}/reject       → reject draft (approver role)
GET    /mdg-{mdg}/{record_id}/history      → audit trail per record
```

### 2.5 Recipe (MDG-003 + MDG-005)
```
GET    /recipes/mon/{ma_mon}            → recipe BOM for món
PUT    /recipes/mon/{ma_mon}            → upsert recipe (steward)
GET    /recipes/btp/{ma_btp}            → BTP recipe + yield
PUT    /recipes/btp/{ma_btp}            → upsert (yield_qty + components)
```

### 2.6 Excel V2 round-trip
```
POST   /excel/import                    → multipart upload .xlsx → returns sync_run_id (async)
GET    /excel/sync/{run_id}/status      → run status + diff preview
GET    /excel/sync/{run_id}/diffs       → list diffs (paginated)
POST   /excel/sync/{run_id}/diffs/{diff_id}/accept → accept diff
POST   /excel/sync/{run_id}/diffs/{diff_id}/reject → reject diff
POST   /excel/sync/{run_id}/commit      → commit all accepted diffs
GET    /excel/export?mdg=008            → download .xlsx (born-clean format compatible)
```

### 2.7 Costing engine
```
POST   /engine/compute-recipe-cost      → body: {ma_mon, transaction_datetime}  → cost breakdown
POST   /engine/freeze-bill              → body: {bill_id, items[]} → freezes cost in fact_sales_detail_with_cost
GET    /engine/cost-history/{ma_nvl}    → price history time-machine (per ADR-005)
POST   /engine/recompute-snapshot       → trigger daily snapshot (admin)
POST   /engine/validate-recipe          → dry-run cost compute (engineering tool)
```

### 2.8 Anomaly + monitoring
```
GET    /anomalies                       → list (paginated)
       Query: ?type=MISSING_RECIPE&severity=HIGH&from=2026-06-01
POST   /anomalies/{anomaly_id}/resolve  → mark resolved
GET    /health                          → service health
GET    /metrics                         → Prometheus
```

## 3. Schema definitions (key examples — @coder generates Pydantic from OpenAPI)

### 3.1 MDG-008 NVL schema (illustrative)
```yaml
NVLBase:
  type: object
  required: [ma_nvl, ten_nvl, muc_so_che, dvt_mua, dvt_xuat]
  properties:
    ma_nvl:
      type: string
      pattern: '^(RAW|PRE|RTU|CON|INT)-[0-9]{4}$'
    ten_nvl: {type: string, maxLength: 200}
    cap_1: {type: string}
    cap_2: {type: string}
    tinh_trang: {type: string, enum: [FRE, FRO, DRY, CAN, LIQ]}
    muc_so_che: {type: string, enum: [RAW, PRE, RTU, CON, INT]}
    dvt_mua: {type: string}
    he_so_quy_doi: {type: number, format: float, minimum: 0}
    dvt_xuat: {type: string}
    phuong_phap_tinh_gia:
      type: string
      enum: [POINT_IN_TIME_PRICE, FIFO, BINH_QUAN, DICH_DANH]
      default: POINT_IN_TIME_PRICE
    ap_dung_bu:
      type: array
      items: {type: string}
    tk_kho: {type: string, default: '152'}
    tk_chi_phi: {type: string, default: '632'}
    ma_nvl_goc: {type: string, nullable: true}
    ty_le_yield: {type: number, format: float, minimum: 0, maximum: 1}
    don_gia_sau_so_che: {type: number, format: float, minimum: 0}
    is_active: {type: boolean, default: true}
```

### 3.2 Lock response
```yaml
LockResponse:
  type: object
  properties:
    lock_key: {type: string}
    acquired_at: {type: string, format: date-time}
    expires_at: {type: string, format: date-time}
    held_by: {type: string}
LockConflict (HTTP 423):
  type: object
  properties:
    held_by: {type: string}
    held_since: {type: string, format: date-time}
    expires_at: {type: string, format: date-time}
    contact_url: {type: string}
```

### 3.3 Audit log entry
```yaml
AuditEntry:
  type: object
  properties:
    audit_id: {type: integer}
    event_at: {type: string, format: date-time}
    user_id: {type: string}
    user_role: {type: string}
    action: {type: string, enum: [CREATE,READ,UPDATE,DELETE,APPROVE,REJECT,LOCK_ACQUIRE,LOCK_RELEASE,LOCK_FORCE_RELEASE,EXCEL_IMPORT,EXCEL_EXPORT]}
    mdg_domain: {type: string}
    record_id: {type: string}
    before_state_json: {type: object}
    after_state_json: {type: object}
    ip_address: {type: string}
    note: {type: string}
```

## 4. RBAC enforcement matrix (per ADR-002 §2.4)

| Endpoint pattern | Required role |
|---|---|
| GET / | ROLE_VIEW_ALL |
| POST /mdg-{mdg} | ROLE_STEWARD_<mdg> |
| PUT, PATCH /mdg-{mdg}/{id} | ROLE_STEWARD_<mdg> + lock |
| DELETE /mdg-{mdg}/{id} | ROLE_APPROVER_<mdg> |
| POST /mdg-{mdg}/{id}/approve | ROLE_APPROVER_<mdg> |
| POST /excel/import (steward), commit (approver) | role per step |
| POST /engine/* | ROLE_VIEW_ALL (read) + ROLE_APPROVER for freeze writes |
| POST /locks/.../force | ROLE_CIO |
| POST /rbac/* | ROLE_CIO |

## 5. Pagination + filtering convention

```
?page=1&size=50              → 1-indexed, max size=200
?filter=field:op:value       → DSL: 'is_active:eq:true', 'ten_nvl:contains:Bò', etc.
?sort=field:asc|desc
```

Response wrapper:
```json
{
  "data": [...],
  "meta": {
    "page": 1, "size": 50, "total": 4097, "total_pages": 82
  }
}
```

## 6. References

- ADR-001 stack: `02-design/ADR-001-stack-choice.md`
- ADR-002 RBAC: `02-design/ADR-002-rbac-record-lock-design.md`
- SPEC-002 DDL: `02-design/SPEC-002-postgres-ddl.md`
- SPEC-004 RBAC matrix: `02-design/SPEC-004-rbac-matrix.md`
- SPEC-005 Excel V2 round-trip: `02-design/SPEC-005-excel-v2-roundtrip.md`
- SPEC-006 Engine LOGIC-SPEC: `02-design/SPEC-006-costing-engine-logic-spec.md`
