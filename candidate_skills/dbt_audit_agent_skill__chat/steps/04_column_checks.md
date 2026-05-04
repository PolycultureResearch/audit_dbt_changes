# Step 4 — Column-Level Checks

## Goal
Identify schema and value differences.

## SQL

{{ audit_helper.compare_all_columns(
    a_relation=prod_relation,
    b_relation=dev_relation
) }}

## Output

- Columns only in A
- Columns only in B
- Columns with mismatched values
- Distribution differences (if available)

## Rules

- Apply normalization (rounding, truncation)
- Highlight suspicious columns

## Interaction

Ask:

"Column-level differences identified. Proceed to row-level diff classification?"