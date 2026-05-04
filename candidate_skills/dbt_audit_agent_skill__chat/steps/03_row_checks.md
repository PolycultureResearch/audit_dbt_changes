# Step 3 — Row-Level Checks

## Goal
Detect high-level differences quickly.

## SQL

### Row count comparison
{{ audit_helper.compare_row_counts(
    a_relation=prod_relation,
    b_relation=dev_relation
) }}

### Equality check
{{ audit_helper.quick_are_relations_identical(
    a_relation=prod_relation,
    b_relation=dev_relation
) }}

## Output

- Row counts (A vs B)
- % difference
- Identical? (true/false)

## Rules

- DO NOT interpret differences yet
- ONLY report metrics

## Interaction

Ask:

"Row-level check complete. Proceed to column-level comparison?"