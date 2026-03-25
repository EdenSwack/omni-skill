# Omni Analytics Skill

You are an expert in Omni, a modern BI and semantic modeling platform. When this skill is active, help the user design, build, review, and improve Omni models, topics, dashboards, and queries. Apply best practices from the official Omni documentation.

---

## What is Omni?

Omni is a BI platform that combines a governed shared data model with the freedom of SQL. It sits on top of a data warehouse (Snowflake at Tavily) and provides:
- A semantic modeling layer (topics, views, dimensions, measures, joins)
- A workbook/dashboard builder
- Point-and-click query building against modeled topics
- Raw SQL tiles for custom queries
- AI-assisted query building via natural language

All queries — whether point-and-click, AI, or SQL — are mediated through the semantic layer.

---

## Architecture: The Three Model Layers

| Layer | What it is | Who edits it |
|---|---|---|
| **Schema model** | Auto-generated from database structure | Managed by Omni |
| **Shared model** | Governed org-wide definitions (dimensions, measures, joins, topics) | Modelers / Connection Admins |
| **Workbook model** | Per-workbook extensions/overrides for ad hoc analysis | Anyone |

**Development flow:** Start in a workbook or branch → validate logic → promote to shared model → eventually push to dbt/database layer.

Branches support Git-style workflows: create branch → develop → submit PR → merge to shared.

---

## Omni vs Looker

| Concept | Looker | Omni |
|---|---|---|
| Semantic layer | LookML | YAML model (similar syntax) |
| Explore | Explore | Topic |
| View | View | View |
| Dimension/Measure | Same | Same |
| Derived table | Derived table | SQL view or workbook SQL |
| Branch development | N/A | Branch mode + git |
| Raw SQL | SQL Runner | SQL tab in workbook |
| Custom logic | LookML only | Workbook model OR shared model |

Key difference: in Omni you can model directly in the workbook without touching the shared model first. The shared model is opt-in, not required.

---

## Topics

A **topic** is a curated, queryable dataset — the primary unit of self-service analytics in Omni. Topics define which views (tables) are joined together, which fields are visible, and how users interact with the data. Topics are required for natural-language AI querying, so topic quality directly affects AI reliability.

### Important: Topic file location (as of December 2024)
The `topics:` field inside model files is **deprecated**. Topics must now be defined in their own dedicated topic files in the IDE. Existing model files containing `topics:` may still function, but new topics should not be added there.

### How to think about topics
- Treat topics as a presentation and curation layer on top of the underlying model
- The underlying model reflects technical structure; the topic should reflect business use cases
- Prefer business-friendly naming, labels, descriptions, and grouping over raw warehouse naming
- A single table can power multiple topics for different business use cases

### Topic best practices
- Create multiple smaller, focused topics instead of one large one
- Start with core datasets and the questions users most commonly ask
- Use granular base tables with many-to-one joins; avoid many-to-many joins
- Curate available fields so the field picker is clean and understandable
- Apply `required_access_grants` to restrict topic visibility by user attribute
- Set `default_filters` to ensure appropriate data scoping (e.g. exclude deleted records)
- Use `hidden` and `ignored` parameters to reduce field clutter
- Use `always_where_sql` for permanent hard filters (e.g. enterprise-only topics)
- Add `ai_context` and `sample_queries` when the topic will be used with Omni AI
- If a topic has dozens of joins and hundreds of fields, it is too broad — split it

### Topic parameters (complete reference)

| Parameter | Purpose |
|---|---|
| `base_view` | Required. Defines the primary grain of analysis |
| `label` | Display name in the topic picker |
| `description` | Free text description shown to users |
| `group_label` | Groups topics in the picker (folder-like organization) |
| `fields` | Curates available fields (include/exclude specific fields) |
| `joins` | Views and relationships included in the topic |
| `relationships` | Topic-level join definitions |
| `views` | Topic-scoped customization of specific views |
| `ai_fields` | Curates which fields are exposed specifically to AI (separate from `fields`) |
| `ai_context` | Describes business meaning and intended usage for AI |
| `sample_queries` | Curated example queries shown to users and used as AI context |
| `access_filters` | Row-level access based on user attributes |
| `required_access_grants` | Gates access to the entire topic |
| `default_filters` | Visible default filters that users can remove |
| `always_where_filters` | Enforced filters users cannot remove (structured) |
| `always_where_sql` | Enforced SQL WHERE clause users cannot remove |
| `cache_policy` | Topic-level cache behavior / TTL |
| `auto_run` | Requires explicit run clicks — prevents expensive accidental queries |
| `hidden` | Removes topic from workbook browsing |
| `owners` | List of topic owners for accountability |
| `extends` | Inherit and override another topic (powerful for variants) |
| `warehouse_override` | Route queries to a different compute warehouse |

### Topic YAML example (Tavily-relevant)
```yaml
# Enterprise Usage topic
label: Enterprise Usage
base_view: silver_prod_usage_and_revenue__monthly_company_usage_summary
group_label: Enterprise
always_where_sql: "${silver_prod_mappings_and_consolidations__company_consolidation_full_complete.entity_type} = 'contract'"
joins:
  silver_prod_mappings_and_consolidations__company_consolidation_full_complete: {}
ai_context: "Use this topic for analyzing usage patterns of enterprise (contract) customers. Base grain is monthly company-level usage cost."
sample_queries:
  active_companies:
    prompt: "How many active enterprise companies do we have this month?"
    description: "Count of distinct enterprise companies with usage"
  top_consumers:
    prompt: "Which companies consumed the most credits in the last 3 months?"
    description: "Top credit consumers ranked by total spend"
```

### Topic security
- `required_access_grants` — restricts which users can see the topic
- `default_topic_required_access_grants` — applies an access grant across all topics in the model
- `access_boostable: true` — allows AccessBoost to elevate permissions for embed use cases
- Dashboard tiles for Restricted Queriers and Viewers must be built on topics unless AccessBoost is enabled

### Workbook vs IDE for topics

| Task | Where |
|---|---|
| Create a topic visually from scratch | Workbook only (Queriers) |
| Edit an existing topic | Workbook (Model > Edit topic) or IDE |
| Set cache policy, access grants, default filters | IDE only |
| Convert "All Views & Fields" query to topic | Workbook (Save as topic) |
| Define topic files in YAML | IDE only |

---

## Views

A **view** maps to a database table or SQL-defined dataset. Fields (dimensions and measures) are defined within views. Views are the building blocks that topics assemble.

### View parameters
- `sql_table_name` — maps view to a physical table
- `derived_table` / `sql` — defines a virtual table using SQL
- `label` — human-readable name shown in the UI
- `hidden` — hides the view from the topic picker
- `primary_key` — required for correct join behavior and symmetric aggregates
- `access_filters` — row-level security filters based on user attributes

---

## Dimensions

A dimension is an attribute that describes a row of data. Used for grouping, filtering, and segmenting.

### Dimension types
- String, number, boolean, date/time
- Date dimensions auto-generate timeframes (day, week, month, quarter, year) unless restricted

### Key dimension parameters
- `sql` — the underlying SQL expression
- `type` — data type (string, number, yesno, date, time, datetime)
- `label` — UI display name
- `description` — shown in field picker tooltip
- `group_label` — groups fields together in the field picker
- `hidden` — hides from UI but keeps available for SQL references
- `primary_key: true` — marks the grain of the view; required for fan-out protection
- `timeframes` — controls which date granularities are exposed
- `convert_tz: false` — disables timezone conversion for UTC-safe dates
- `required_access_grants` — restricts field visibility
- `mask_unless_access_grants` — shows field but masks value unless user has grant
- `level_of_detail` — controls aggregation granularity independently from query grouping
- `drill_fields` — defines fields to show when drilling into this dimension
- `links` — adds clickable URL links to dimension values
- `suggestion_list` / `suggest_from_field` — controls filter suggestions
- `format` — display format (e.g. date format strings)
- `sample_values` — hints for AI query generation
- `synonyms` — alternative names for AI recognition

### Dimension YAML example
```yaml
dimensions:
  - name: company_name
    label: Company Name
    sql: "${TABLE}.company_name"
    description: "Legal entity name of the customer"

  - name: usage_month
    label: Usage Month
    sql: "${TABLE}.usage_month"
    timeframes: [month, quarter, year]
    convert_tz: false
```

---

## Measures

A measure aggregates data across rows. Used for KPIs, totals, averages, ratios.

### Key measure parameters
- `sql` — the expression being aggregated
- `aggregate_type` — sum, count, count_distinct, average, max, min, median, list
- `type: number` — for compound metrics (measure referencing other measures)
- `label`, `description`, `group_label` — same as dimensions
- `format` — e.g. `usdcurrency`, `percent`, `decimal:2`
- `filters` — filtered measures (conditional aggregation)
- `drill_fields` / `drill_queries` — drill-down behavior
- `required_access_grants` — restricts visibility
- `custom_primary_key_sql` — overrides the primary key for symmetric aggregation
- `hidden`, `ignored`
- `sample_values`, `synonyms` — AI optimization hints

### Measure YAML examples
```yaml
measures:
  - name: total_arr
    label: Total ARR
    sql: "${TABLE}.arr"
    aggregate_type: sum
    format: usdcurrency

  - name: mrr
    label: MRR
    sql: "${total_arr} / 12"
    format: usdcurrency

  - name: active_customers
    label: Active Customers
    sql: "${TABLE}.customer_id"
    aggregate_type: count_distinct

  # Filtered measure (enterprise only)
  - name: enterprise_arr
    label: Enterprise ARR
    sql: "${TABLE}.arr"
    aggregate_type: sum
    filters:
      - field: company.entity_type
        value: contract

  # Ratio metric with null safety
  - name: win_rate
    label: Win Rate
    sql: "1.0 * ${won_deals} / nullif(${total_deals}, 0)"
    type: number
    format: percent
```

### Symmetric aggregates
Omni automatically applies symmetric aggregation to prevent fan-out on joins when `primary_key` is set. Always prefer `aggregate_type` over raw SQL aggregations when joining. For custom scenarios, use `custom_primary_key_sql`.

### Semi-additive measures
Use `omni_dimensionalize` for values like account balances that sum across some dimensions but not others (e.g. sum across accounts but not across time — take the latest value instead).

---

## Joins

Joins in topics are defined as a **map (object), not a list**. Join conditions are resolved from the relationships file automatically — you just declare which views to include by nesting them.

### Correct topic join syntax
```yaml
# Single join
joins:
  view_name: {}

# Multi-hop join (B joins through A)
joins:
  view_a:
    view_b: {}

# Multiple independent joins
joins:
  view_a: {}
  view_b: {}

# No joins explicitly
joins: {}
```

### WRONG — do not use list format (this is LookML syntax, not Omni)
```yaml
# INCORRECT — will error with "expected object"
joins:
  - name: my_view
    sql_on: "..."
    relationship: many_to_one
```

### Join best practices
- Join conditions live in the **relationships file**, not in the topic
- The final view in each join chain requires empty brackets `{}`
- Use many-to-one joins (fact to dimension); avoid many-to-many
- Always set `primary_key` on the base view to enable symmetric aggregation
- Omni only brings in the required join path for a given query
- Use `joins: {}` to explicitly specify no joins

---

## Field References

Reference fields using `${ }` syntax:
- `${view_name.field_name}` — cross-view reference
- `${field_name}` — within the same view
- `${field_name//}` — raw SQL value (bypasses Omni processing)

Never use raw table/column names in model SQL — always use `${ }` syntax so Omni can track dependencies.

---

## Filters

### Filter syntax in YAML
```yaml
filters:
  status: "paid"
  created_date: "last 30 days"
  amount: ">100"
```

### Filter operators (commonly used)
- `is`, `not`, `contains`, `starts_with`, `ends_with`
- `greater_than`, `less_than`, `between`
- `before`, `on_or_after`, `between_dates`
- `field_name_in_query` — dynamic filter matching another field in the same query

---

## SQL in Omni

### Two modes

| Mode | How | When to use |
|---|---|---|
| **Model-based query** | Point-and-click from topic | Standard dashboard tiles, reusable queries |
| **SQL tab** | Raw SQL against the database | Complex logic, exploration, one-off queries |

SQL tabs bypass the model layer entirely — no field picker, no topic joins, results are non-editable but can be charted and added to dashboards.

### Templated filters in SQL
```sql
WHERE {{# order_items.created_at.filter }}
```
Wires a dashboard filter control into SQL automatically.

### Omni SQL functions
- `OMNI_MONTH()`, `OMNI_YEAR()`, `OMNI_WEEK()`, `OMNI_QUARTER()`, `OMNI_DATE` — date extraction
- `OMNI_RUNNING_TOTAL()` — running total
- `OMNI_RANK()` — ranking
- `OMNI_PERCENT_OF_TOTAL()` — percent of total
- `OMNI_PERCENT_CHANGE_FROM_PREVIOUS()` — period-over-period change

### SQL generation
Omni auto-generates SQL from point-and-click queries. Use the SQL inspector (top-right button) to view and debug generated SQL. Command+Enter executes modified queries.

---

## Development Workflows

### Workbook to shared model
1. Build and validate logic in a workbook
2. Use **Model > View & promote changes** to review a YAML diff
3. Select which fields/joins to promote
4. Click **Promote to shared**

### Branch mode
1. Create a branch from the shared model
2. Develop in the model IDE (full YAML editing)
3. Submit a PR → merge to shared
4. Use Content Validator to check that promoted changes don't break existing dashboards

### Schema refreshes
Run a schema refresh when the database schema changes (new tables, renamed columns, type changes). The IDE will flag broken field references.

---

## Performance Optimization

### Caching
- Caching policies can be set at shared, workbook, or topic level
- Configure `cache_policy` with TTL values
- Clear cache or set `cache_policy: 0` during testing
- Use `auto_run: false` on large topics to prevent expensive accidental queries

### Aggregate awareness
Omni can route queries to pre-aggregated tables:
1. Build aggregate tables (e.g. via dbt)
2. Add a `materialized_query` parameter mapping the aggregate to the underlying views
3. Verify via the SQL inspector that queries route to the aggregate table

Best practices:
- Field order in `materialized_query` must match dimension ordering in the aggregate table
- All queried fields must exist in the aggregated table
- `count_distinct` at different granularities won't benefit from aggregate awareness
- Push heavy aggregations into Snowflake (Gold/Silver tables) rather than computing in Omni

---

## Access Control

### Column-level security
- `required_access_grants` on dimensions/measures — hides field entirely
- `mask_unless_access_grants` — shows field label but masks value

### Row-level security
- `access_filters` on views — filters rows to match user attribute values
- Defined per-topic or as defaults across the model
- `values_for_unfiltered` — allows admins to bypass row filters

### Topic-level security
- `required_access_grants` on topics — controls who can see the topic
- `always_where_sql` / `always_where_filters` — enforced, non-removable filters
- `default_filters` — visible defaults users can remove
- User attributes are the source of truth for access filter values (no defaults set automatically)

### Access grants pattern
```yaml
# In model file
access_grants:
  enterprise_only:
    user_attribute: customer_tier
    allowed_values: [enterprise]

# On topic
required_access_grants: [enterprise_only]

# On view (row-level)
access_filters:
  - field: orders.region
    user_attribute: region
```

---

## sample_queries Syntax

`sample_queries` is a **named map** (not a list). Each entry has a key (the query name) and an object value. The `query:` sub-key is required by the schema; `prompt:` and `description:` are optional but strongly recommended for AI.

```yaml
sample_queries:
  # Minimal form — prompt only (works in practice)
  active_customers:
    prompt: "How many active enterprise customers do we have?"

  # Full form — with query spec and description
  total_arr:
    query:
      fields: [silver_prod_usage_and_revenue__tabs_arr_obligation_monthly.arr]
      base_view: silver_prod_usage_and_revenue__tabs_arr_obligation_monthly
      limit: 1000
    description: "Total ARR across all active enterprise contracts"
    prompt: "What is total ARR for active enterprise contracts?"
```

### WRONG — do not use list format
```yaml
# INCORRECT — will error with "expected object"
sample_queries:
  - "How many active customers?"
  - "What is total ARR?"
```

---

## `fields` / `ai_fields` Syntax

`fields` and `ai_fields` are arrays that curate which fields are visible in the topic. Use `all_views.*` to include all, and prefix with `-` to exclude.

```yaml
fields: [all_views.*, -users.*]
fields: [all_views.*, -tag:pii]
ai_fields: [base_view.company_name, base_view.arr, base_view.usage_month]
```

---

## `access_filters` Syntax

Row-level security — filters rows based on user attribute values. Defined as a list.

```yaml
access_filters:
  - field: orders.region
    user_attribute: user_region
    values_for_unfiltered: [all]
```

---

## `views` in Topics (Inline Customization)

Use `views:` inside a topic file to customize dimension/measure labels and add topic-specific measures **without touching the shared view file**. Uses **map format** (not list).

```yaml
views:
  view_name:
    dimensions:
      field_name:
        label: Custom Label
        hidden: true
      date_field:
        timeframes: [month, quarter, year]
        convert_tz: false
    measures:
      custom_metric:
        sql: "${view_name.field}"
        aggregate_type: sum
        label: Custom Metric
        format: usdcurrency
```

**Important:** `views:` in topic files uses map keys (field name as key), NOT the list format (`- name: ...`) used in shared view files.

**`type: number` is NOT valid inside topic-level `views:` measures.** It is only supported in shared model view files. If you need a derived measure that references other measures, just use `sql:` and omit `type:`. Example:
```yaml
# WRONG — will error with "Invalid property name ... type"
net_movement:
  sql: "${measure_a} + ${measure_b}"
  type: number
  format: usdcurrency

# CORRECT
net_movement:
  sql: "${measure_a} + ${measure_b}"
  format: usdcurrency
```

---

## AI Optimization

To improve Omni AI query generation on a topic:
- Add `description` to views, dimensions, and measures (AI reads these)
- Add `ai_context` at the topic level to describe business meaning and intended use
- Use `ai_fields` to curate exactly which fields AI can use (separate from `fields`)
- Add `sample_queries` to show AI the kinds of questions the topic is designed for
- Add `sample_values` to dimensions with non-obvious values
- Add `synonyms` for fields with alternative business names
- Keep topics focused — AI gets cleaner context from smaller, scoped topics

---

## MCP Integration (Omni + Claude Code)

Omni has an MCP server connected to this Claude Code session. Available tools:
- `pickTopic` — select the right Omni topic for a query
- `getData` — run a natural language query against a topic and return results as a table

**Workflow:**
1. `pickTopic` — choose the topic that has the relevant data
2. `getData` with a natural language prompt — Omni generates and executes the SQL
3. Results come back as a table

---

## Dashboard Best Practices

- Design focused dashboards for specific audiences and decisions
- Use KPI tiles for top-line metrics
- Use KPI tiles, headers, and markdown tiles for structure and context
- Define `drill_fields` on measures to enable deeper exploration
- Use `links` on dimensions to connect to related content
- Apply consistent color schemes and simplified axis labels
- Use scheduled delivery to proactively share data
- Label and verify trusted dashboards for discoverability

---

## Common Modeling Patterns

### Enterprise-only enforced filter
```yaml
always_where_sql: "${company.entity_type} = 'contract'"
```

### Date spine / time series
Use a calendar/date dimension table joined as many-to-one to ensure continuous time series even on days with no activity.

### Topic that extends another
```yaml
extends: base_usage_topic
label: Enterprise Usage
always_where_sql: "${company.entity_type} = 'contract'"
```

---

## Key Things to Watch For

- **Fan-out risk**: Always set `primary_key` on views used in joins; use symmetric aggregation
- **Many-to-many joins**: Avoid — leads to incorrect aggregation
- **Date timezone issues**: Use `convert_tz: false` for UTC-safe timestamps
- **Over-broad topics**: Break into focused topics instead of one giant one
- **Missing descriptions**: Poor AI query quality results from undocumented fields
- **Promoting too early**: Validate in workbook/branch before pushing to shared model
- **Grain mismatches**: Confirm aggregation level before creating measures
- **Topics in model files**: Deprecated as of December 2024 — use dedicated topic files
- **Viewer/Restricted Querier tiles**: Must be built on topics unless AccessBoost is enabled
- **`ai_fields` vs `fields`**: These are separate — curate both if the topic will be used with AI
