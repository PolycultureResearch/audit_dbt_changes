# audit_dbt_changes

A Codex skill for auditing dbt model changes against production data with
`dbt-labs/audit_helper`.

The skill is designed for refactors, model rewrites, layer migrations, and PRs
that touch modeled SQL. It starts with cheap checks, escalates only when needed,
and produces a structured audit report with the SQL and sampled rows used as
evidence.

## What It Does

- Scopes audits around mart-layer models by default (`fct_*`, `dim_*`).
- Rebuilds changed models in a dev schema, preferably with dbt state deferral.
- Compares dev output against prod using `audit_helper` macros.
- Stops at gates when row counts, schemas, or row-level classifications diverge.
- Requires evidence before calling a root cause confirmed.
- Produces a Markdown audit report from the included template.

## Repository Layout

```text
dbt_audit/
  SKILL.md                         # Main Codex skill instructions
  assets/
    audit_report_template.md       # Final audit report template
  references/
    setup.md                       # audit_helper setup and dbt target notes
    macros.md                      # audit_helper macro reference
```

## Installation

Install this skill by copying or symlinking the `dbt_audit` directory into your
Codex skills directory.

```bash
mkdir -p ~/.codex/skills
ln -s "$(pwd)/dbt_audit" ~/.codex/skills/dbt_audit
```

Restart Codex after installing so the skill is discovered.

## Usage

Ask Codex to audit a dbt PR or validate dbt model changes. Example prompts:

```text
Use the dbt-audit skill to audit this PR.
```

```text
Check whether my dbt model refactor changed production outputs.
```

```text
Audit the downstream marts for this branch.
```

The skill will first propose scope and ask for confirmation before running
builds or warehouse comparisons.

## dbt Project Prerequisites

The target dbt project should have:

- `dbt-labs/audit_helper` installed in `packages.yml`.
- A writable dev target schema.
- Access to production relations for comparison.
- Production artifacts for `--defer --state`, or enough time for a full dev
  build.
- A known primary key for each model being audited.

See [dbt_audit/references/setup.md](dbt_audit/references/setup.md) for setup
details and example commands.

## Audit Flow

The skill follows this order:

1. Scope changed and downstream mart models.
2. Build dev outputs.
3. Run quick equality checks when supported.
4. Compare schemas.
5. Compare row counts.
6. Classify rows as identical, added, removed, or modified.
7. Drill into differing columns.
8. Test root-cause hypotheses with focused SQL.
9. Write the final audit report.

The workflow intentionally pauses when discrepancies appear, so a user can
confirm whether a difference is expected before deeper or more expensive checks.

## Notes

This repository contains the skill package only. It does not contain a dbt
project, warehouse credentials, or generated audit results.
