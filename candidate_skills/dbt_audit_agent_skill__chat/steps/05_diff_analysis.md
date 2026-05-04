# Step 5 — Row-Level Diff Classification

## Goal
Understand *how* rows differ.

## SQL

{{ audit_helper.compare_and_classify_relation_rows(
    a_relation=prod_relation,
    b_relation=dev_relation,
    primary_key=primary_key
) }}

## Output

- Rows only in A
- Rows only in B
- Rows with mismatched values

## Additional Actions

- Sample 20–100 discrepant rows
- Show FULL row contents for inspection

## Rules

- MUST show real data examples
- DO NOT summarize without examples

## Interaction

Ask:

"Diff classification complete. Proceed to root cause investigation?"
