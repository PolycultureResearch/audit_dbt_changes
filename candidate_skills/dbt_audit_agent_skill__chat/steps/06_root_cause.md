# Step 6 — Root Cause Investigation

## Goal
Explain WHY differences exist.

## Method

For each discrepancy:

1. Form hypothesis:
   - Logic change
   - Join cardinality change
   - Filter change
   - Data freshness
   - Upstream change

2. Validate hypothesis with SQL

## Example Patterns

### Join explosion check
SELECT COUNT(*) before_join, COUNT(*) after_join

### Filter change check
Compare WHERE clauses

### Upstream diff
Run audit on upstream model

## Output

For each issue:
- Hypothesis
- SQL used to test
- Result
- Conclusion

## Rules

- NO speculation without SQL evidence
- MULTIPLE hypotheses allowed

## Interaction

Ask:

"Root causes identified. Do you want to:
1. Accept differences
2. Investigate further
3. Reject changes?"