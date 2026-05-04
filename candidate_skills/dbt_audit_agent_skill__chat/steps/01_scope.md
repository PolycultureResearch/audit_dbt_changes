# Step 1 — Scope Models

## Goal
Identify which dbt models should be audited.

## Actions

1. Inspect dbt project graph (manifest.json if available)
2. Identify models impacted by:
   - Modified SQL files
   - Upstream dependencies
3. Prioritize:
   - marts (fct_, dim_)
   - high-impact models

## Output

- List of candidate models
- Reason each model was selected

## Interaction (REQUIRED)

Ask user:

"These are the models I propose to audit. Do you want to:
1. Approve
2. Modify the list
3. Focus on a subset?"

DO NOT proceed without confirmation.