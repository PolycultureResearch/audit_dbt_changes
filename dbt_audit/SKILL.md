---
name: dbt-audit
description: >
  Audit changes to a dbt project — refactors, layer migrations, model rewrites,
  or any PR that touches modeled SQL — by comparing dev output against prod with
  the dbt-labs/audit_helper package. Use whenever the user mentions auditing,
  validating, diffing, or "checking" dbt models; opens a dbt PR; refactors a
  project; or asks "did my changes break anything?" The skill focuses on the
  mart/consumption layer by default (fct_*, dim_*) because that is what
  downstream consumers depend on. It walks the user through a fixed sequence
  of cheap-to-expensive checks, pauses at every gate that finds a discrepancy,
  refuses to diagnose a root cause without evidence, and produces a structured
  audit report at the end.
---

# dbt Audit Skill

A partner-style workflow for validating dbt model changes. Modeled on the
audit pattern from Hex's analytics engineering team and the dbt-labs
`audit_helper` package. The agent proposes, runs, reports, and pauses — it
never accepts a discrepancy or diagnoses a root cause without explicit user
sign-off.

## Core principles

1. **Mart-first.** Audit the consumption layer (`fct_*`, `dim_*`) by default.
   Intermediate models matter only if a divergence in a mart is traced back
   to one. Staging models are usually out of scope unless the user says so.
2. **Cheap before expensive.** Run the cheapest check that can rule a model
   out (quick-equal → row count → schema → row classification → column
   drilldown). Stop early when you can.
3. **No root cause without evidence.** Hypothesize freely, but never present
   a cause as established without (a) a specific SQL test, (b) a sample of
   the actual rows, and (c) the user's confirmation.
4. **Show your work.** Print every SQL string you ran. Print actual sampled
   rows, not just counts. Save them into the final report.

## Default scope

Unless the user says otherwise, audit:

- Every `fct_*` and `dim_*` model touched by the PR or downstream of a
  touched model.
- Every other mart-layer model (configured `materialized: table` or `incremental`
  in the marts directory).

Skip by default:

- `stg_*` models — assume their tests caught regressions.
- `int_*` models — only audit if a mart audit fails and points back to one.

Always pause and ask the user to confirm scope before running anything.

## Prerequisites

Before starting, confirm:

- `dbt-labs/audit_helper` is in `packages.yml` and `dbt deps` has run.
  See `references/setup.md` if not.
- A dev target writes to a schema the user can rebuild (e.g. `dbt_<user>`).
- Production artifacts are available locally (for `--defer --state`), or the
  user is willing to do a full dev build.
- The user knows the primary key for each model being audited.

If any prerequisite is missing, resolve it with the user before proceeding.

## Workflow

The steps are fixed in order. Steps 1–3 are mandatory. Steps 4–7 are
mandatory only when an earlier step shows a discrepancy.

### Step 1 — Scope

Read the PR diff, `git status`, or `dbt ls --select state:modified+ --state <prod>`
to discover what changed. Apply the default-scope rules above to produce a
candidate list. Briefly say *why* each model is in or out.

> Pause: "Here's the audit scope I'm proposing. Add, drop, or confirm?"

### Step 2 — Build the dev schema

Rebuild only what changed, deferring upstream dependencies to prod:

```bash
dbt build \
  --select <model>+ \
  --target dev \
  --defer --state <path/to/prod/artifacts>
```

(For dbt-fusion, the same flags work; if `--state` is unavailable, do a full
`dbt build --select <model>+ --target dev`.)

**Detect time-scoping early.** Grep the model and its upstream chain for
`limit_build`, `is_incremental()`, or `var('start_date')`. If any are present,
note the date range the dev build covers. *Every* later comparison against
prod must apply the same `WHERE` clause, or the comparison is meaningless.

If the build fails, stop and surface the errors. Do not audit a stale or
partial dev schema.

### Step 3 — Quick equality check (Snowflake / BigQuery only)

For each model, try the fast binary check first:

```sql
{{ audit_helper.quick_are_relations_identical(
    a_relation = adapter.get_relation(database='<prod_db>', schema='<prod_schema>', identifier='<model>'),
    b_relation = ref('<model>')
) }}
```

If `true`, mark the model **PASS** and skip to the next model. If `false`,
or if the adapter is Redshift/Postgres/other, continue to step 4. Note the
result either way — it's audit evidence.

### Step 4 — Schema diff

```sql
{{ audit_helper.compare_relation_columns(
    a_relation = prod_relation,
    b_relation = dev_relation
) }}
```

Report: columns added, removed, or with a changed data type. New columns
that are part of the PR are expected; missing columns or changed types in
existing columns are not.

> Pause if any column disappeared or any data type changed unexpectedly.

### Step 5 — Row count

```sql
{{ audit_helper.compare_row_counts(
    a_relation = prod_relation,
    b_relation = dev_relation
) }}
```

Print both totals (not just the delta). Apply the same date filter as the
dev build if `limit_build` was detected.

| Delta vs prod | Action |
| --- | --- |
| 0 rows | Continue to step 6. |
| < 1% | Flag and ask the user whether the change is expected before continuing. |
| ≥ 1% | **Stop.** Do not advance without explicit user approval. |

### Step 6 — Row classification

```sql
{{ audit_helper.compare_and_classify_relation_rows(
    a_relation = prod_relation,
    b_relation = dev_relation,
    primary_key_columns = ["<pk>"],
    columns = None,
    sample_limit = 20
) }}
```

If a column was renamed or filters need to be applied, switch to
`compare_and_classify_query_results` with explicit SELECTs. Apply default
noise reduction in the SELECT:

- `DATE_TRUNC('hour', ts_col) AS ts_col`
- `ROUND(numeric_col::float, 2) AS numeric_col`
- `COALESCE(string_col, '') AS string_col` for nullable string joins

Log every transformation applied — these decisions affect what counts as
identical.

Report the identical / added / removed / modified counts and percentages.
Print up to 20 sample rows from the `modified` set.

| Identical % | Action |
| --- | --- |
| 100% | Mark **PASS** and continue to next model. |
| 99–<100% | Flag, print the sample, ask whether the diff is expected. |
| < 99% | **Stop.** Do not advance without explicit user approval. |

Do not propose a root cause yet.

### Step 7 — Column drilldown

Only when step 6 shows modified rows. Use one summary call rather than
iterating per column:

```sql
{{ audit_helper.compare_all_columns(
    a_relation = prod_relation,
    b_relation = dev_relation,
    primary_key = "<pk>",
    exclude_columns = ["loaded_at", "updated_at"],
    summarize = true
) }}
```

For columns flagged with conflicts, follow up with `compare_column_values`
to see the distribution of mismatch types (perfect match / both null /
missing on one side / conflicting).

Then sample real rows side-by-side:

```sql
SELECT a.<pk>, a.<col> AS prod_value, b.<col> AS dev_value
FROM <prod_relation> a
JOIN {{ ref('<model>') }} b USING (<pk>)
WHERE a.<col> IS DISTINCT FROM b.<col>
LIMIT 20
```

### Step 8 — Root cause

Form one or more hypotheses (logic change, join cardinality, filter change,
late-arriving data, upstream model). For each one, run a focused SQL test
and print the result. State a hypothesis as confirmed only when:

1. A specific SQL query proves the mechanism.
2. Sample rows are shown to the user.
3. The user explicitly accepts the explanation.

"Data drift" is not an explanation; it's a hypothesis that must be tested
the same way (find an `updated_at` newer than the prod build, show the
matching rows).

If a mart audit traces back to an upstream `int_*` model, audit that model
with the same workflow before concluding.

### Step 9 — Report

Produce a Markdown report using `assets/audit_report_template.md`. Each
audited model gets a verdict of:

- **PASS** — All checks passed.
- **EXPECTED CHANGES** — Discrepancies detected, root cause confirmed, user
  approved.
- **UNRESOLVED** — Discrepancies remain; PR is not safe to merge.

Each section includes the SQL run (collapsed in `<details>` tags), the
sample rows, and a one-paragraph plain-English summary. Save the report
alongside the PR.

## Hard rules

- Never accept a discrepancy as "expected" without user confirmation.
- Never compare a `limit_build` dev model against unrestricted prod.
- Never iterate per-column with `compare_column_values` when
  `compare_all_columns(summarize=true)` answers the same question in one query.
- Never write the report until every gate is resolved.
- Never skip a pause gate, even when results look clean.

## Reference files

- `references/setup.md` — Installing audit_helper, configuring profiles,
  state-based deferral, and (for dbt-fusion users) flag differences.
- `references/macros.md` — Full signatures, output schemas, and noise-reduction
  patterns for every audit_helper macro.
- `assets/audit_report_template.md` — The PR audit report template, with a
  worked example.

Read a reference file when you need detail not covered above.
