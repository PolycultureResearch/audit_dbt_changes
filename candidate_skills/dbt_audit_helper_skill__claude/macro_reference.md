# Macro Reference: dbt-audit-helper

Full signatures, output schemas, and usage notes for all audit_helper macros.

---

## Table of Contents

1. [compare_and_classify_relation_rows](#compare_and_classify_relation_rows) ← primary row check
2. [compare_and_classify_query_results](#compare_and_classify_query_results) ← when you need custom SQL
3. [quick_are_relations_identical](#quick_are_relations_identical) ← fast binary check
4. [compare_row_counts](#compare_row_counts) ← row count sanity check
5. [compare_which_relation_columns_differ](#compare_which_relation_columns_differ) ← column triage
6. [compare_column_values](#compare_column_values) ← per-column drill-down
7. [compare_all_columns](#compare_all_columns) ← all-column summary
8. [compare_relation_columns](#compare_relation_columns) ← schema comparison
9. [Legacy macros](#legacy-macros)

---

## compare_and_classify_relation_rows

**Best for:** Main row-level audit when both relations have the same column names.

```sql
{% set prod_relation = adapter.get_relation(
    database = "<prod_db>",
    schema   = "<prod_schema>",
    identifier = "<model_name>"
) %}
{% set dev_relation = ref('<model_name>') %}

{{ audit_helper.compare_and_classify_relation_rows(
    a_relation = prod_relation,
    b_relation = dev_relation,
    primary_key_columns = ["<pk_col>"],  -- required; list of column names
    columns = None,                       -- None = auto-detect intersecting cols
    sample_limit = 20                     -- rows per status in output
) }}
```

**Output columns:**
- All model columns
- `dbt_audit_in_a` — bool: row exists in prod
- `dbt_audit_in_b` — bool: row exists in dev
- `dbt_audit_row_status` — `identical` | `added` | `removed` | `modified`
- `dbt_audit_num_rows_in_status` — count of PKs with this status
- `dbt_audit_sample_number` — sample index within status

**Notes:**
- `modified` counts each PK once even though it appears twice (old + new value)
- Pass explicit `columns` list to exclude new/removed columns from breaking the intersect

---

## compare_and_classify_query_results

**Best for:** When you need to rename columns, cast types, apply filters, or handle renamed PKs.

```sql
{% set prod_query %}
    SELECT
        id           AS order_id,   -- handle renamed column
        amount,
        customer_id,
        DATE_TRUNC('hour', created_at) AS created_at  -- noise reduction
    FROM prod_db.analytics.fct_orders
    WHERE created_at >= '2024-01-01'  -- date scope for incremental models
{% endset %}

{% set dev_query %}
    SELECT
        order_id,
        amount,
        customer_id,
        DATE_TRUNC('hour', created_at) AS created_at
    FROM {{ ref('fct_orders') }}
    WHERE created_at >= '2024-01-01'
{% endset %}

{{ audit_helper.compare_and_classify_query_results(
    a_query = prod_query,
    b_query = dev_query,
    primary_key_columns = ["order_id"],
    columns = ["order_id", "amount", "customer_id", "created_at"],  -- required
    sample_limit = 20
) }}
```

**Same output schema as `compare_and_classify_relation_rows`.**

---

## quick_are_relations_identical

**Best for:** Fast binary pass/fail before doing full row comparisons. Snowflake + BigQuery only.

```sql
{% set prod_relation = adapter.get_relation(
    database="<prod_db>", schema="<prod_schema>", identifier="<model>"
) %}
{% set dev_relation = ref('<model>') %}

{{ audit_helper.quick_are_relations_identical(
    a_relation = prod_relation,
    b_relation = dev_relation,
    columns = None  -- or explicit list
) }}
```

**Output:** Single row, single column: `are_tables_identical` (boolean)

---

## compare_row_counts

**Best for:** Step 4 sanity check — fast, cheap, no primary key needed.

```sql
{{ audit_helper.compare_row_counts(
    a_relation = prod_relation,
    b_relation = dev_relation
) }}
```

**Output:**

| relation_name | total_records |
|---|---|
| prod_db.analytics.fct_orders | 34,231 |
| dev_db.dbt_devon.fct_orders | 34,231 |

---

## compare_which_relation_columns_differ

**Best for:** Step 6 triage — identify which columns have any value-level differences.

```sql
{{ audit_helper.compare_which_relation_columns_differ(
    a_relation = prod_relation,
    b_relation = dev_relation,
    primary_key_columns = ["order_id"],
    columns = None
) }}
```

**Output:**

| column_name | has_difference |
|---|---|
| order_id | False |
| amount | True |
| status | False |

Only considers records present in **both** relations (i.e., excludes added/removed rows).

---

## compare_column_values

**Best for:** Step 6 drill-down — understand the magnitude and nature of differences in one column.

```sql
{{ audit_helper.compare_column_values(
    a_query = "SELECT * FROM prod_db.analytics.fct_orders",
    b_query = "SELECT * FROM {{ ref('fct_orders') }}",
    primary_key = "order_id",
    column_to_compare = "status",
    emojis = true,
    a_relation_name = "prod",
    b_relation_name = "dev"
) }}
```

**Output:**

| match_status | count | percent_of_total |
|---|---|---|
| ✅: perfect match | 37,721 | 79.03 |
| ✅: both are null | 5,789 | 12.13 |
| 🤷: missing from a | 5 | 0.01 |
| ❌: values do not match | 4,064 | 8.51 |

**To iterate over all differing columns:**

```sql
{% set columns_to_check = ['amount', 'status', 'customer_id'] %}

{% for col in columns_to_check %}
    {{ log('--- Comparing column: ' ~ col ~ ' ---', info=True) }}
    {% set audit_query = audit_helper.compare_column_values(
        a_query = "SELECT * FROM prod_db.analytics.fct_orders",
        b_query = "SELECT * FROM {{ ref('fct_orders') }}",
        primary_key = "order_id",
        column_to_compare = col
    ) %}
    {% set results = run_query(audit_query) %}
    {% if execute %}
        {% do results.print_table() %}
    {% endif %}
{% endfor %}
```

---

## compare_all_columns

**Best for:** Getting a summary across all columns in one query (alternative to iterating compare_column_values).

```sql
{{ audit_helper.compare_all_columns(
    a_relation = prod_relation,
    b_relation = dev_relation,
    primary_key = "order_id",
    exclude_columns = ["loaded_at", "updated_at"],
    summarize = true  -- false gives row-level detail
) }}
```

**Summarized output:**

| column_name | perfect_match | null_in_a | null_in_b | missing_from_a | missing_from_b | conflicting_values |
|---|---|---|---|---|---|---|
| order_id | 10 | 0 | 0 | 0 | 0 | 0 |
| amount | 2 | 0 | 0 | 0 | 0 | 8 |

---

## compare_relation_columns

**Best for:** Step 3 schema comparison — check for added/removed/type-changed columns.

```sql
{{ audit_helper.compare_relation_columns(
    a_relation = prod_relation,
    b_relation = dev_relation
) }}
```

**Output:**

| column_name | a_data_type | b_data_type | has_data_type_match | in_a_only | in_b_only | in_both |
|---|---|---|---|---|---|---|
| order_id | integer | integer | True | False | False | True |
| order_date | timestamp | date | False | False | False | True |
| new_col | — | varchar | — | False | True | False |

**Flag:**
- `in_a_only = True` → column removed in dev (breaking if downstream depends on it)
- `in_b_only = True` → new column in dev (expected in refactors)
- `has_data_type_match = False` → type changed (investigate)

---

## Legacy Macros

These still work but prefer the newer macros above.

### compare_relations (→ use compare_and_classify_relation_rows instead)

```sql
{{ audit_helper.compare_relations(
    a_relation = prod_relation,
    b_relation = dev_relation,
    exclude_columns = ["loaded_at"],
    primary_key = "order_id",
    summarize = true
) }}
```

### compare_queries (→ use compare_and_classify_query_results instead)

```sql
{{ audit_helper.compare_queries(
    a_query = old_query,
    b_query = new_query,
    primary_key = "order_id",
    summarize = true
) }}
```

---

## Noise Reduction Patterns

Apply these in your query strings when using the `_query_results` variants:

```sql
-- Truncate timestamps to hour (avoids micro-timing differences)
DATE_TRUNC('hour', created_at) AS created_at

-- Round floats/doubles to 2 decimal places (avoids floating point noise)
ROUND(amount::FLOAT, 2) AS amount

-- Handle NULL safely in string comparisons
COALESCE(status, '') AS status
```

## Using with dbt Singular Tests

To add a persistent audit test to your project:

```sql
-- tests/audit_fct_orders.sql
{{ 
  audit_helper.compare_all_columns(
    a_relation=ref('fct_orders'),
    b_relation=api.Relation.create(
        database='prod_db', 
        schema='analytics', 
        identifier='fct_orders'
    ),
    exclude_columns=['updated_at'], 
    primary_key='order_id'
  ) 
}}
where conflicting_values
```

Run with: `dbt test --select fct_orders`
