---
spec_id: SPEC-002
spec_name: "PostgreSQL DDL — 7 MDG Tables + Audit + Lock Registry + RBAC"
spec_version: "1.0.0"
status: draft
tier: LITE
stage: "02"
category: architecture
owner: architect-itadmin-ceo-joint
created: 2026-06-28
last_updated: 2026-06-28
binding_sources:
  - "MDP-SOP-002/003/005/008/009/011/012 (NQDL Master_Data_Platform)"
  - "WI-101→105 (07-FB-Costing)"
  - "MASTER-DATA-V2.xlsx Operations (Excel V2 schema kiểm nghiệm)"
---

# SPEC-002 — PostgreSQL DDL

**Target**: PostgreSQL 16 on S2 LXC 210 (existing instance). Database: `mdm`. Schema: `public`. Migrations via Alembic.

## 1. Migration baseline

```bash
# @coder Ms. Quỳnh: bootstrap
alembic init alembic
# Edit alembic.ini sqlalchemy.url per .env
# Migration 001 = baseline (all DDL below)
alembic revision -m "baseline_mdm_schema" 
# Apply
alembic upgrade head
```

## 2. RBAC core tables

```sql
-- ============================================================
-- RBAC core
-- ============================================================

CREATE TABLE mdm_user (
    user_id         TEXT PRIMARY KEY,                 -- e.g. 'thy', 'quynh', 'ms_hong', 'ktt', 'hao', 'dvhiep'
    full_name       TEXT NOT NULL,
    email           TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMP NOT NULL DEFAULT now(),
    updated_at      TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE mdm_role (
    role_id         TEXT PRIMARY KEY,                 -- e.g. 'ROLE_STEWARD_MDG-008', 'ROLE_APPROVER_MDG-008'
    role_name       TEXT NOT NULL,
    description     TEXT,
    created_at      TIMESTAMP NOT NULL DEFAULT now()
);

CREATE TABLE mdm_user_role (
    user_id         TEXT NOT NULL REFERENCES mdm_user(user_id),
    role_id         TEXT NOT NULL REFERENCES mdm_role(role_id),
    assigned_by     TEXT NOT NULL,
    assigned_at     TIMESTAMP NOT NULL DEFAULT now(),
    PRIMARY KEY (user_id, role_id)
);

-- Initial seed (per ADR-002 §2.2 owner matrix + SPEC-004)
-- Inserted via separate seed migration, not baseline DDL
```

## 3. Audit log (per ADR-002 §2.6)

```sql
CREATE TABLE mdm_audit_log (
    audit_id            BIGSERIAL PRIMARY KEY,
    event_at            TIMESTAMP NOT NULL DEFAULT now(),
    user_id             TEXT NOT NULL,
    user_role           TEXT NOT NULL,
    action              TEXT NOT NULL CHECK (action IN ('CREATE','READ','UPDATE','DELETE','APPROVE','REJECT','LOCK_ACQUIRE','LOCK_RELEASE','LOCK_FORCE_RELEASE','EXCEL_IMPORT','EXCEL_EXPORT')),
    mdg_domain          TEXT NOT NULL,                -- 'MDG-002', 'MDG-003', ...
    record_id           TEXT NOT NULL,
    before_state_json   JSONB,
    after_state_json    JSONB,
    ip_address          INET,
    session_id          TEXT,
    note                TEXT
);

CREATE INDEX idx_audit_event_at ON mdm_audit_log(event_at DESC);
CREATE INDEX idx_audit_mdg_record ON mdm_audit_log(mdg_domain, record_id);
CREATE INDEX idx_audit_user ON mdm_audit_log(user_id);

-- Append-only enforcement (PostgreSQL: revoke UPDATE + DELETE for app role)
REVOKE UPDATE, DELETE ON mdm_audit_log FROM PUBLIC;
GRANT INSERT, SELECT ON mdm_audit_log TO mdm_app;
-- Retention: handled by application-level partition + archive job (7-year policy)
```

## 4. Lock registry (denormalized for query — Redis primary)

```sql
-- Redis is primary lock store. PostgreSQL mirror for query/audit only.
CREATE TABLE mdm_lock_registry (
    lock_key            TEXT PRIMARY KEY,             -- e.g. 'mdm:lock:MDG-008:RAW-0088'
    mdg_domain          TEXT NOT NULL,
    record_id           TEXT NOT NULL,
    held_by             TEXT NOT NULL,                -- user_id
    session_id          TEXT NOT NULL,
    acquired_at         TIMESTAMP NOT NULL DEFAULT now(),
    expires_at          TIMESTAMP NOT NULL,           -- acquired_at + 15 min
    heartbeat_at        TIMESTAMP NOT NULL DEFAULT now()
);

CREATE INDEX idx_lock_mdg_record ON mdm_lock_registry(mdg_domain, record_id);
CREATE INDEX idx_lock_held_by ON mdm_lock_registry(held_by);
```

## 5. MDG-002 Supplier (per MDP-SOP-002)

```sql
CREATE TABLE mdg_002_supplier (
    ma_ncc              TEXT PRIMARY KEY,             -- 'NCC-####'
    ten_ncc             TEXT NOT NULL,
    mst                 TEXT,                         -- mã số thuế
    dia_chi             TEXT,
    sdt                 TEXT,
    email               TEXT,
    ap_dung_bu          TEXT[],                       -- array BU codes
    ncc_chinh_cho       TEXT[],                       -- ma_nvl[] mặc định
    tk_phai_tra         TEXT DEFAULT '331',
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    nguoi_de_xuat       TEXT,
    nguoi_phe_duyet     TEXT,
    ngay_phe_duyet      DATE,
    source_system       TEXT NOT NULL DEFAULT 'MDM',  -- 'MDM' | 'EXCEL_V2' | 'BFLOW2'
    created_at          TIMESTAMP NOT NULL DEFAULT now(),
    updated_at          TIMESTAMP NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMP                     -- soft delete
);
```

## 6. MDG-003 Product (Mã món — per MDP-SOP-003 + WI-104)

```sql
CREATE TABLE mdg_003_product (
    ma_mon              TEXT PRIMARY KEY,             -- 'PRD-####'
    ten_mon_vi          TEXT NOT NULL,
    ten_cukcuk_hoadon   TEXT,                         -- bridge cho CukCuk JOIN
    ap_dung_cukcuk      BOOLEAN DEFAULT TRUE,
    ten_mon_en          TEXT,
    dinh_luong_pax      DECIMAL(8,2),
    category            TEXT,
    sub_category        TEXT,
    product_type        TEXT,                         -- 'main' | 'topping' | 'beverage' | 'combo'
    primary_bu          TEXT,
    available_bus       TEXT[],
    is_cross_brand      BOOLEAN DEFAULT FALSE,
    selling_price       DECIMAL(18,2),
    vat_rate            DECIMAL(5,2),
    cogs_method         TEXT DEFAULT 'FIFO',          -- metadata only (interim POINT_IN_TIME_PRICE)
    recipe_id           TEXT,                         -- FK→mdg_recipe_mon_bom (composite)
    has_recipe          BOOLEAN DEFAULT FALSE,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    nguoi_de_xuat       TEXT,
    nguoi_phe_duyet     TEXT,
    ngay_phe_duyet      DATE,
    source_system       TEXT NOT NULL DEFAULT 'MDM',
    created_at          TIMESTAMP NOT NULL DEFAULT now(),
    updated_at          TIMESTAMP NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMP
);
```

## 7. MDG-004 Site (BU/branch — per MDP-SOP doctrine)

```sql
CREATE TABLE mdg_004_site (
    ma_site             TEXT PRIMARY KEY,             -- 'BKL', 'TMM', 'Kupid', 'AFS', 'THOM', 'ADR1', 'ADR2'
    ten_site            TEXT NOT NULL,
    site_type           TEXT,                         -- 'PC' (Profit Center) | 'sub-PC' | 'CC' (Cost Center)
    parent_site         TEXT,                         -- sub-PC parent ref
    dia_chi             TEXT,
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    is_pc_authoritative BOOLEAN DEFAULT FALSE,        -- per project_bu_pc_structure_27062026
    source_system       TEXT NOT NULL DEFAULT 'MDM',
    created_at          TIMESTAMP NOT NULL DEFAULT now(),
    updated_at          TIMESTAMP NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMP
);
```

## 8. MDG-005 BTP (Bán thành phẩm — per MDP-SOP-005 + WI-103)

```sql
CREATE TABLE mdg_005_btp (
    ma_btp              TEXT PRIMARY KEY,             -- 'INT-####'
    ten_btp             TEXT NOT NULL,
    dvt                 TEXT NOT NULL,
    yield_qty           DECIMAL(15,4),                -- CỘT L Excel V2 (CRITICAL per @mis §C.4)
    yield_uom           TEXT,                         -- CỘT M Excel V2
    don_gia_sau_che_bien DECIMAL(18,2),               -- pre-computed cache, recompute trigger when components/yield change
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    source_system       TEXT NOT NULL DEFAULT 'MDM',
    created_at          TIMESTAMP NOT NULL DEFAULT now(),
    updated_at          TIMESTAMP NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMP
);

-- BTP recipe (BTP → component NVL/PRE)
CREATE TABLE mdg_005_btp_recipe (
    ma_btp              TEXT NOT NULL REFERENCES mdg_005_btp(ma_btp),
    line_no             INT NOT NULL,
    ma_component        TEXT NOT NULL,                -- 'PRE-####' or 'RTU-####'
    component_type      TEXT NOT NULL CHECK (component_type IN ('PRE','RTU','CON')),
    dinh_luong          DECIMAL(15,4) NOT NULL,
    dvt                 TEXT NOT NULL,
    PRIMARY KEY (ma_btp, line_no)
);
```

## 9. MDG-008 NVL (per MDP-SOP-008 + WI-101 + WI-102)

```sql
CREATE TABLE mdg_008_nvl (
    ma_nvl              TEXT PRIMARY KEY,             -- 'RAW-####', 'PRE-####', 'RTU-####', 'CON-####'
    ten_nvl             TEXT NOT NULL,
    cap_1               TEXT,                         -- per ref_ValidationLists
    cap_2               TEXT,
    tinh_trang          TEXT,                         -- 'FRE'|'FRO'|'DRY'|'CAN'|'LIQ'
    muc_so_che          TEXT NOT NULL CHECK (muc_so_che IN ('RAW','PRE','RTU','CON','INT')),
    dvt_mua             TEXT NOT NULL,
    he_so_quy_doi       DECIMAL(15,4) DEFAULT 1,
    dvt_xuat            TEXT NOT NULL,
    phuong_phap_tinh_gia TEXT DEFAULT 'POINT_IN_TIME_PRICE'  -- per ADR-003 honest-ceiling (interim)
                         CHECK (phuong_phap_tinh_gia IN ('POINT_IN_TIME_PRICE','FIFO','BINH_QUAN','DICH_DANH')),
    ap_dung_bu          TEXT[],
    tk_kho              TEXT DEFAULT '152',
    tk_chi_phi          TEXT DEFAULT '632',
    -- Yield (per WI-102 sơ chế) — RAW→PRE conversion
    ma_nvl_goc          TEXT,                          -- FK→mdg_008_nvl(ma_nvl) for PRE pointing to RAW source
    ty_le_yield         DECIMAL(5,4),                  -- yield rate per sơ chế
    don_gia_sau_so_che  DECIMAL(18,2),                 -- cột H Excel V2 = post-yield price (INTERIM PRICE SOURCE per ADR-005 EXCEL_V2)
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    nguoi_de_xuat       TEXT,
    nguoi_phe_duyet     TEXT,
    ngay_phe_duyet      DATE,
    source_system       TEXT NOT NULL DEFAULT 'MDM',
    created_at          TIMESTAMP NOT NULL DEFAULT now(),
    updated_at          TIMESTAMP NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMP
);
```

## 10. MDG-009 Customer (per NQH-MDP-SOP-009 + CFO centralized)

```sql
CREATE TABLE mdg_009_customer (
    ma_kh               TEXT PRIMARY KEY,             -- 'CUS-####'
    ten_kh              TEXT NOT NULL,
    mst                 TEXT,
    loai_kh             TEXT,                         -- 'le' | 'doan' | 'doanh_nghiep'
    nguon_kh            TEXT,                         -- 'walk_in' | 'booking' | 'referral' | 'partner'
    sdt                 TEXT,
    email               TEXT,
    dia_chi             TEXT,
    ngay_sinh           DATE,
    gioi_tinh           TEXT,
    consent_marketing   BOOLEAN DEFAULT FALSE,        -- PDPL 2025 compliance
    consent_date        DATE,
    loyalty_tier        TEXT,                         -- 'standard' | 'silver' | 'gold' | 'platinum'
    primary_bu          TEXT,                         -- BU first contact
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    source_system       TEXT NOT NULL DEFAULT 'MDM',
    created_at          TIMESTAMP NOT NULL DEFAULT now(),
    updated_at          TIMESTAMP NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMP
);
```

## 11. MDG-011 Cost/Financial (per MDP-SOP-011 + WI-105)

```sql
CREATE TABLE mdg_011_cost_master (
    ma_chi_phi          TEXT PRIMARY KEY,
    ten_chi_phi         TEXT NOT NULL,
    tk_chi_phi          TEXT NOT NULL,                -- TK 6xx series
    tk_doi_ung          TEXT,
    loai_chi_phi        TEXT,                         -- 'cost_of_sale' | 'opex' | 'admin' | 'capex'
    phan_bo_method      TEXT,                         -- 'direct' | 'pro_rata' | 'activity_based'
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    source_system       TEXT NOT NULL DEFAULT 'MDM',
    created_at          TIMESTAMP NOT NULL DEFAULT now(),
    updated_at          TIMESTAMP NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMP
);
```

## 12. MDG-012 Equipment (per MDP-SOP-012 DRAFT)

```sql
CREATE TABLE mdg_012_equipment (
    ma_thiet_bi         TEXT PRIMARY KEY,             -- 'EQ-####' or 'FA-####'
    ten_thiet_bi        TEXT NOT NULL,
    loai                TEXT NOT NULL CHECK (loai IN ('TSCD','CCDC')),  -- TSCĐ vs CCDC
    nha_san_xuat        TEXT,
    so_seri             TEXT,
    ngay_mua            DATE,
    nguyen_gia          DECIMAL(18,2),
    gia_tri_hien_tai    DECIMAL(18,2),
    thoi_gian_su_dung_thang INT,
    bu_su_dung          TEXT,                         -- FK→mdg_004_site
    tk_tscd             TEXT,                         -- '211' for TSCĐ
    tk_khau_hao         TEXT,                         -- '214' for TSCĐ
    tinh_trang          TEXT,                         -- 'active' | 'maintenance' | 'retired'
    is_active           BOOLEAN NOT NULL DEFAULT TRUE,
    source_system       TEXT NOT NULL DEFAULT 'MDM',
    created_at          TIMESTAMP NOT NULL DEFAULT now(),
    updated_at          TIMESTAMP NOT NULL DEFAULT now(),
    deleted_at          TIMESTAMP
);
```

## 13. Recipe (PRD → component) per WI-104

```sql
CREATE TABLE mdg_recipe_mon_bom (
    ma_mon              TEXT NOT NULL REFERENCES mdg_003_product(ma_mon),
    line_no             INT NOT NULL,
    ma_component        TEXT NOT NULL,                -- PRE/RTU/INT/CON/PRD (combo)
    component_type      TEXT NOT NULL CHECK (component_type IN ('NVL','PRE','RTU','BTP','CON','PRD')),
    dinh_luong          DECIMAL(15,4) NOT NULL,
    dvt                 TEXT NOT NULL,
    ap_dung_bu          TEXT[],
    PRIMARY KEY (ma_mon, line_no)
);
```

## 14. Excel V2 sync staging

```sql
CREATE TABLE mdm_excel_sync_run (
    run_id              BIGSERIAL PRIMARY KEY,
    started_at          TIMESTAMP NOT NULL DEFAULT now(),
    completed_at        TIMESTAMP,
    drive_modified_time TIMESTAMP NOT NULL,
    status              TEXT NOT NULL CHECK (status IN ('RUNNING','SUCCESS','FAILED','PARTIAL')),
    rows_total          INT,
    rows_imported       INT,
    rows_skipped        INT,
    error_msg           TEXT
);

CREATE TABLE mdm_excel_sync_diff (
    run_id              BIGINT NOT NULL REFERENCES mdm_excel_sync_run(run_id),
    diff_id             BIGSERIAL,
    mdg_domain          TEXT NOT NULL,
    record_id           TEXT NOT NULL,
    diff_type           TEXT NOT NULL CHECK (diff_type IN ('NEW','UPDATE','DELETE','CONFLICT')),
    excel_state_json    JSONB,
    mdm_state_json      JSONB,
    review_status       TEXT NOT NULL DEFAULT 'PENDING' CHECK (review_status IN ('PENDING','ACCEPTED','REJECTED','MANUAL_FIX')),
    reviewed_by         TEXT,
    reviewed_at         TIMESTAMP,
    PRIMARY KEY (run_id, diff_id)
);
```

## 15. Migration sequence

```
001_baseline.sql            -- All tables above
002_seed_rbac.sql           -- Roles + MDP authority chain users (Thy, Quỳnh, Ms.Hồng, KTT, Hào, dvhiep, CFO)
003_seed_sites.sql          -- MDG-004 from project_bu_pc_structure_27062026 (BKL/TMM/Kupid/AFS/THOM/ADR1/ADR2)
004_seed_excel_v2_import.py -- Initial import from MASTER-DATA-V2.xlsx (all 7 MDGs)
```

## 16. References

- ADR-001 stack: `02-design/ADR-001-stack-choice.md`
- ADR-002 RBAC: `02-design/ADR-002-rbac-record-lock-design.md`
- MDP-SOPs (8 files): `core/docs/RAG_Library/01_NQH-SOPs/02_Nhat_Quang_Da_Lat/QA_ISO/Master_Data_Platform/`
- WI-101→105: `core/docs/RAG_Library/01_NQH-SOPs/07_Accounting_Handbook/07-FB-Costing/`
- Excel V2 schema: `MASTER-DATA-V2.xlsx` (em verified 16 tabs 28/06)
- Memory: `project_bu_pc_structure_27062026` (MDG-004 PC list) · `feedback_role_based_org_design`
