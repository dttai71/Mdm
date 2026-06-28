---
spec_id: SPEC-001
spec_name: "Durable Costing-Engine LOGIC-SPEC Review Checklist (Language-Agnostic, Port-Ready)"
spec_version: "1.1.0"
status: RATIFIED
tier: LITE
stage: "02"
category: architecture
owner: architect-itadmin-ceo-joint
created: 2026-06-28
last_updated: 2026-06-28
authority: "@cto approve green-light pull-forward 28/06; @mis RATIFY 28/06 empirically-backed (5/5 PROVEN values match)"
ratify_evidence: "@mis independent engine-method reimpl from V2 — 5/5 PRD match: 0173=60863 · 0034=8030 · 0181=8333 · 0325=25666 · 0009=116034"
---

## ⚠️ HONEST-CEILING per @mis RATIFY rider #1 (28/06)

This checklist validates **engine-LOGIC correctness** (logic-validate per §A-§G). It does **NOT** signoff **VALUE-correctness** (price/yield audited-correct).

- **What this checklist proves**: engine impl matches @mis-method LOGIC ±0.5% (logic-correct)
- **What this checklist does NOT prove**: source data (prices/yields) is audited-correct
- **Value-truth layer** = KTT-materiality-spot-check + VAT-price-loop (per ADR-005 Stage 2) — SEPARATE concern
- **@mis-PROVEN values = best-available** (DWH-died, manual-recompute) — NOT auditor-confirmed
- "Engine khớp @mis-values" ≠ "COGS audited-correct"

Engine-review = LOGIC gate only. Production COGS-audit = downstream KTT + VAT-loop scope.


# SPEC-001 — Engine-Review Checklist (LOGIC-SPEC scope, port-ready)

## 0. Critical sharpening per @cto verdict 28/06

> *Durable IP = LOGIC-SPEC (method / formula / rules), NOT Python-code. Python = reference-impl only. If BFlow 2 financial = Go, Python-engine = rewrite (logic preserved, NOT binary-lift).*

→ This checklist validates **LOGIC-SPEC** (port-able any-language), NOT Python-impl specifics.

**Engine-review PASS criteria**: every section §A-§G must score PASS independently of implementation language. Failure → BUILD blocked until LOGIC-SPEC defects fixed.

## 1. Scope

This checklist gates the durable-costing-engine BUILD trigger per Decision Log `785ffd07`. Excludes:
- ❌ MVP-lean web-app (RBAC/record-lock/UI) — separate checklist post engine-review PASS
- ❌ Python-impl specifics (style, libs, build tool) — those are implementation concerns, not LOGIC-SPEC
- ❌ Frontend (Vue) — throwaway at sunset per ADR-001 §5

Scope = engine LOGIC-SPEC only: formulas, rules, invariants, schemas, audit semantics, anomaly classes, sunset portability.

## 2. Alignment with Excel-COGS Phase 1 PROVEN method

Per @mis §A0.5 28/06: Phase 1 PROVEN end-to-end with born-clean v2 + **cột-K (`Don_Gia_Sau_So_Che` — đơn-giá-đúng-đơn-vị post-sơ-chế)** unit-fix → real gross-profit numbers (Cơm Lam 21,970đ 73% · Cá tầm 138,137đ 69% · Rau lẩu 51,667đ 86%).

LOGIC-SPEC MUST encode this proven-method. Any drift from Phase 1 method = checklist FAIL.

> **Per @mis rider #2 (28/06)**: cột-K = `Don_Gia_Sau_So_Che` (đơn giá sau sơ chế) is canonical unit-price. cột-H = đơn-giá-nhập per-kg = the column that CAUSED the 60M-bug if used directly. SPEC-006 §2.1 LOGIC pseudocode correctly uses cột-K (via `don_gia_sau_so_che` field). Em correct §2 prose here per @mis rider #2.

## 3. Engine-review checklist

### §A. CORRECTNESS INVARIANTS

| # | Invariant | Source | PASS criterion |
|---|---|---|---|
| A.1 | Per-bill cost frozen at `transaction_datetime` (NEVER `now()`) | COGS pipeline ADR-001 §R-37 | Synthetic test: bill at T1, price changed at T2>T1, engine MUST return price-at-T1 cost. Implementation MUST NOT use now()/current_timestamp/system-time-fn in price lookup predicate |
| A.2 | Bill cost immutability post-freeze | COGS FR-009 | After cost row written, NO mutation allowed. Only MANUAL_OVERRIDE escape (new row + audit link, NOT in-place update). Test: re-run freeze on existing bill = idempotent (0 mutations) |
| A.3 | Recipe roll-up depth limit = 8 + cycle-detection | SPEC-0005 §2.4 | Synthetic recipe with depth-9 → engine MUST raise CircularRecipeError. Cyclic ref (A→B→A) → MUST detect + reject |
| A.4 | BTP yield divide (per WI-103 §C.4) | SPEC-0010 §C.4 | INT recipe formula: `btp_unit_cost = Σ(component_qty × unit_price) ÷ yield_qty`. NOT skip yield (Excel cột H/L). Test: INT-0006 NVL 2000g → BTP 1200g → unit_cost = total_cost / 1200, NOT / 2000 |
| A.5 | PRE/RTU/CON leaf-stop | WI-102 §1.3 anti-double-yield | Engine MUST stop recursion at PRE/RTU/CON (price already post-yield). Test: PRE recipe lookup → return price directly, MUST NOT descend to RAW |
| A.6 | PRD→PRD combo branch | SPEC-0010 §C.2 | Combo PRD (3 exist: PRD-0425/0426/0427) → engine MUST recurse into component PRDs. Test: Combo cost = Σ(sub-món cost) |
| A.7 | Excel-COGS Phase 1 method match | @mis §A0.5 PROVEN | Engine output for known món matches Phase 1 PROVEN values (Cơm Lam GP 21,970 · Cá tầm 138,137 · Rau lẩu 51,667) within ±0.5% rounding tolerance |

### §B. PROVENANCE (honest-ceiling)

| # | Invariant | PASS criterion |
|---|---|---|
| B.1 | `cost_basis` enum on every cost row | enum: POINT_IN_TIME / BACKFILL_APPROX / BACKFILL_NO_PRICE_AVAILABLE / MANUAL_OVERRIDE. NULL = FAIL |
| B.2 | `costing_method_basis` enum | POINT_IN_TIME_PRICE (interim) reserved; TRUE_PER_METHOD only post-BFlow 2 |
| B.3 | `price_source` enum on price-history rows | EXCEL_SSOT (deprecated) / EXCEL_V2 / BACKFILL_INITIAL / BFLOW_PO / VAT_INVOICE / MANUAL_OVERRIDE per ADR-005 |
| B.4 | Multi-source priority resolution | When multiple sources have prices overlapping same `(ma_nvl, time)`: priority VAT_INVOICE > EXCEL_V2 > BACKFILL_INITIAL (pre-BFlow 2); BFLOW_PO > VAT_INVOICE (post-BFlow 2) |
| B.5 | Source-blind query layer | Dashboard/agent queries MUST consume "effective price" view (priority-resolved), NOT depend on source enum value. Test: swap source for same ma_nvl/time → query result unchanged |
| **B.6** | **LOGIC-correct vs VALUE-correct distinction (per @mis rider #1)** | Checklist validates engine-LOGIC matches @mis-method ±0.5%. Does NOT signoff VALUE-correctness of source prices/yields. Value-truth = KTT-materiality-spot-check + VAT-price-loop (separate layer, per ADR-005 Stage 2). Engine-review PASS ≠ "COGS audited-correct". @bod/reports MUST NOT claim audit-grade COGS based on engine-review PASS alone — only "method-correct per @mis-PROVEN at YYYY-MM-DD" |

### §C. AUDIT TRAIL

| # | Requirement | PASS criterion |
|---|---|---|
| C.1 | Every freeze writes immutable row | `fact_sales_detail_with_cost` row contains: `cost_price`, `cost_basis`, `costing_method_basis`, `price_resolution_timestamp` (= transaction_datetime), `recipe_version`, `override_of_id` (NULL for live), `source_system` |
| C.2 | Every NVL price change logged | `fact_master_data_change_log_daily` captures `change_type` (PRICE_CHANGE / NEW_NVL / etc.), `business_key`, `old_value`, `new_value`, `detected_at`, `pct_delta`, `approval_required_tier` |
| C.3 | MANUAL_OVERRIDE explicit audit | Override row inserted (NOT in-place update) with `override_of_id` link to original; matching audit log entry with `change_type='COST_MANUAL_OVERRIDE'`, `approver`, `justification`, `original_cost`, `override_cost` |
| C.4 | Audit retention ≥ 7 years | Schema constraint + retention policy documented; accounting compliance |

### §D. IDEMPOTENCY

| # | Requirement | PASS criterion |
|---|---|---|
| D.1 | ReplacingMergeTree dedup verified | Re-run backfill OR re-trigger freeze for existing bill = 0 dup rows |
| D.2 | Event-hook idempotent | Multiple fires for same invoice (network retry) = 1 row final |
| D.3 | OPTIMIZE FINAL stable | After OPTIMIZE TABLE FINAL, row count matches pre-optimize dedup-expected count |

### §E. PORTABILITY (LOGIC-SPEC language-agnostic — durable IP)

| # | Requirement | PASS criterion |
|---|---|---|
| **E.1** | **LOGIC-SPEC documented INDEPENDENT of Python code** | Formulas + rules + invariants expressed as pseudocode + math notation (NOT Python-syntax). Reviewer with NO Python knowledge MUST understand engine semantics from spec alone |
| **E.2** | Schema portable to any RDBMS | DDL uses ANSI SQL types where possible; ClickHouse-specific (ReplacingMergeTree, Enum8) annotated as "engine-specific, replace with equivalent in target" |
| **E.3** | source_system enum present on every dim/fact row | EXCEL_SSOT / EXCEL_V2 / BFLOW2 / VAT_INVOICE — flips at sunset, schema unchanged |
| **E.4** | Target tables same logical model at sunset | BFlow 2 ports = consume same dim/fact tables; only `source_system` flips, no schema migration |
| **E.5** | Engine logic source-blind | Engine reads dim/fact tables without filtering on source_system. Switching source at sunset = drop-in |
| **E.6** | Pseudocode for compute_recipe_cost | LOGIC-SPEC includes pseudocode block (see SPEC-0010 §5 algorithm — language-agnostic). Any-language impl MUST match pseudocode semantics |
| **E.7** | Test fixtures language-neutral | Test fixtures = SQL seed + expected output CSV (NOT Python pytest-only). Other-language ports re-run same fixtures to validate |
| **E.8** | Decision tree for transformation rules | SO_CHE vs PASSTHROUGH vs CUSTOM (per ADR-004 §4.3) documented as decision tree, not Python if/elif |

### §F. ANOMALY DETECTION

| # | Requirement | PASS criterion |
|---|---|---|
| F.1 | 5 base anomaly classes | MISSING_RECIPE / NVL_NO_PRICE / PRD_NOT_IN_MENU / RECIPE_INVALID_REF / DRIFT_QTY (per FR-006 SPEC-0002) |
| F.2 | NVL_BFLOW_NO_MAP class | Bflow PO arrives with unmapped `ma_nvl_bflow` (per ADR-004 §4.4) |
| F.3 | NVL_VAT_NO_MAP class | VAT invoice arrives with unmapped `ten_hang` (per ADR-005 fold; em propose) |
| F.4 | Telegram routing tier-correct | <5% → Kế toán kho · 5-10% → Mua hàng Mgr · >±10% → CFO (per WI-105 §5.1). Daily digest = Thy + dtdanh ONLY (D22 distraction-shield, NOT CEO unless escalate) |
| F.5 | Anomaly severity scales (per WI-102 §6.1) | ±5% Monitor · ±5-10% Investigate · >±10% Escalate. Enum: LOW/MEDIUM/HIGH/CRITICAL |

### §G. SDLC 6.4.0 COMPLIANCE

| # | Requirement | PASS criterion |
|---|---|---|
| G.1 | §R-37 test fixture EXISTS + PASSES | Test_freeze_r37 covers A.1 invariant; fail = production deploy BLOCKED |
| G.2 | §R-71 service account discipline | rotation cadence ≤90d, registry entry, audit |
| G.3 | All design docs cross-referenced | LOGIC-SPEC cites COGS pipeline ADR-001/003/004/005 + SPEC-0004/0005 + VatDownload ADR-017/018 + INT-001/002 |
| G.4 | AmendmentC L1-L5 drift-lessons alignment | L1 govern-proportional · L2 evidence-of-need (BUILD gated, not speculative) · L3 anti-spiral (gov:code ratio monitored) · L4 outcome KPI (0 corruption events/month) · L5 dogfood post-MVP |
| G.5 | Mental Model #9 Demand Before Surface | Engine surfaces (API endpoints, dashboards, agent queries) each have daily-user + daily-job + re-eval trigger documented |

## 4. PASS/FAIL outcome

- ALL §A-§G items PASS → engine-review PASS → unlocks durable-engine BUILD (per Decision Log build-sequence step 1)
- ANY §A.1-A.7 FAIL → engine LOGIC defect → fix BEFORE re-review (correctness blocker)
- ANY §B/§C FAIL → audit/provenance defect → fix BEFORE re-review (compliance blocker)
- §D/§F FAIL → operational defect → may proceed with documented exception + remediation plan (review-er judgment)
- §E FAIL → portability defect → fix BEFORE re-review (sunset risk = unacceptable per Decision Log "0-waste port")
- §G FAIL → framework non-compliance → fix or formal waiver per @mis governance

## 5. Review process

1. @itadmin (em) DRAFT (this doc) → CO-AUTHOR with CEO per @architect joint role
2. @mis RATIFY (governance authority) → checklist becomes BINDING gate
3. dtdanh / Ms. Quỳnh implements per LOGIC-SPEC
4. Engine-review session: @cto + @mis + em walk through §A-§G against implementation (live test scenarios)
5. Per-item PASS/FAIL recorded at `05-test/engine-review-evidence-YYYY-MM-DD.md`
6. PASS verdict → @cto-CC sign-off → BUILD unlocks
7. FAIL verdict → defect log → fix cycle → re-review

## 6. References

- @cto verdict 28/06: sharpening LOGIC-SPEC vs Python-impl
- Decision Log `785ffd07`: build-sequence (durable-engine FIRST, MVP-lean SECOND)
- COGS pipeline ADR-001: `apps/clickhouse/docs/02-design/ADR-001-Per-Bill-PointInTime-Freeze.md` (§R-37)
- COGS pipeline ADR-003: same dir/ADR-003-Costing-Method-Interim-vs-Bflow2.md (honest-ceiling)
- COGS pipeline ADR-004: same dir/ADR-004-Bflow-Price-Source-Integration.md
- COGS pipeline ADR-005: same dir/ADR-005-VAT-Interim-Price-Source.md (multi-source priority)
- SPEC-0004: same dir/SPEC-0004-Schema-DDL-Migrations-35-38.md (enum definitions)
- SPEC-0005: same dir/SPEC-0005-Python-Module-Design.md (compute_recipe_cost reference-impl; LOGIC-SPEC in SPEC-0010)
- SPEC-0010: same dir/SPEC-0010-Data-Flow-Sheets-to-ClickHouse.md (§5 language-agnostic algorithm)
- @mis §A0.5 PROVEN values: HANDOFF-ITADMIN-COGS-INGEST-TEST-28062026.md
- SDLC 6.4.0 AmendmentC: `/home/nqh/shared/.sdlc-framework/09-Continuous-Improvement/AmendmentC-Drift-Lessons-Catalog.md`
- Memory anchors: `feedback_pull_forward_preaudit_optimization` (pull-forward) · `project_excel_master_data_ssot_transitional`
