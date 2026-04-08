# BigQuery Expert

A comprehensive BigQuery plugin for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Five integrated skill areas that activate automatically -- writing queries, designing schemas, optimizing costs, detecting anti-patterns, and navigating BigQuery-specific features.

## Skills

| Skill | Coverage |
|-------|----------|
| **Query Generation** | Generate optimized SQL from natural language. Convert queries from PostgreSQL, MySQL, Snowflake, Redshift, and SQL Server. |
| **Query Optimization** | Detect and fix 11 SQL anti-patterns with before/after rewrites. Project-wide scanning across `.sql` and code files. |
| **Schema Design** | Partitioning (time-unit, integer-range, ingestion-time), clustering, nested/repeated fields (STRUCT/ARRAY), denormalization, table types, and data type selection. |
| **Cost Optimization** | On-demand vs editions pricing, bytes-billed reduction, slot optimization, materialized views, query caching, storage management, and dry-run estimation. |
| **BigQuery Features** | STRUCT/ARRAY/UNNEST, MERGE DML, scripting, JSON functions, approximate aggregation, geography, BigQuery ML, search indexes, and vector search. |

Skills activate based on context. Ask Claude to write a query and the generation skill engages. Discuss partitioning and the schema design skill kicks in. Multiple skills can activate simultaneously when a request spans areas.

## Installation

### Add the marketplace

```
/plugin marketplace add justvinhhere/bigquery-expert
```

### Install the plugin

```
/plugin install bigquery-expert@justvinhhere-bigquery-expert
```

Or open `/plugin`, go to the **Discover** tab, and select **bigquery-expert**.

### Activate

```
/reload-plugins
```

## Commands

| Command | What It Does |
|---------|-------------|
| `/bigquery-expert:bq-generate` | Generate optimized SQL from a natural language description |
| `/bigquery-expert:bq-review` | Review SQL for performance anti-patterns |
| `/bigquery-expert:bq-optimize` | Rewrite SQL with all detected anti-patterns fixed |
| `/bigquery-expert:bq-design-table` | Design a table schema with partitioning, clustering, and data types |
| `/bigquery-expert:bq-estimate-cost` | Estimate the cost of a query or table |
| `/bigquery-expert:bq-explain` | Explain a BigQuery feature with working examples |

All commands accept a file path, inline SQL, or a description as an argument. Without arguments, they use the most recent SQL in the conversation.

**Examples:**

```
/bigquery-expert:bq-generate "daily active users grouped by country for the last 30 days"
/bigquery-expert:bq-review path/to/query.sql
/bigquery-expert:bq-optimize "SELECT * FROM `project.dataset.events`"
/bigquery-expert:bq-design-table "user click events with timestamp, page URL, and session ID"
/bigquery-expert:bq-estimate-cost path/to/expensive_query.sql
/bigquery-expert:bq-explain "MERGE for upserts"
```

## Agents

Agents run autonomously across your project when you ask naturally:

| Agent | Use When You Say... |
|-------|---------------------|
| **bq-reviewer** | "Review all SQL files in this project for anti-patterns" |
| **bq-schema-advisor** | "Audit my table schemas and recommend partitioning strategies" |
| **bq-cost-analyzer** | "Which queries in this project are the most expensive?" |

## Anti-Pattern Detection

The query optimization skill detects 11 BigQuery SQL anti-patterns:

| # | Pattern | Fix | Severity |
|---|---------|-----|----------|
| 1 | `SELECT *` on single-table query | Specify only needed columns | High |
| 2 | `IN`/`NOT IN` without `DISTINCT` | Add `DISTINCT` to subquery | Medium |
| 3 | CTE referenced multiple times | Convert to `CREATE TEMP TABLE` | High |
| 4 | `ORDER BY` without `LIMIT` | Add `LIMIT` clause | Medium |
| 5 | `REGEXP_CONTAINS` for simple patterns | Use `LIKE` instead | Low |
| 6 | `ROW_NUMBER()` + `WHERE rn = 1` | Use `ARRAY_AGG(... LIMIT 1)` | High |
| 7 | Subquery inside WHERE | Extract to `DECLARE` variable or CTE | Medium |
| 8 | WHERE predicates not ordered by selectivity | Reorder by operator cost (advisory) | Low |
| 9 | Smaller table first in JOIN | Place largest table first (advisory) | Low |
| 10 | `CREATE TEMP TABLE` without `DROP` | Add `DROP TABLE` at end of script | Low |
| 11 | `CREATE TABLE` + `DROP TABLE` in same script | Use `CREATE TEMP TABLE` instead | Low |

Based on [BigQuery Anti-Pattern Recognition](https://github.com/GoogleCloudPlatform/bigquery-antipattern-recognition) by Google Cloud Platform (Apache 2.0).

## Example: Before and After

**Before** -- `ROW_NUMBER()` for latest record per group (High severity):

```sql
SELECT taxi_id, trip_seconds, fare
FROM (
  SELECT taxi_id, trip_seconds, fare,
    ROW_NUMBER() OVER (PARTITION BY taxi_id ORDER BY fare DESC) rn
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips`
)
WHERE rn = 1
```

**After** -- `ARRAY_AGG` (optimized):

```sql
SELECT event.*
FROM (
  SELECT ARRAY_AGG(
    t ORDER BY t.fare DESC LIMIT 1
  )[OFFSET(0)] event
  FROM `bigquery-public-data.chicago_taxi_trips.taxi_trips` t
  GROUP BY t.taxi_id
)
```

## Plugin Contents

```
skills/
  bigquery-optimization/       11 anti-pattern references
  bigquery-query-generation/   Schema-aware generation, common patterns, dialect conversion
  bigquery-schema-design/      Partitioning, clustering, nested fields, denormalization, types
  bigquery-cost-optimization/  Pricing, bytes-billed, slots, materialized views, storage
  bigquery-features/           STRUCT/ARRAY, MERGE, scripting, JSON, geo, BQML, vector search
commands/
  bq-generate, bq-review, bq-optimize, bq-design-table, bq-estimate-cost, bq-explain
agents/
  bq-reviewer, bq-schema-advisor, bq-cost-analyzer
```

## Uninstall

```
/plugin uninstall bigquery-expert@justvinhhere-bigquery-expert
/plugin marketplace remove justvinhhere-bigquery-expert
```

## Contributing

Contributions welcome. Fork the repository, create a feature branch, and submit a pull request.

For bugs and feature requests, [open an issue](https://github.com/justvinhhere/bigquery-expert/issues).

## License

Apache License 2.0 -- see [LICENSE](LICENSE) for details.
