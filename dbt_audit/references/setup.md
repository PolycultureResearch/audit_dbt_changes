# Setup reference

Everything you need to bootstrap `audit_helper` in a project.

## Install the package

Add to `packages.yml`:

```yaml
packages:
  - package: dbt-labs/audit_helper
    version: [">=0.13.0", "<1.0.0"]
```

Check the latest version at <https://hub.getdbt.com/dbt-labs/audit_helper/latest/>.

Install:

```bash
dbt deps
```

Confirm the macros loaded:

```bash
dbt ls --select audit_helper
```

## Targets and profiles

The audit assumes:

- `prod`: read-only access to the production schema.
- `dev`: a writable personal schema, e.g. `dbt_<user>` or `pr_<branch>`.

A typical `profiles.yml`:

```yaml
my_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      schema: "dbt_{{ env_var('DBT_USER', 'dev') }}"
      # ... connection settings
    prod:
      type: snowflake
      schema: analytics
      # ... connection settings
```

## State-based deferral

Deferral lets you rebuild only changed models in dev while resolving
upstream `ref()`s to prod. This is the fastest, most accurate way to set
up a dev schema for auditing.

```bash
# If you have prod artifacts locally:
dbt build \
  --select state:modified+ \
  --target dev \
  --defer --state ./prod_artifacts

# In dbt Cloud, download prod artifacts first:
dbt-cloud run download-artifacts --run-id <prod_run_id> --path ./prod_artifacts
```

If artifacts aren't available, fall back to a full dev build:

```bash
dbt build --select <model>+ --target dev
```

## dbt-fusion notes

`dbt-fusion` (the Rust-based runtime, in preview as of 2026) accepts the
same `--defer --state` flags as classic dbt. A few things to know:

- `dbt deps` writes packages to `dbt_packages/` the same way.
- The `target/` directory layout matches classic dbt for `manifest.json`
  and `compiled/` SQL, so audit_helper's compile-then-paste workflow works.
- `dbt run-operation` is supported.
- If a flag isn't yet implemented in your fusion preview, fall back to a
  full dev build for now.

## Detecting limit_build / incremental scoping

Before auditing, scan the model and its upstream chain for date-limiting
patterns:

```bash
grep -r "limit_build\|is_incremental\|var('start_date')" models/ --include="*.sql" -l
```

If any are present, find the date range the dev build covers (often the
last N days). Apply the same `WHERE` clause to every prod query in the
audit:

```sql
-- e.g. dev built for the last 7 days
WHERE created_at >= DATEADD('day', -7, CURRENT_DATE)
```

## Running audit_helper macros

`audit_helper` macros emit SQL; they don't materialize tables. Three ways
to run them:

### Analysis files (recommended for ad-hoc audits)

Create `analyses/audit_<model>.sql`:

```sql
{% set prod_relation = adapter.get_relation(
    database = 'prod_db',
    schema   = 'analytics',
    identifier = 'fct_orders'
) %}
{% set dev_relation = ref('fct_orders') %}

{{ audit_helper.compare_and_classify_relation_rows(
    a_relation = prod_relation,
    b_relation = dev_relation,
    primary_key_columns = ['order_id']
) }}
```

Compile and run:

```bash
dbt compile --select analyses/audit_fct_orders
# Then run the SQL from target/compiled/<project>/analyses/audit_fct_orders.sql
# in your warehouse SQL client.
```

### Singular tests (recommended for persistent audits)

Drop a file in `tests/audit_<model>.sql`:

```sql
{{ audit_helper.compare_all_columns(
    a_relation = ref('fct_orders'),
    b_relation = api.Relation.create(database='prod_db', schema='analytics', identifier='fct_orders'),
    primary_key = 'order_id',
    exclude_columns = ['updated_at']
) }}
where conflicting_values
```

Run:

```bash
dbt test --select fct_orders
```

### dbt Cloud / IDE SQL runner

Paste the macro call into the SQL preview pane and run it directly.

## Adapter compatibility

| Macro | Snowflake | BigQuery | Redshift | Postgres |
| --- | --- | --- | --- | --- |
| `quick_are_relations_identical` | yes | yes | no | no |
| All other macros | yes | yes | yes | yes |
| Ordinal position in schema diff | yes | yes | yes | yes |

For adapters without `quick_are_relations_identical`, skip step 3 of the
workflow and start with the schema diff.
