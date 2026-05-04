# Step 2 — Prepare Comparable Environments

## Goal
Ensure dev and prod datasets are directly comparable.

## Actions

1. Build dev models:
   dbt build --select <models> --target dev --defer --state prod_artifacts

2. Identify:
   - Incremental models
   - Time filters
   - Late-arriving data risks

3. Align datasets:
   - Apply same date filters
   - Ensure same grain
   - Ensure no partial loads

## Output

- Confirmation of environment setup
- Any assumptions or filters applied

## Guardrails

- NEVER compare mismatched time windows
- NEVER proceed if environments are inconsistent

## Interaction

Ask:

"Environment is ready. Proceed to row-level checks?"