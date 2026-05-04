# dbt Audit Report

**PR:** <!-- link or title -->
**Branch:** <!-- branch name -->
**Author:** <!-- name -->
**Date:** <!-- YYYY-MM-DD -->
**Models audited:** <!-- comma-separated list -->
**Date scope applied:** <!-- e.g. created_at >= 2026-04-27, or "none" -->

---

## Summary

| Model | Quick equal | Schema | Row count | Row match | Verdict |
| --- | --- | --- | --- | --- | --- |
| `fct_orders` | true | unchanged | equal | 100% | PASS |
| `dim_customers` | false | +1 col | equal | 99.3% | EXPECTED CHANGES |
| `int_order_items` | false | unchanged | equal | 100% | PASS |

Verdicts:

- **PASS** — every check passed.
- **EXPECTED CHANGES** — discrepancies found, root cause confirmed by author.
- **UNRESOLVED** — discrepancies remain; do not merge.

---

## Model audits

<!-- One block per model. -->

### `fct_orders`

**Verdict:** PASS

`quick_are_relations_identical` returned `true`. No further checks required.

<details>
<summary>SQL</summary>

```sql
{{ audit_helper.quick_are_relations_identical(
    a_relation = adapter.get_relation(database='prod_db', schema='analytics', identifier='fct_orders'),
    b_relation = ref('fct_orders')
) }}
```

</details>

---

### `dim_customers`

**Verdict:** EXPECTED CHANGES

**Schema diff.** New column `loyalty_tier` (varchar) added in dev. No
columns removed; no type changes. Expected — column is part of this PR.

**Row count.** Prod 12,004 rows; dev 12,004 rows; delta 0.

**Row classification.**

- Identical: 11,924 (99.33%)
- Added: 0
- Removed: 0
- Modified: 80 (0.67%)

`loyalty_tier` was excluded from the row classification (added in dev,
broke the intersect).

**Column drilldown.** `customer_segment` is the only column with conflicts
(80 rows). All other columns: perfect match.

**Root cause (confirmed).** The 80 modified rows correspond to customers
re-segmented by the new tiering logic in `int_customer_segments`. Sample
rows reviewed and accepted by the author.

| customer_id | customer_segment (prod) | customer_segment (dev) |
| --- | --- | --- |
| 1042 | Bronze | Silver |
| 2891 | Silver | Gold |
| ... | ... | ... |

<details>
<summary>SQL</summary>

```sql
-- Schema diff
{{ audit_helper.compare_relation_columns(
    a_relation = adapter.get_relation(database='prod_db', schema='analytics', identifier='dim_customers'),
    b_relation = ref('dim_customers')
) }}

-- Row count
{{ audit_helper.compare_row_counts(
    a_relation = adapter.get_relation(database='prod_db', schema='analytics', identifier='dim_customers'),
    b_relation = ref('dim_customers')
) }}

-- Row classification (excludes new column)
{{ audit_helper.compare_and_classify_relation_rows(
    a_relation = adapter.get_relation(database='prod_db', schema='analytics', identifier='dim_customers'),
    b_relation = ref('dim_customers'),
    primary_key_columns = ['customer_id'],
    columns = dbt_utils.get_filtered_columns_in_relation(ref('dim_customers'), except=['loyalty_tier'])
) }}

-- Column drilldown
{{ audit_helper.compare_all_columns(
    a_relation = adapter.get_relation(database='prod_db', schema='analytics', identifier='dim_customers'),
    b_relation = ref('dim_customers'),
    primary_key = 'customer_id',
    exclude_columns = ['loyalty_tier'],
    summarize = true
) }}
```

</details>

---

<!-- Add additional model blocks above this line. -->

## Unresolved items

<!-- List any open checks. Write "None." if the audit is fully resolved. -->

None.

---

## Notes

- Timestamps were truncated to the nearest hour for all comparisons to
  reduce noise from pipeline timing differences.
- Floats were rounded to 2 decimal places.
- `dim_customers` row classification excluded `loyalty_tier` (new column);
  the schema diff above documents the addition.
- Date scope: <!-- e.g. "created_at >= 2026-04-27 (matched to dev limit_build window)" or "none — full history compared" -->
