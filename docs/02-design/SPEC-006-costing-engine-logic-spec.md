---
spec_id: SPEC-006
spec_name: "Costing-Engine LOGIC-SPEC (Language-Agnostic Pseudocode)"
spec_version: "1.0.0"
status: draft
tier: LITE
stage: "02"
category: architecture
owner: architect-itadmin-ceo-joint
created: 2026-06-28
last_updated: 2026-06-28
binding_sources:
  - "WI-101 §5.2 (per-NVL costing method) + WI-102 §1.2 (post-yield formula) + WI-103 (BTP) + WI-104 §4.1 (recipe COGS) + WI-105 (food cost %)"
  - "apps/clickhouse/docs/02-design/ADR-001 (per-bill freeze §R-37) + ADR-003 (honest-ceiling) + ADR-005 (price source priority)"
  - "apps/clickhouse/docs/02-design/SPEC-0010 §5 (proven algorithm)"
  - "@mis Phase 1 COGS-PASS PROVEN values (PRD-0173/0034/0181/0325/0009)"
  - "@cto sharpening 28/06: LOGIC-SPEC = durable IP, NOT Python-code"
---

# SPEC-006 — Costing-Engine LOGIC-SPEC

## 0. CRITICAL — Durable IP per @cto

> *Engine LOGIC-SPEC = durable IP (port-able to any-language: Go, Java, TypeScript, Rust). Python = REFERENCE-IMPL. If BFlow 2 financial = Go, engine rewrites in Go using THIS spec as authority — logic preserved, not binary-lift.*

This spec MUST be readable by reviewer WITHOUT Python knowledge. Pseudocode + math notation, language-neutral.

## 1. Engine surface (3 callable operations)

```
OP-1: compute_recipe_cost(ma: ID, transaction_datetime: TIMESTAMP) → Result<RecipeCost>
OP-2: freeze_bill_line(line: SalesDetailLine) → Result<FrozenCost>
OP-3: lookup_nvl_effective_price(ma_nvl: ID, at: TIMESTAMP) → Result<EffectivePrice>
```

All operations are PURE FUNCTIONS of inputs + price-history time-machine state. NO mutation of input state (idempotent + replayable).

## 2. OP-1: compute_recipe_cost (recursive)

### 2.1 Pseudocode (language-agnostic)

```
FUNCTION compute_recipe_cost(ma: ID, transaction_datetime: TIMESTAMP, depth: INT = 0):
    
    IF depth > MAX_DEPTH (=8):
        RAISE CircularRecipeError("Max recursion depth 8 exceeded for " + ma)
    
    LET prefix := SUBSTRING(ma, 0, 3)
    
    -- LEAF cases: PRE/RTU/CON have price directly (post-yield, per WI-102 §1.3)
    -- DO NOT recurse into RAW — yield already absorbed into don_gia_sau_so_che
    IF prefix IN ('PRE', 'RTU', 'CON'):
        LET price := lookup_nvl_effective_price(ma, transaction_datetime)
        IF price IS NULL:
            EMIT anomaly: NVL_NO_PRICE for ma at transaction_datetime
            RETURN RecipeCost{frozen_cost: NULL, components: [], missing_prices: [ma]}
        RETURN RecipeCost{
            frozen_cost: price.value_per_unit,
            unit: price.unit,
            components: [{ma: ma, qty: 1, unit_price: price.value_per_unit}],
            recipe_version: NULL  -- leaf has no recipe
        }
    
    -- BTP case: recursive into components, divide by yield
    IF prefix = 'INT':
        LET recipe_rows := SELECT FROM mdg_005_btp_recipe WHERE ma_btp = ma
        IF recipe_rows IS EMPTY:
            EMIT anomaly: MISSING_RECIPE for ma
            RETURN RecipeCost{frozen_cost: NULL, ...}
        
        LET btp_header := SELECT FROM mdg_005_btp WHERE ma_btp = ma
        LET yield_qty := btp_header.yield_qty
        
        IF yield_qty IS NULL OR yield_qty <= 0:
            EMIT anomaly: RECIPE_INVALID_REF (yield missing) for ma
            RETURN RecipeCost{frozen_cost: NULL, ...}
        
        LET total_component_cost := 0
        LET components := []
        FOR EACH row IN recipe_rows:
            LET sub := compute_recipe_cost(row.ma_component, transaction_datetime, depth + 1)
            IF sub.frozen_cost IS NULL:
                RETURN RecipeCost{frozen_cost: NULL, missing_prices: union(sub.missing_prices, ...)}
            total_component_cost := total_component_cost + (row.dinh_luong * sub.frozen_cost)
            components.append({ma: row.ma_component, qty: row.dinh_luong, unit_price: sub.frozen_cost, subtotal: row.dinh_luong * sub.frozen_cost})
        
        -- CRITICAL per WI-103 + @cto §C.4: divide by yield (anti-40%-error)
        LET unit_cost := total_component_cost / yield_qty
        
        RETURN RecipeCost{
            frozen_cost: unit_cost,
            unit: btp_header.yield_uom,
            components: components,
            recipe_version: btp_header.updated_at  -- audit trail
        }
    
    -- PRD (món bán) case: recursive into BOM
    IF prefix = 'PRD':
        LET recipe_rows := SELECT FROM mdg_recipe_mon_bom WHERE ma_mon = ma
        IF recipe_rows IS EMPTY:
            EMIT anomaly: MISSING_RECIPE for ma
            RETURN RecipeCost{frozen_cost: NULL, ...}
        
        LET total_cost := 0
        LET components := []
        FOR EACH row IN recipe_rows:
            LET sub := compute_recipe_cost(row.ma_component, transaction_datetime, depth + 1)
            IF sub.frozen_cost IS NULL:
                RETURN RecipeCost{frozen_cost: NULL, ...}
            total_cost := total_cost + (row.dinh_luong * sub.frozen_cost)
            components.append({...})
        
        RETURN RecipeCost{
            frozen_cost: total_cost,  -- món-level cost = Σ(component qty × unit price)
            unit: 'phần',  -- per món, per WI-104
            components: components,
            recipe_version: latest_updated_at(recipe_rows)
        }
    
    -- RAW case: SHOULD NEVER appear in recipe (per WI-102 §1.3)
    -- Recipes use PRE/RTU/INT/CON only; RAW is upstream of PRE
    IF prefix = 'RAW':
        RAISE InvalidRecipeRefError("RAW " + ma + " should not appear in recipe — use PRE instead")
    
    RAISE UnknownPrefixError("Unknown prefix for " + ma)
```

### 2.2 Invariants

- **I.1** (Termination): `MAX_DEPTH=8` prevents infinite recursion; cycle-detection via depth-count
- **I.2** (Determinism): same inputs (ma, transaction_datetime) + same DB snapshot = same output
- **I.3** (Idempotency): re-run with same inputs = same result, no mutation
- **I.4** (Leaf-stop): PRE/RTU/CON terminate recursion (NO descent to RAW — anti double-yield)
- **I.5** (Yield divide): INT computes `total ÷ yield_qty` (anti 40% error per @cto §C.4)
- **I.6** (Combo support): PRD can contain PRD (combo) per `mdg_recipe_mon_bom.component_type='PRD'`

## 3. OP-2: freeze_bill_line

### 3.1 Pseudocode

```
FUNCTION freeze_bill_line(line: SalesDetailLine):
    
    -- §R-37 INVARIANT: use line.transaction_datetime, NEVER now()
    -- Per ADR-001 / SPEC-001 §A.1 correctness gate
    LET tx_dt := line.transaction_datetime  -- MUST come from bill, NOT current time
    
    LET recipe := compute_recipe_cost(line.ma_mon, tx_dt)
    
    IF recipe.frozen_cost IS NULL:
        -- Missing prices OR recipe — write NULL cost with provenance
        RETURN FrozenCost{
            bill_line_id: line.id,
            cost_price: NULL,
            cost_basis: 'BACKFILL_NO_PRICE_AVAILABLE',
            costing_method_basis: 'POINT_IN_TIME_PRICE',
            price_resolution_timestamp: tx_dt,
            recipe_version: NULL,
            anomaly: missing_prices
        }
    
    -- frozen_cost = unit cost × quantity sold
    LET total_cost := recipe.frozen_cost * line.quantity
    
    RETURN FrozenCost{
        bill_line_id: line.id,
        cost_price: total_cost,
        cost_basis: 'POINT_IN_TIME',
        costing_method_basis: 'POINT_IN_TIME_PRICE',
        price_resolution_timestamp: tx_dt,  -- explicit audit
        recipe_version: recipe.recipe_version,
        components_snapshot: recipe.components  -- per-line cost breakdown
    }
```

### 3.2 Invariants

- **I.7** (§R-37 gate): MUST use `line.transaction_datetime` for price lookup, MUST NOT use `now()` or `current_timestamp()` in any predicate
- **I.8** (Immutability): once frozen row exists for `bill_line_id`, NEVER update — only MANUAL_OVERRIDE creates new row linked via `override_of_id`
- **I.9** (Provenance honesty): `cost_basis` reflects actual freeze provenance (POINT_IN_TIME for live event-hook, BACKFILL_APPROX for historical, BACKFILL_NO_PRICE_AVAILABLE for missing-data)

## 4. OP-3: lookup_nvl_effective_price (multi-source priority)

### 4.1 Pseudocode (per ADR-005 priority resolution)

```
FUNCTION lookup_nvl_effective_price(ma_nvl: ID, at: TIMESTAMP):
    
    -- Query price-history time-machine with priority-resolution
    -- Per ADR-005 §4: VAT_INVOICE > EXCEL_V2 > BACKFILL_INITIAL (pre-BFlow 2)
    --                  Post-BFlow 2: BFLOW_PO > VAT_INVOICE
    
    -- Priority ordering (lowest number = highest priority):
    LET priority_order := IF current_era = 'PRE_BFLOW2' THEN
        ['VAT_INVOICE': 1, 'EXCEL_V2': 2, 'BACKFILL_INITIAL': 3, 'MANUAL_OVERRIDE': 0]
    ELSE  -- POST_BFLOW2
        ['BFLOW_PO': 1, 'VAT_INVOICE': 2, 'EXCEL_V2': 3, ..., 'MANUAL_OVERRIDE': 0]
    
    LET candidates := SELECT 
        ma_nvl, don_gia, dvt, price_source, valid_from, valid_to
    FROM fact_nvl_price_history
    WHERE ma_nvl = $ma_nvl
      AND valid_from <= $at
      AND valid_to > $at   -- exclusive open interval
    
    IF candidates IS EMPTY:
        EMIT anomaly: NVL_NO_PRICE for ma_nvl at $at
        RETURN NULL
    
    -- Select highest-priority source (lowest priority_order value)
    LET selected := candidates ORDER BY priority_order[price_source] ASC LIMIT 1
    
    RETURN EffectivePrice{
        value_per_unit: selected.don_gia,
        unit: selected.dvt,
        source: selected.price_source,
        valid_from: selected.valid_from,
        lookup_at: $at  -- audit trail
    }
```

### 4.2 Invariants

- **I.10** (Source-blind query result): callers consume `EffectivePrice` without branching on `source` (priority resolution opaque)
- **I.11** (§R-37 inheritance): `$at` parameter MUST come from bill `transaction_datetime`, NOT `now()`

## 5. Anomaly emission (consumed by anomaly cron — per COGS pipeline FR-006)

Engine MUST emit (write to queue OR direct INSERT) these anomaly types when conditions met:

| Anomaly | Trigger |
|---|---|
| `MISSING_RECIPE` | OP-1 recipe_rows empty for non-leaf prefix |
| `NVL_NO_PRICE` | OP-3 no candidates for (ma_nvl, at) |
| `PRD_NOT_IN_MENU` | PRD code appears in fact_sales but not mdg_003_product |
| `RECIPE_INVALID_REF` | recipe row's `ma_component` not in mdg_008_nvl OR yield_qty NULL on BTP |
| `DRIFT_QTY` | actual_qty_consumed_30d / recipe_estimated_qty_30d > 1.50 |
| `NVL_BFLOW_NO_MAP` | (post-BFlow 2 era) Bflow PO unmapped |
| `NVL_VAT_NO_MAP` | VAT invoice unmapped via crosswalk |

## 6. Test fixtures (language-neutral)

### 6.1 §R-37 gate test (PRIORITY: BLOCKS production deploy)

```
GIVEN fact_nvl_price_history has TWO rows for TEST-NVL-001:
  Row 1: valid_from='2026-06-01', valid_to='2026-06-15', don_gia=100
  Row 2: valid_from='2026-06-15', valid_to='9999-12-31', don_gia=200
GIVEN synthetic bill_line: transaction_datetime='2026-06-10 14:23', ma_mon='TEST-PRD-001' (recipe=1 unit of TEST-NVL-001)
WHEN OP-2 freeze_bill_line invoked at runtime 2026-06-25 (AFTER price change)
THEN result.cost_price MUST = 100 (price at transaction_datetime, NOT 200)
  AND result.price_resolution_timestamp = '2026-06-10 14:23'
  AND impl code review confirms NO use of now()/current_timestamp() in price lookup
PASS REQUIRED — fail = production deploy BLOCKED per SPEC-001 §A.1
```

### 6.2 BTP yield test (CRITICAL anti-40%-error)

```
GIVEN mdg_005_btp INT-TEST-001 with yield_qty=1200, yield_uom='gr'
GIVEN mdg_005_btp_recipe: 2 components, total_dinh_luong contributing cost 240,000đ
WHEN OP-1 compute_recipe_cost('INT-TEST-001', any_dt)
THEN result.frozen_cost = 240000 / 1200 = 200 đ/gr
  AND NOT 240000 / 2000 (anti-error if yield bypassed)
PASS REQUIRED per SPEC-001 §A.4
```

### 6.3 PRD combo test (CRITICAL anti-missing-branch)

```
GIVEN mdg_003_product PRD-COMBO-TEST contains 3 sub-PRDs (Combo món lồng món)
GIVEN each sub-PRD has compute_recipe_cost result: 50,000đ, 80,000đ, 30,000đ
GIVEN mdg_recipe_mon_bom: 1 unit each sub-PRD
WHEN OP-1 compute_recipe_cost('PRD-COMBO-TEST', any_dt)
THEN result.frozen_cost = 50000 + 80000 + 30000 = 160,000đ
  AND recursion handles PRD→PRD branch correctly
PASS REQUIRED per SPEC-001 §A.6
```

### 6.4 Cycle detection test

```
GIVEN mdg_recipe_mon_bom has cycle: PRD-A → PRD-B → PRD-A
WHEN OP-1 compute_recipe_cost('PRD-A', any_dt)
THEN raises CircularRecipeError at depth=8 (recursion limit)
PASS REQUIRED per SPEC-001 §A.3
```

### 6.5 @mis Phase 1 PROVEN values alignment

```
GIVEN MDM PostgreSQL synced from MASTER-DATA-V2.xlsx (latest)
GIVEN @mis PROVEN ground-truth (5/5 empirical confirm 28/06): PRD-0173=60,863đ, PRD-0034=8,030đ, PRD-0181=8,333đ, PRD-0325=25,666đ, **PRD-0009 (Gà Hấp Gừng Rừng) = 116,034đ** (gap closed in V2 per @mis 28/06)
WHEN OP-1 compute_recipe_cost(prd_code, current_time) for each
THEN engine output matches PROVEN values within ±0.5% rounding tolerance
PASS REQUIRED per SPEC-001 §A.7 (Excel-COGS Phase 1 method alignment)
```

## 7. Implementation handoff to @coder Ms. Quỳnh

- Reference-impl module: `mdm/services/costing_engine.py`
- Function signatures match OP-1/OP-2/OP-3 per §1
- Test file: `tests/unit/test_costing_engine_logic_spec.py` (each invariant I.1-I.11 + each §6 fixture)
- §R-37 gate test = blocking CI (deploy fail if test fails)
- BTP yield + PRD combo + cycle detection = blocking
- @mis PROVEN values alignment = nightly CI smoke test

## 8. References

- SPEC-001 engine-review checklist: `02-design/SPEC-001-engine-review-checklist.md`
- WI-101→105: `core/docs/RAG_Library/01_NQH-SOPs/07_Accounting_Handbook/07-FB-Costing/`
- COGS pipeline ADR-001 (§R-37): `apps/clickhouse/docs/02-design/ADR-001-Per-Bill-PointInTime-Freeze.md`
- COGS pipeline ADR-003 (honest-ceiling): same dir/ADR-003-Costing-Method-Interim-vs-Bflow2.md
- COGS pipeline ADR-005 (price source priority): same dir/ADR-005-VAT-Interim-Price-Source.md
- SPEC-0010 §5 (proven algorithm): `apps/clickhouse/docs/02-design/SPEC-0010-Data-Flow-Sheets-to-ClickHouse.md`
- @mis Operations 28/06 PROVEN values
- Memory: `feedback_verify_data_presence_not_just_columns` · `project_excel_master_data_ssot_transitional`
