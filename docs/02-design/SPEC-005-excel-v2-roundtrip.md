---
spec_id: SPEC-005
spec_name: "Excel V2 Round-Trip — Schema Bind + Mapper + Sync Protocol"
spec_version: "1.0.0"
status: draft
tier: LITE
stage: "02"
category: architecture
owner: architect-itadmin-ceo-joint
created: 2026-06-28
last_updated: 2026-06-28
binding_sources:
  - "MASTER-DATA-V2.xlsx (Operations Excel V2 kiểm nghiệm 28/06)"
  - "MASTER-DATA-TEMPLATE-V2-CREATION-GUIDE-27062026.md (born-clean schema)"
  - "NQDL-MDP-SOP-000 §SLA (Excel sync workflow)"
---

# SPEC-005 — Excel V2 Round-Trip

## 1. Bind to Excel V2 schema (16 tabs)

Em verified via openpyxl 28/06. Tabs to ingest (active 7 MDGs + 3 ref + aux):

| Excel V2 tab | MDM table | Notes |
|---|---|---|
| `README` | (skip — metadata only) | |
| `MDG-002_Supplier` | `mdg_002_supplier` | |
| `MDG-003_Product` | `mdg_003_product` | Bridge cột `Ten_CukCuk_HoaDon` for CukCuk JOIN |
| `MDG-004_Site` | `mdg_004_site` | 6 rows; map sub-PC per `project_bu_pc_structure_27062026` |
| `MDG-008_Food` | `mdg_008_nvl` | RAW master + sơ chế dependency to Recipe_SoChe |
| `MDG-011_Cost_Master` | `mdg_011_cost_master` | |
| `MDG-012_Equipment` | `mdg_012_equipment` | |
| `CON_Consumables` | `mdg_008_nvl` (with `muc_so_che='CON'`) | 168 rows — store in NVL with CON marker |
| `PKG_Packaging` | `mdg_008_nvl` (with `muc_so_che='CON'` + cat='PKG') | OR separate table if business distinct (em recommend merge per @cto sharpening) |
| `Recipe_SoChe` | `mdg_008_nvl` (PRE rows joined to RAW via `ma_nvl_goc`) | **cột H = `don_gia_sau_so_che`** — FALLBACK only (per dtdanh empirical 30/06 + ADR-005 multi-source priority): primary price source = `fact_nvl_price_history` (VAT_INVOICE live + future BFLOW_PO). Excel cột-H used ONLY when no live price exists for that NVL at transaction_datetime |
| `Recipe_BTP` | `mdg_005_btp_recipe` + `mdg_005_btp` (yield_qty header) | **cột H = `don_gia_thanh_phan`** (BTP component price); yield_qty in `Recipe_BTP` header section |
| `Recipe_Mon_BOM` | `mdg_recipe_mon_bom` | PRD→component (PRE/RTU/INT/CON/PRD combo) |
| `MDG-002_Supplier` already listed | | |
| `MDG-004_Site` already listed | | |
| `ref_ValidationLists` | (in-memory validation table) | 134 cols × 12 rows — Cap_1/Cap_2/Tinh_Trang/etc. enum source |
| `Migration_Log` | (skip — Thy's worksheet log) | |
| `Validation_Report` | (skip — Thy's worksheet QA) | |

## 2. Round-trip data flow

### 2.1 Import flow (Excel V2 → MDM PostgreSQL)

```
[Thy/Quỳnh edit MASTER-DATA-V2.xlsx in Google Drive]
       │
       ▼ User clicks "Import Excel" in MDM UI
       │    OR
       ▼ Daily cron 06:00 polls Drive modifiedTime change
[MDM /excel/import endpoint]
       │
       │ multipart upload .xlsx (~350KB) OR Drive API fetch
       │
       ▼
[ExcelImportService] (Python module)
       │
       │ openpyxl read_only mode → 16 tabs
       │ Validate each tab schema (per §1 mapping)
       │ Build proposed CRUD ops vs current MDM state
       │
       ▼
[mdm_excel_sync_run row + mdm_excel_sync_diff rows]
       │
       │ status='RUNNING' → 'SUCCESS' or 'FAILED'
       │
       ▼
[Web UI sync_run detail view]
       │
       │ Steward reviews each diff (NEW/UPDATE/DELETE/CONFLICT)
       │ Bulk accept/reject OR per-row review
       │
       ▼
[Steward clicks "Commit accepted diffs"]
       │
       │ Apply diffs to mdg_* tables (transactional batch)
       │ Audit log entries per row
       │ Update Excel V2 sync watermark
```

### 2.2 Export flow (MDM PostgreSQL → Excel V2)

```
[Steward clicks "Export" OR scheduled daily 17:00]
       │
       ▼
[MDM /excel/export endpoint]
       │
       │ SELECT FROM mdg_* tables (filter: is_active, NOT deleted, latest version)
       │ Build openpyxl workbook matching born-clean v2 schema
       │ Preserve all 16 tab structure (even skip-tabs as empty)
       │
       ▼
[Generated .xlsx download]
       │
       │ Thy/Quỳnh save → Drive override
       │ OR push to Drive via service account (with backup of prev version)
       │
       ▼
[Excel V2 reflects MDM state]
```

### 2.3 Conflict resolution (CONFLICT diff_type)

When same record edited in BOTH Excel V2 AND MDM since last sync:
- `diff_type='CONFLICT'`
- UI shows side-by-side: Excel state · MDM state
- Steward picks one OR manually merges (input free-form combined state)
- Audit log records decision + reason

Anti-pattern: silent overwrite. Always surface conflict for human resolution.

## 3. Critical column mappings

### 3.1 Recipe_SoChe (per @mis Operations 28/06 kiểm nghiệm)

```
Excel cột     →  MDM column
A: Ma_NVL_Goc            → mdg_008_nvl.ma_nvl (RAW)
B: Ma_NVL_So_Che         → mdg_008_nvl.ma_nvl (PRE row)
C: Ten_NVL               → mdg_008_nvl.ten_nvl
D: Muc_So_Che            → mdg_008_nvl.muc_so_che (THO/INT/SC)
E: DVT_Nhap              → mdg_008_nvl.dvt_mua
F: DVT_Quy_Doi           → mdg_008_nvl.dvt_xuat
G: Ty_Le_Yield           → mdg_008_nvl.ty_le_yield
H: Don_Gia_Sau_So_Che    → mdg_008_nvl.don_gia_sau_so_che  (FALLBACK only — primary = fact_nvl_price_history.don_gia where price_source ∈ VAT_INVOICE/BFLOW_PO at transaction_datetime; per dtdanh empirical 30/06 §R-83)
I: Trang_Thai            → mdg_008_nvl.is_active
```

**FK convention**: PRE row's `ma_nvl_goc` populated from Excel cột A.

### 3.2 Recipe_BTP (per @mis Operations 28/06 + @cto §C.4 yield)

```
Excel cột     →  MDM column
A: Ma_BTP                → mdg_005_btp_recipe.ma_btp + mdg_005_btp.ma_btp (header)
B: Ten_BTP               → mdg_005_btp.ten_btp
C: Ma_Thanh_Phan         → mdg_005_btp_recipe.ma_component
D: Loai_Thanh_Phan       → mdg_005_btp_recipe.component_type (PRE/RTU/CON)
E: Ten_Thanh_Phan        → (auto-fetched from mdg_008_nvl.ten_nvl, not stored)
F: DVT                   → mdg_005_btp_recipe.dvt
G: Dinh_Luong            → mdg_005_btp_recipe.dinh_luong
H: Don_Gia_Thanh_Phan    → (NOT stored directly — recomputed from mdg_008_nvl.don_gia_sau_so_che)
J: Tong_Dinh_Luong_Thanh_Pham → mdg_005_btp.yield_qty   ⭐ CRITICAL (per @cto §C.4, anti-40%-error)
K: DVT_Thanh_Pham             → mdg_005_btp.yield_uom
L: Don_Gia_Sau_Che_Bien       → mdg_005_btp.don_gia_sau_che_bien  ⭐ UNIT-PRICE-FACT (per @mis rider #3 29/06; parallel NVL cột-K)
```

⚠️ **Positional drift V2 vs source — empirically verified 29/06 (post-Kimi-import)**: V2 Recipe_BTP has 12 cols (A-L). The 3 yield/price cols land at J/K/L (NOT L/M/N as in source `Định lượng BTP`). Source has additional rollup cols (Thành tiền, Tổng Chi Phí) between H and the yield-block; V2 omits these (per @mis "FLATTEN value, drop derived J/K" handoff guardrail). Parser MUST use V2 positions J/K/L, NOT source positions L/M/N.

**@mis rider #3 (29/06) — 3 columns Kimi đang import vào V2 (handoff 7abd463e):**

| V2 column added | Source col | Role |
|---|---|---|
| `Tong_Dinh_Luong_Thanh_Pham` | L | `yield_qty` (chia) |
| `DVT_Thanh_Pham` | M | `yield_uom` |
| `Don_Gia_Sau_Che_Bien` | N | BTP unit-cost-fact (= K÷L sanity, parallel với NVL cột-K) |

**Authoritative chain (interim Phase 1):**
- BTP cost lookup → use `mdg_005_btp.don_gia_sau_che_bien` direct (Excel-authoritative)
- Engine yield-divide compute → VALIDATION pattern (assert engine_recompute ≈ Excel cột-N ±tolerance)
- Phase 2 OR Excel cột-N stale → engine compute via full yield-divide (per SPEC-006 §6.2)

**Parser convention (per 29/06 empirical verify source `Định lượng BTP`)**: cột L+M values appear ONLY on the FIRST row of each BTP group (e.g. INT-0006 yield=1200gr in header row; subsequent component rows L12+ = NULL). Parser MUST:
1. Read L+M from header row → write ONCE to `mdg_005_btp.yield_qty + yield_uom` per BTP
2. Skip L+M on subsequent component rows (already captured)
3. Validate: every distinct `ma_btp` MUST have exactly 1 row with non-NULL L+M; if missing → emit anomaly `RECIPE_INVALID_REF (yield missing)` per SPEC-006 §6.2 BTP-yield test gate

This convention matches `mdg_005_btp` 1:N `mdg_005_btp_recipe` DDL design (SPEC-002 §mdg_005_btp).

### 3.3 Recipe_Mon_BOM

```
Excel cột     →  MDM column
A: RCP_ID                → (audit only, not stored as PK)
B: Ma_Mon                → mdg_recipe_mon_bom.ma_mon
C: Ten_Mon               → (auto-fetched from mdg_003_product, not stored)
D: Ma_Thanh_Phan         → mdg_recipe_mon_bom.ma_component
E: Loai_Thanh_Phan       → mdg_recipe_mon_bom.component_type (NVL/PRE/RTU/BTP/CON/PRD)
F: Ten_Thanh_Phan        → (auto-fetched, not stored)
G: DVT                   → mdg_recipe_mon_bom.dvt
H: Dinh_Luong            → mdg_recipe_mon_bom.dinh_luong
I: Ap_Dung_BU            → mdg_recipe_mon_bom.ap_dung_bu (array split by comma)
```

## 4. Validation rules (during import)

- Required fields present per MDG schema (§1 mapping)
- FK references resolve (e.g., `Ma_NVL_Goc` exists in MDG-008)
- Enum values valid (per ref_ValidationLists tab)
- Unit conversion consistent (per WI-102 yield formula)
- Duplicate detection (warn if `(ma_nvl, ten_nvl)` pair exists elsewhere)
- Negative numbers blocked (qty, price, yield)
- Yield ratio 0 < ty_le_yield ≤ 1

Validation errors collected into `mdm_excel_sync_diff` with `diff_type='CONFLICT'` and `review_status='MANUAL_FIX'` for steward attention.

## 5. Round-trip preservation rules

- **Format preservation**: openpyxl write mode writes Excel-2007+ .xlsx with born-clean v2 structure (header row 1, data row 2+)
- **Column order preservation**: per §1 mapping, MDM exports columns in same order Excel V2 expects
- **No formula injection**: write static values only (no formulas per @mis born-clean rule)
- **Tab order**: preserve original 16 tabs (even skip-tabs as empty placeholders)
- **README tab**: regenerate from MDM with version + export timestamp + counts
- **Migration_Log + Validation_Report**: regenerate per export (MDM authoritative)

## 6. Drive API integration (Phase 1 — service account)

- Service account: `core/ops/config/secrets/nqh-sheet-reader.json` (existing per em 25/06 ADR-018 ref)
- Drive file ID: `1K-SL4aeorrZtgvoTiOxarC_sNRewktAi` (per @mis pivot 28/06)
- Polling: cron 06:00 ICT daily; check `modifiedTime > last_synced_modified_time`
- Manual trigger: `/excel/import` accepts multipart upload (preferred for steward-driven import)
- Write-back: Phase 1 = manual download + Drive upload by Thy/Quỳnh (NO auto Drive write to avoid conflict with their workflow). Phase 2 = auto write to versioned backup `MASTER-DATA-V2-mdm-{timestamp}.xlsx` (ON-DEMAND).

## 7. Implementation handoff to @coder Ms. Quỳnh

- Library: `openpyxl` (em verified read works) + `pandas` (optional for transforms)
- Module: `mdm/services/excel_v2_service.py` (parser + diff builder + commit)
- Tests: synthetic .xlsx fixture per §3 mappings (golden file vs MDM expected state)
- Drive integration: reuse pattern from `apps/clickhouse/cukcuk/scraper/scraper.py` (service account auth)

## 8. References

- @mis Operations 28/06: kiểm nghiệm Excel V2 tạo + import
- `MASTER-DATA-TEMPLATE-V2-CREATION-GUIDE-27062026.md`
- `THY-CHECKLIST-BUILD-PILOT-MDG008-TAB-27062026.md`
- ADR-001 stack: `02-design/ADR-001-stack-choice.md`
- ADR-002 RBAC: `02-design/ADR-002-rbac-record-lock-design.md`
- SPEC-002 DDL: `02-design/SPEC-002-postgres-ddl.md`
- SPEC-003 OpenAPI §2.6 Excel endpoints: `02-design/SPEC-003-openapi-spec.md`
- ADR-005 (clickhouse/docs): EXCEL_V2 = price_source canonical interim
- Memory: `project_excel_master_data_ssot_transitional`
