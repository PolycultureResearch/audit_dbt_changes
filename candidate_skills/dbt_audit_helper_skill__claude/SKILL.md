---
name: dbt-audit-helper
description: >
  An agentic skill for auditing changes to a dbt project—especially large refactors, layer migrations,
  or model rewrites—using the dbt-labs/dbt-audit-helper package. Use this skill whenever the user:
  - mentions auditing, validating, or checking dbt model changes
  - is doing a dbt refactor, migration, or PR review and needs to verify data integrity
  - asks to compare prod vs dev schemas in dbt
  - says things like "make sure my changes didn't break anything", "validate my PR", "check my dbt models"
  - needs to produce a PR summary or audit report for a dbt change
  Covers the full audit lifecycle: schema build → row count checks → row/column comparison → root cause
  analysis → PR report generation. Always partner with the user; never make unilateral decisions about
  scope or accept discrepancies without explicit approval.
---

# dbt Audit Helper Skill

A structured, end-to-end agent skill for validating dbt project changes. Modeled on the workflow
described by Hex's data team and the dbt-labs `audit_helper` package. The agent is a **partner**, not
an autonomous actor: it proposes, pauses, reports, and asks—then proceeds only with approval.

---

## Core Principles

These govern every step. Re-read them if you're unsure how to proceed.

1. **Be a good partner.** Proactively suggest which models to audit and why. Always explain your
   reasoning. Present options, not mandates.
2. **Work smarter, not harder.** Build fresh dev schemas with deferral, apply noise reduction
   (timestamp truncation, float rounding), handle large/incremental models by date-scoping.
3. **Don't make assumptions.** Require hard evidence before diagnosing a root cause. "Data drift"
   is not an acceptable explanation unless you can prove it. Pause for approval at every gate.
4. **Show your work.** Print the SQL you ran. Surface sampled rows. Generate a structured PR report
   at the end.

---

## Prerequisites

Before starting, confirm:

- [ ] `dbt-audit-helper` is in the project's `packages.yml` (see `references/setup.md`)
- [ ] User has a dev target configured (e.g., `dev` or `pr_<branch>` schema)
- [ ] User can run `dbt` from the command line in this session
- [ ] Primary key column(s) are known for each model to be audited

If any are missing, help the user resolve them before continuing.

---

## Step-by-Step Audit Workflow

### Step 1 — Scope the Audit

**Goal:** Decide which models to audit.

1. Read `dbt ls` output or the user's PR diff to understand what changed.
2. **Proactively suggest** models to audit, prioritizing in this order:
   - `fct_` and `dim_` (core/mart layer) — highest priority; downstream consumers depend on these
   - `int_` (intermediate) — audit if a fct/dim pulls from it
   - `stg_` (staging) — audit only if the change originated here and wasn't caught above
3. Briefly explain why each suggested model should (or doesn't need to) be audited.
4. **Pause. Ask the user to confirm, add, or remove models before proceeding.**

> 💬 "Here are the models I'd suggest auditing based on the diff. Does this scope look right, or
> would you like to add/remove anything?"

---

### Step 2 — Build the Dev Schema

**Goal:** Ensure the dev schema is fresh before comparing against prod.

Run dbt with state-based deferral to rebuild only what's needed:

```bash
# Standard rebuild with deferral (swap target/defer-state as appropriate)
dbt build \
  --select <model_name>+ \
  --target dev \
  --defer \
  --state path/to/prod/artifacts
```

**Incremental / large models:** Check whether the model or any upstream dependency uses a
`limit_build` macro or equivalent date-truncation pattern. If so:
- Note the date range of the dev build
- In all subsequent audit queries, add a `WHERE` clause to prod that matches the same range
- Log: "Detected `limit_build` in `<model>`. Scoping prod query to `<date_range>` to ensure
  apples-to-apples comparison."

**Gate:** Confirm build succeeded (no errors). If there are build errors, stop and surface them
to the user. Do not proceed with a stale or partial dev schema.

---

### Step 3 — Schema Comparison

**Goal:** Verify the column structure hasn't changed unexpectedly.

Run `compare_relation_columns` for each model:

```sql
-- In a dbt analysis file or run-operation
{% set prod_relation = adapter.get_relation(
    database = "<prod_database>",
    schema   = "<prod_schema>",
    identifier = "<model_name>"
) %}
{% set dev_relation = ref('<model_name>') %}

{{ audit_helper.compare_relation_columns(
    a_relation = prod_relation,
    b_relation = dev_relation
) }}
```

**Report to user:**
- Any columns added, removed, or changed in data type
- Any ordinal position changes (warn, but these are usually acceptable)

**Gate:** If unexpected columns are missing or types differ in a breaking way, **stop and ask
the user** whether to continue or fix first.

---

### Step 4 — Row Count Check

**Goal:** Quick sanity check before doing expensive row-level comparisons.

```sql
{% set prod_relation = adapter.get_relation(
    database="<prod_db>", schema="<prod_schema>", identifier="<model>"
) %}
{% set dev_relation = ref('<model>') %}

{{ audit_helper.compare_row_counts(
    a_relation = prod_relation,
    b_relation = dev_relation
) }}
```

**Rules:**
- A difference of 0 rows → green light, move on
- A difference of > 0 but < 1% of total rows → flag, ask if expected
- A difference of ≥ 1% → **mandatory stop**; ask the user to explain before proceeding
- If using `limit_build`, verify both queries are scoped to the same date range first

**Always print the actual counts** (not just the delta).

---

### Step 5 — Row-Level Comparison

**Goal:** Identify added, removed, identical, and modified records.

Use `compare_and_classify_relation_rows` (preferred) or `compare_and_classify_query_results`:

```sql
{% set prod_relation = adapter.get_relation(
    database="<prod_db>", schema="<prod_schema>", identifier="<model>"
) %}
{% set dev_relation = ref('<model>') %}

{{ audit_helper.compare_and_classify_relation_rows(
    a_relation = prod_relation,
    b_relation = dev_relation,
    primary_key_columns = ["<pk_column>"],
    columns = None  -- auto-detects intersecting columns
) }}
```

**Noise reduction (apply by default, log when applied):**
- Truncate timestamps to the nearest hour: `DATE_TRUNC('hour', <ts_col>)`
- Round numeric columns with decimal scale to 2 decimal places
- If a column was renamed in the refactor, use `compare_and_classify_query_results` with explicit
  SELECT aliases to map old → new names

**Report:**
- Print the status summary: identical / added / removed / modified counts and percentages
- Print a sample (up to 20 rows) of **modified** records

**Gate — match rate thresholds:**

| Identical % | Action |
|-------------|--------|
| 100%        | ✅ Pass — proceed |
| 99–100%     | ⚠️ Flag — print sample, ask if expected |
| < 99%       | 🛑 Stop — do not proceed without explicit approval |

**Never diagnose a root cause here.** Just report facts and ask.

> 💬 "I'm seeing X% of rows as modified. Here's a sample. Do these differences look expected to
> you, or should I dig deeper?"

---

### Step 6 — Column-Level Drill-Down

**Goal:** Identify *which* columns are driving discrepancies. Use only when Step 5 reveals
unexpected changes.

```sql
{{ audit_helper.compare_which_relation_columns_differ(
    a_relation = prod_relation,
    b_relation = dev_relation,
    primary_key_columns = ["<pk_column>"],
    columns = None
) }}
```

Then for each column flagged as differing, run `compare_column_values`:

```sql
{{ audit_helper.compare_column_values(
    a_query = "select * from <prod_relation>",
    b_query = "select * from {{ ref('<model>') }}",
    primary_key = "<pk_column>",
    column_to_compare = "<column_name>"
) }}
```

**For each column, report:**
- Match rate breakdown (perfect match / null mismatches / conflicting values)
- Sample of conflicting rows (print the actual values from both sides)

**Root cause diagnosis rules — all three must be satisfied before proposing a cause:**
1. Confirm that a relevant timestamp in the model (e.g., `created_at`, `updated_at`) is recent
   enough to plausibly explain drift
2. Print a sample of changed records with those timestamps visible
3. Pause and ask the user: "Does this explanation hold up for you, or should I keep digging?"

**Never explain away discrepancies as "data drift" without evidence meeting all three criteria.**

---

### Step 7 — Quick Identical Check (Optional Fast Path)

For Snowflake or BigQuery users, if the user wants a fast binary answer before doing deeper
checks, offer this first:

```sql
{{ audit_helper.quick_are_relations_identical(
    a_relation = prod_relation,
    b_relation = dev_relation,
    columns = None
) }}
```

If `true` → audit is complete, skip Steps 4–6.
If `false` → proceed with the full workflow.

---

### Step 8 — Generate PR Report

**Goal:** Produce a structured, uniform audit summary for the PR.

After completing all model audits, generate a Markdown report. See `assets/pr_report_template.md`
for the full template.

Each model section uses one of these outcomes:

- **✅ PASSED** — All checks passed, no discrepancies
- **✅ EXPECTED CHANGES** — Discrepancies detected and approved by user; documented with rationale
- **⚠️ UNRESOLVED** — Discrepancies remain under investigation

Always include:
- The SQL used for each check (collapsed in `<details>` tags for readability)
- The sample rows for any modified records
- A plain-English summary of what was validated and what the conclusion is

---

## Important Don'ts

- ❌ Never accept "data drift" as a root cause without timestamps evidence + sample rows + user approval
- ❌ Never skip the pause gates, even when everything looks clean
- ❌ Never compare a `limit_build` dev model against unrestricted prod without date-scoping prod
- ❌ Never advance to the next step if the current one returned errors or ambiguous results
- ❌ Never write the PR report until all audits are complete and all gates are resolved

---

## Reference Files

- `references/setup.md` — Installing audit_helper, packages.yml setup, dbt deps
- `references/macro_reference.md` — Full macro signatures and output schemas
- `assets/pr_report_template.md` — Structured PR report template

Read a reference file when you need details not covered in this SKILL.md.
