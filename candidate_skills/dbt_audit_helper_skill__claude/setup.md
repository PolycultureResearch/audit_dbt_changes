# Setup Reference: dbt-audit-helper

## Installation

### 1. Add to packages.yml

```yaml
packages:
  - package: dbt-labs/audit_helper
    version: [">=0.13.0", "<1.0.0"]  # check hub.getdbt.com for latest
```

Check the latest version at: https://hub.getdbt.com/dbt-labs/audit_helper/latest/

### 2. Install the package

```bash
dbt deps
```

### 3. Confirm installation

```bash
dbt ls --select audit_helper
```

You should see the audit_helper macros listed.

---

## Adapter Compatibility

The package supports all major adapters. Some features are adapter-specific:

| Feature | Snowflake | BigQuery | Redshift | Postgres | Others |
|---------|-----------|----------|----------|----------|--------|
| `quick_are_relations_identical` | ✅ | ✅ | ❌ | ❌ | ❌ |
| All other macros | ✅ | ✅ | ✅ | ✅ | ✅ |
| Ordinal position in schema compare | ✅ | ✅ | ✅ | ✅ | Inferred |

For adapters without `quick_are_relations_identical` support, skip Step 7 and go straight to the
full workflow.

---

## Target / Profile Setup

The audit assumes:
- **`prod`** target: points to your production schema (read-only is fine)
- **`dev`** target: your personal dev schema (writable)

A typical `profiles.yml` setup:

```yaml
my_project:
  target: dev
  outputs:
    dev:
      type: snowflake
      schema: "dbt_{{ env_var('DBT_USER', 'dev') }}"
      # ... other connection settings
    prod:
      type: snowflake
      schema: analytics
      # ... other connection settings
```

---

## Using State-Based Deferral

Deferral lets you rebuild only the changed models while pulling upstream dependencies from prod.
This is essential for fast, accurate dev schema builds.

```bash
# Download prod artifacts first (if using dbt Cloud)
dbt-cloud run download-artifacts --run-id <latest_prod_run_id> --path ./prod_artifacts

# Or if you have them locally:
dbt build \
  --select state:modified+ \
  --target dev \
  --defer \
  --state ./prod_artifacts
```

If you don't have prod artifacts, you can do a full dev build:
```bash
dbt build --select <model>+ --target dev
```

---

## Running Audit Macros

Audit helper macros are typically run as **operations** (not models), meaning they execute SQL
and return results without creating a table.

### Method 1: dbt run-operation (recommended for ad-hoc auditing)

```bash
dbt run-operation <macro_name> --args '{...}'
```

However, audit_helper macros are typically invoked via analysis files or inline in the IDE.

### Method 2: Analysis files

Create a file in `analyses/audit_<model_name>.sql`:

```sql
{% set prod_relation = adapter.get_relation(
    database = target.database,
    schema   = "analytics",
    identifier = "fct_orders"
) %}

{% set dev_relation = ref('fct_orders') %}

{{ audit_helper.compare_and_classify_relation_rows(
    a_relation = prod_relation,
    b_relation = dev_relation,
    primary_key_columns = ["order_id"],
    columns = None
) }}
```

Then compile and run:
```bash
dbt compile --select analyses/audit_fct_orders.sql
# Copy compiled SQL from target/compiled/ and run in your SQL client
```

### Method 3: dbt Cloud IDE

Paste the macro call directly into the IDE's SQL runner and click Preview.

---

## Checking for limit_build Patterns

Before auditing incremental models, check for date-limiting macros:

```bash
# Search for common limit patterns in your project
grep -r "limit_build\|is_incremental\|var('start_date')" models/ --include="*.sql" -l
```

If any model in the audit's upstream chain uses such a macro, note the date range it produces
and apply a matching `WHERE` filter to your prod query.

Example:
```sql
-- If dev builds last 7 days, scope prod to match:
{% set prod_date_filter %}
    WHERE created_at >= DATEADD('day', -7, CURRENT_DATE)
{% endset %}
```
