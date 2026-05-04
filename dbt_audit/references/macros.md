# audit_helper macro reference

Signatures, output schemas, and notes. Read this when SKILL.md doesn't
have the detail you need.

## Macros, in order of usual use

1. `quick_are_relations_identical` — fast pass/fail
2. `compare_relation_columns` — schema diff
3. `compare_row_counts` — row count sanity
4. `compare_and_classify_relation_rows` — primary row-level comparison
5. `compare_and_classify_query_results` — same, but with custom SQL
6. `compare_all_columns` — per-column summary
7. `compare_column_values` — per-column distribution drilldown
8. `compare_which_relation_columns_differ` — column triage (alternative to 6)

---

## quick_are_relations_identical

Snowflake / BigQuery only. Returns one boolean.

```sql
{{ audit_helper.quick_are_relations_identical(
    a_relation = adapter.get_relation(database='prod_db', schema='analytics', identifier='fct_orders'),
    b_relation = ref('fct_orders'),
    columns = None
) }}
```

Output: a single row with `are_tables_identical` (boolean).

---

## compare_relation_columns

Schema-level diff. Use it as the first deep check after the quick pass/fail.

```sql
{{ audit_helper.compare_relation_columns(
    a_relation = prod_relation,
    b_relation = dev_relation
) }}
```

Output:

| column_name | a_data_type | b_data_type | has_data_type_match | in_a_only | in_b_only | in_both |
| --- | --- | --- | --- | --- | --- | --- |
| order_id | integer | integer | true | false | false | true |
| order_date | timestamp | date | false | false | false | true |
| new_col | — | varchar | — | false | true | false |

Flag:

- `in_a_only = true` → column removed in dev (breaking if downstream relies on it).
- `in_b_only = true` → new column in dev (expected for additive PRs).
- `has_data_type_match = false` → type changed (investigate).

---

## compare_row_counts

Cheap, no primary key required.

```sql
{{ audit_helper.compare_row_counts(
    a_relation = prod_relation,
    b_relation = dev_relation
) }}
```

Output:

| relation_name | total_records |
| --- | --- |
| prod_db.analytics.fct_orders | 34,231 |
| dev_db.dbt_devon.fct_orders | 34,231 |

If the model uses `limit_build` or is incremental, scope both queries to
the same date window first by switching to
`compare_and_classify_query_results`.

---

## compare_and_classify_relation_rows

The primary row-level audit. Use when both relations have the same column
names (no rename refactors).

```sql
{{ audit_helper.compare_and_classify_relation_rows(
    a_relation = prod_relation,
    b_relation = dev_relation,
    primary_key_columns = ['order_id'],   -- required, list of strings
    columns = None,                        -- None auto-detects intersecting cols
    sample_limit = 20                      -- rows per status in output
) }}
```

Output columns:

- All columns of the model (so you can see the actual values).
- `dbt_audit_in_a` — bool, row exists in prod.
- `dbt_audit_in_b` — bool, row exists in dev.
- `dbt_audit_row_status` — one of `identical | added | removed | modified`.
- `dbt_audit_num_rows_in_status` — count of PKs with that status.
- `dbt_audit_sample_number` — sample index within the status.

Notes:

- A `modified` row appears twice in the output (the prod values and the
  dev values), but is counted once in `dbt_audit_num_rows_in_status`.
- If a column was added in dev, pass an explicit `columns` list that
  excludes it, otherwise the intersect that drives the macro fails.

---

## compare_and_classify_query_results

Same output schema as the relation variant, but takes two query strings.
Use when:

- A column was renamed and you need an alias.
- You need to scope to a date window (`limit_build`).
- You need to apply noise reduction (timestamp truncation, float rounding).
- The PK was renamed.

```sql
{% set prod_query %}
    SELECT
        id            AS order_id,                    -- handle rename
        amount,
        customer_id,
        DATE_TRUNC('hour', created_at) AS created_at  -- noise reduction
    FROM prod_db.analytics.fct_orders
    WHERE created_at >= '2026-04-27'                  -- match dev's window
{% endset %}

{% set dev_query %}
    SELECT
        order_id,
        amount,
        customer_id,
        DATE_TRUNC('hour', created_at) AS created_at
    FROM {{ ref('fct_orders') }}
    WHERE created_at >= '2026-04-27'
{% endset %}

{{ audit_helper.compare_and_classify_query_results(
    a_query = prod_query,
    b_query = dev_query,
    primary_key_columns = ['order_id'],
    columns = ['order_id', 'amount', 'customer_id', 'created_at'],  -- required
    sample_limit = 20
) }}
```

---

## compare_all_columns

Single-query alternative to looping `compare_column_values`. Prefer this
for the column drilldown step — one warehouse round-trip instead of N.

```sql
{{ audit_helper.compare_all_columns(
    a_relation = prod_relation,
    b_relation = dev_relation,
    primary_key = 'order_id',
    exclude_columns = ['loaded_at', 'updated_at'],
    summarize = true
) }}
```

Summarized output:

| column_name | perfect_match | null_in_a | null_in_b | missing_from_a | missing_from_b | conflicting_values |
| --- | --- | --- | --- | --- | --- | --- |
| order_id | 10 | 0 | 0 | 0 | 0 | 0 |
| amount | 2 | 0 | 0 | 0 | 0 | 8 |
| status | 8 | 0 | 0 | 0 | 0 | 2 |

Set `summarize = false` for row-level detail.

---

## compare_column_values

Distribution of mismatch types for a single column. Use to drill into a
column flagged by `compare_all_columns`.

```sql
{{ audit_helper.compare_column_values(
    a_query = "SELECT * FROM prod_db.analytics.fct_orders",
    b_query = "SELECT * FROM " ~ ref('fct_orders'),
    primary_key = 'order_id',
    column_to_compare = 'amount'
) }}
```

Output:

| match_status | count | percent_of_total |
| --- | --- | --- |
| perfect match | 37,721 | 79.03 |
| both are null | 5,789 | 12.13 |
| missing from a | 5 | 0.01 |
| values do not match | 4,064 | 8.51 |

---

## compare_which_relation_columns_differ

Returns a yes/no per column for whether *any* value differs in the rows
present in both relations. Cheaper than `compare_all_columns` when you
only need triage, not a breakdown.

```sql
{{ audit_helper.compare_which_relation_columns_differ(
    a_relation = prod_relation,
    b_relation = dev_relation,
    primary_key_columns = ['order_id'],
    columns = None
) }}
```

| column_name | has_difference |
| --- | --- |
| order_id | false |
| amount | true |
| status | false |

Considers only rows present in both relations (excludes added / removed).

---

## Noise reduction patterns

Apply these inside the SELECTs you pass to the `_query_results` variants
(or in a CTE wrapping the relation). Always log when applied.

```sql
-- Timestamps: pipeline timing differences create false positives
DATE_TRUNC('hour', created_at) AS created_at

-- Floats / decimals: warehouse precision noise
ROUND(amount::float, 2) AS amount

-- Strings: NULL vs empty string mismatches
COALESCE(status, '') AS status

-- Arrays / JSON: serialize for stable comparison (Snowflake)
ARRAY_TO_STRING(ARRAY_SORT(tags), ',') AS tags_normalized
```

Pick the *coarsest* normalization that still preserves the semantic you
care about. If a refactor was supposed to change a value's precision,
don't paper over it with `ROUND(_, 0)`.

---

## Choosing the right macro

| Question | Macro |
| --- | --- |
| "Are these two tables byte-for-byte identical?" (Snowflake/BQ) | `quick_are_relations_identical` |
| "Did the schema change?" | `compare_relation_columns` |
| "Are the row counts the same?" | `compare_row_counts` |
| "How many rows changed, and what do the changes look like?" | `compare_and_classify_relation_rows` |
| "...with a column rename / date scope / noise reduction" | `compare_and_classify_query_results` |
| "Which columns are driving the mismatch?" | `compare_all_columns` (summarize) |
| "What does the mismatch in this one column look like?" | `compare_column_values` |
