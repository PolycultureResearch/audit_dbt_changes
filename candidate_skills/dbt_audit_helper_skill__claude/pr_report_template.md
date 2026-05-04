# dbt Audit Report

**PR:** <!-- link or title -->
**Branch:** <!-- branch name -->
**Author:** <!-- your name -->
**Date:** <!-- YYYY-MM-DD -->
**Models audited:** <!-- comma-separated list -->

---

## Summary

| Model | Row Count | Schema | Row Match Rate | Verdict |
|-------|-----------|--------|----------------|---------|
| `fct_orders` | ✅ Equal | ✅ No changes | 100% | ✅ PASSED |
| `dim_customers` | ✅ Equal | ⚠️ 1 new col | 99.4% | ✅ EXPECTED CHANGES |
| `int_order_items` | ✅ Equal | ✅ No changes | 100% | ✅ PASSED |

---

## Model Audits

<!-- Repeat this block for each model -->

### `fct_orders`

**Verdict: ✅ PASSED**

**Schema comparison:**
No column additions, removals, or type changes detected.

**Row count:**
- Prod: 34,231 rows
- Dev: 34,231 rows
- Delta: 0 (0%)

**Row-level comparison:**
- Identical: 34,231 (100%)
- Added: 0
- Removed: 0
- Modified: 0

No discrepancies detected.

<details>
<summary>SQL used</summary>

```sql
-- Row count check
{{ audit_helper.compare_row_counts(
    a_relation = adapter.get_relation(database="prod_db", schema="analytics", identifier="fct_orders"),
    b_relation = ref('fct_orders')
) }}

-- Row classification
{{ audit_helper.compare_and_classify_relation_rows(
    a_relation = adapter.get_relation(database="prod_db", schema="analytics", identifier="fct_orders"),
    b_relation = ref('fct_orders'),
    primary_key_columns = ["order_id"],
    columns = None
) }}
```

</details>

---

### `dim_customers`

**Verdict: ✅ EXPECTED CHANGES**

**Schema comparison:**
- New column added in dev: `loyalty_tier` (varchar) — expected, part of this PR

**Row count:**
- Prod: 12,004 rows
- Dev: 12,004 rows
- Delta: 0 (0%)

**Row-level comparison:**
- Identical: 11,924 (99.3%)
- Modified: 80 (0.7%)

**Root cause (approved):** 80 records show changes to `customer_segment` — these correspond to
customers who were re-segmented as part of the new segmentation logic introduced in this PR.
Sample reviewed and approved by @<!-- your name -->.

**Sample of modified records:**

| customer_id | customer_segment (prod) | customer_segment (dev) | approved |
|---|---|---|---|
| 1042 | "Bronze" | "Silver" | ✅ |
| 2891 | "Silver" | "Gold" | ✅ |
| ... | ... | ... | ... |

<details>
<summary>SQL used</summary>

```sql
-- Excludes new 'loyalty_tier' column to prevent intersect errors
{{ audit_helper.compare_and_classify_relation_rows(
    a_relation = adapter.get_relation(database="prod_db", schema="analytics", identifier="dim_customers"),
    b_relation = ref('dim_customers'),
    primary_key_columns = ["customer_id"],
    columns = dbt_utils.get_filtered_columns_in_relation(ref('dim_customers'), except=["loyalty_tier"])
) }}

-- Column-level drill-down on customer_segment
{{ audit_helper.compare_column_values(
    a_query = "SELECT * FROM prod_db.analytics.dim_customers",
    b_query = "SELECT * FROM {{ ref('dim_customers') }}",
    primary_key = "customer_id",
    column_to_compare = "customer_segment"
) }}
```

</details>

---

<!-- Add additional model blocks above this line -->

---

## Unresolved Items

<!-- List any checks that are still open, or write "None." -->

None.

---

## Notes

<!-- Any additional context for the reviewer -->

- Timestamps were truncated to the nearest hour for all comparisons to reduce noise from
  pipeline timing differences.
- `dim_customers` audit excluded `loyalty_tier` (new column) from the row comparison to avoid
  breaking the intersect check; schema diff above documents its addition.
