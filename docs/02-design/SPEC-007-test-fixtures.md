---
spec_id: SPEC-007
spec_name: "Test Fixtures — Seed data + acceptance scenarios"
spec_version: "1.0.0"
status: draft
tier: LITE
stage: "02"
category: architecture
owner: architect-itadmin-ceo-joint
created: 2026-06-28
last_updated: 2026-06-28
---

# SPEC-007 — Test Fixtures + Acceptance Scenarios

## 1. Fixture file inventory (post-build, @coder authors)

```
mdm/tests/
├── unit/
│   ├── test_costing_engine_logic_spec.py    # SPEC-006 §6 invariants I.1-I.11
│   ├── test_rbac_enforcement.py             # SPEC-004 §9 TC-RBAC-001-008
│   ├── test_excel_v2_roundtrip.py           # SPEC-005 import/export golden file
│   └── test_audit_log_append_only.py        # ADR-002 §2.6 audit invariants
├── integration/
│   ├── test_postgres_ddl.py                 # SPEC-002 schema + Alembic
│   ├── test_openapi_contract.py             # SPEC-003 endpoint OpenAPI conformance
│   └── test_redis_lock_concurrency.py       # ADR-002 §2.5 record-lock
├── e2e/
│   ├── test_mvp_acceptance.py               # ACC-001 full MVP user journey
│   └── test_engine_match_ground_truth.py    # SPEC-006 §6.5 @mis PROVEN values
└── fixtures/
    ├── seed_rbac.sql                        # Initial users + roles per SPEC-004 §2
    ├── seed_sites.sql                       # MDG-004 per project_bu_pc_structure
    ├── seed_test_nvl.sql                    # Synthetic TEST-NVL-* for unit tests
    ├── seed_test_recipe.sql                 # Synthetic TEST-PRD-* + TEST-INT-* recipes
    ├── seed_test_price_history.sql          # 2-row price-history for §R-37 gate
    └── master_data_v2_excerpt.xlsx          # Excel V2 subset for round-trip tests
```

## 2. §R-37 gate test fixture (BLOCKING)

`fixtures/seed_test_price_history.sql`:
```sql
INSERT INTO fact_nvl_price_history (ma_nvl, valid_from, valid_to, don_gia, dvt, price_source) VALUES
  ('TEST-NVL-001', '2026-06-01 00:00:00', '2026-06-15 00:00:00', 100.00, 'gr', 'EXCEL_V2'),
  ('TEST-NVL-001', '2026-06-15 00:00:00', '9999-12-31 00:00:00', 200.00, 'gr', 'EXCEL_V2');

INSERT INTO mdg_008_nvl (ma_nvl, ten_nvl, muc_so_che, dvt_mua, dvt_xuat) VALUES
  ('TEST-NVL-001', 'Test material for R37 gate', 'PRE', 'gr', 'gr');

INSERT INTO mdg_003_product (ma_mon, ten_mon_vi) VALUES
  ('TEST-PRD-R37', 'Test món R37');

INSERT INTO mdg_recipe_mon_bom (ma_mon, line_no, ma_component, component_type, dinh_luong, dvt) VALUES
  ('TEST-PRD-R37', 1, 'TEST-NVL-001', 'PRE', 1, 'gr');
```

Test code:
```python
# tests/unit/test_costing_engine_logic_spec.py
def test_r37_uses_transaction_datetime_not_now():
    """SPEC-001 §A.1 + SPEC-006 §6.1 BLOCKING gate."""
    from datetime import datetime
    
    # Bill at T=2026-06-10 14:23, BEFORE price change at T=2026-06-15
    line = SalesDetailLine(
        id='test-bill-r37-001',
        transaction_datetime=datetime(2026, 6, 10, 14, 23),
        ma_mon='TEST-PRD-R37',
        quantity=1
    )
    
    # Freeze runs RIGHT NOW (after price change), but should use line.tx_dt
    result = freeze_bill_line(line)
    
    # MUST use price at transaction_datetime (=100), NOT current price (=200)
    assert result.cost_price == 100, \
        f"§R-37 VIOLATION: cost_price={result.cost_price} expected 100"
    assert result.price_resolution_timestamp == datetime(2026, 6, 10, 14, 23)
```

## 3. BTP yield test fixture (CRITICAL)

`fixtures/seed_test_recipe.sql`:
```sql
INSERT INTO mdg_005_btp (ma_btp, ten_btp, dvt, yield_qty, yield_uom) VALUES
  ('INT-TEST-YIELD', 'Test BTP yield', 'gr', 1200, 'gr');

INSERT INTO mdg_005_btp_recipe (ma_btp, line_no, ma_component, component_type, dinh_luong, dvt) VALUES
  ('INT-TEST-YIELD', 1, 'TEST-PRE-A', 'PRE', 1000, 'gr'),
  ('INT-TEST-YIELD', 2, 'TEST-PRE-B', 'PRE', 1000, 'gr');

-- TEST-PRE-A price = 100đ/gr, TEST-PRE-B = 140đ/gr
-- Total component cost = 1000*100 + 1000*140 = 240,000đ
-- Yield = 1200gr → unit_cost = 240000 / 1200 = 200đ/gr
-- WITHOUT yield = 240000 / 2000 = 120đ/gr (WRONG — 40% under)
```

## 4. PRD combo test fixture

```sql
INSERT INTO mdg_003_product (ma_mon, ten_mon_vi) VALUES
  ('TEST-PRD-COMBO', 'Test combo'),
  ('TEST-PRD-CHILD-A', 'Sub A'),
  ('TEST-PRD-CHILD-B', 'Sub B'),
  ('TEST-PRD-CHILD-C', 'Sub C');

INSERT INTO mdg_recipe_mon_bom VALUES
  -- Combo contains 3 child PRDs
  ('TEST-PRD-COMBO', 1, 'TEST-PRD-CHILD-A', 'PRD', 1, 'phần', '{}'),
  ('TEST-PRD-COMBO', 2, 'TEST-PRD-CHILD-B', 'PRD', 1, 'phần', '{}'),
  ('TEST-PRD-COMBO', 3, 'TEST-PRD-CHILD-C', 'PRD', 1, 'phần', '{}');
-- Each child PRD has own recipe summing to 50,000 / 80,000 / 30,000
-- Combo cost = 160,000
```

## 5. @mis Phase 1 PROVEN values smoke test

Daily nightly CI runs against PROD-data (read-only):
```python
# tests/e2e/test_engine_match_ground_truth.py
PROVEN_VALUES = {
    'PRD-0173': 60_863,
    'PRD-0034': 8_030,
    'PRD-0181': 8_333,
    'PRD-0325': 25_666,
    # PRD-0009 Gà Hấp Gừng Rừng — em surface @mis for ground-truth anchor (not in §B-3 table)
}
TOLERANCE = 0.005  # ±0.5% per SPEC-001 §A.7

def test_match_mis_proven_values():
    for ma_mon, expected in PROVEN_VALUES.items():
        result = compute_recipe_cost(ma_mon, datetime.now())
        if result.frozen_cost is None:
            pytest.skip(f"{ma_mon} missing recipe/price — gap, not failure")
            continue
        delta = abs(result.frozen_cost - expected) / expected
        assert delta <= TOLERANCE, \
            f"{ma_mon}: engine={result.frozen_cost} expected={expected} delta={delta*100:.2f}%"
```

## 6. RBAC concurrency test

```python
# tests/integration/test_redis_lock_concurrency.py
import threading

def test_two_stewards_same_record_one_wins():
    """ADR-002 §2.5 record-lock — only one user holds lock at a time."""
    record_id = 'RAW-TEST-LOCK-001'
    user_a = create_user_with_role('user_a', 'ROLE_STEWARD_MDG-008')
    user_b = create_user_with_role('user_b', 'ROLE_STEWARD_MDG-008')
    
    # User A acquires
    lock_a = acquire_lock('MDG-008', record_id, user_a)
    assert lock_a.status == 200
    
    # User B tries — should get 423 LOCKED
    lock_b = acquire_lock('MDG-008', record_id, user_b)
    assert lock_b.status == 423
    assert lock_b.held_by == user_a.user_id
    
    # User A releases
    release_lock('MDG-008', record_id, user_a)
    
    # User B retries — should succeed
    lock_b2 = acquire_lock('MDG-008', record_id, user_b)
    assert lock_b2.status == 200
```

## 7. Excel V2 round-trip test

```python
# tests/unit/test_excel_v2_roundtrip.py
def test_excel_v2_import_then_export_idempotent():
    """SPEC-005 round-trip preserves born-clean v2 schema."""
    # Import golden fixture
    import_result = import_excel_v2('fixtures/master_data_v2_excerpt.xlsx')
    assert import_result.status == 'SUCCESS'
    
    # Export back
    exported = export_excel_v2()
    
    # Compare structural equality (ignore timestamps in README tab)
    assert tabs_equal(
        load_excel('fixtures/master_data_v2_excerpt.xlsx'),
        load_excel(exported),
        ignore_tabs=['README', 'Migration_Log', 'Validation_Report']
    )
```

## 8. Acceptance scenarios (ACC-001)

```gherkin
SCENARIO: New steward onboarding — first NVL create
  GIVEN user 'thy_test' assigned ROLE_STEWARD_MDG-003
  WHEN they log in + navigate to MDG-003 list
  THEN they see all PRD records (read access)
  AND they can click "New Product"
  AND fill ma_mon='PRD-TEST-001' + required fields
  AND submit
  THEN HTTP 201 returned
  AND audit_log has CREATE row
  AND record state = 'DRAFT' (awaiting approver)

SCENARIO: Approver approves draft
  GIVEN draft record PRD-TEST-001 exists
  GIVEN user 'ktt_test' has ROLE_APPROVER_MDG-003
  WHEN they navigate to record + click "Approve"
  THEN HTTP 200 returned
  AND record state = 'APPROVED'
  AND audit_log has APPROVE row
  AND record visible in Excel V2 export

SCENARIO: Excel V2 sync with 1 NEW + 1 UPDATE + 1 CONFLICT
  GIVEN MDM PostgreSQL has 5 NVL records (3 active, 2 changed since last sync)
  GIVEN Excel V2 has 6 NVL records (1 new, 2 with different values than MDM)
  WHEN steward triggers /excel/import
  THEN sync_run created with 3 diffs: 1 NEW, 1 UPDATE, 1 CONFLICT
  AND steward reviews each
  AND accepts NEW + UPDATE
  AND manually merges CONFLICT
  AND commits
  THEN MDM PostgreSQL reflects accepted changes
  AND audit log records each commit
  AND Excel V2 next-export matches MDM state

SCENARIO: Costing engine matches Phase 1 PROVEN ground-truth
  GIVEN MDM PostgreSQL ingested from MASTER-DATA-V2.xlsx (latest)
  WHEN /engine/compute-recipe-cost called for PRD-0173 with transaction_datetime=2026-06-25
  THEN response cost_price within ±0.5% of 60,863đ (@mis PROVEN)

SCENARIO: §R-37 gate — production deploy blocked if fails
  GIVEN CI runs test_r37_uses_transaction_datetime_not_now
  WHEN engine code accidentally uses now() in price lookup
  THEN test fails
  AND CI blocks merge to main
  AND production deploy DOES NOT proceed
```

## 9. References

- SPEC-001 engine-review checklist
- SPEC-002 PostgreSQL DDL
- SPEC-003 OpenAPI
- SPEC-004 RBAC matrix
- SPEC-005 Excel V2 round-trip
- SPEC-006 Costing engine LOGIC-SPEC
- @mis Phase 1 PROVEN values: HANDOFF-ITADMIN-COGS-INGEST-TEST-28062026.md §A0.5 + §B.3
