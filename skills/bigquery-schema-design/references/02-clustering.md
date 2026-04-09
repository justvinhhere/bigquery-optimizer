# Clustering

**Impact:** High

## What This Covers
Clustering columns, column ordering, interaction with partitioning, and automatic re-clustering behavior.

## Why It Matters
Clustering sorts data within each partition (or the whole table if unpartitioned) by the specified columns. This lets BigQuery skip blocks that don't match filter predicates, reducing bytes scanned and improving query speed -- especially on high-cardinality columns where partitioning is impractical.

## Best Practice

**Clustering with partitioning (recommended for large tables):**
```sql
CREATE TABLE `project.dataset.web_events`
(
  event_ts TIMESTAMP,
  user_id INT64,
  country STRING,
  device STRING,
  page_url STRING
)
PARTITION BY DATE(event_ts)
CLUSTER BY country, device, user_id
OPTIONS(description = "Web events clustered by common filter columns");
```

**Clustering without partitioning (tables without a natural partition key):**
```sql
CREATE TABLE `project.dataset.product_catalog`
(
  product_id INT64,
  category STRING,
  brand STRING,
  price NUMERIC
)
CLUSTER BY category, brand
OPTIONS(description = "Product catalog clustered by category and brand");
```

**Column order matters.** Place the most frequently filtered column first:
```sql
-- Good: queries almost always filter on region, sometimes on department
CLUSTER BY region, department, created_date

-- Bad: least selective column first negates clustering benefit
CLUSTER BY created_date, department, region
```

## Edge Cases / Pitfalls
- **Maximum 4 clustering columns.** Choose the top filters/join keys.
- Clustering benefits compound with partitioning but work independently too. A clustered-only table is valid.
- BigQuery **automatically re-clusters** in the background at no cost. No manual maintenance needed.
- Clustering is most effective on columns used in `WHERE`, `JOIN ON`, and `GROUP BY`. Columns only in `SELECT` provide no clustering benefit.
- For columns with very low cardinality (e.g., boolean), clustering provides minimal block-skipping. Prefer higher-cardinality columns.
- Clustering column order is visible via `INFORMATION_SCHEMA.COLUMNS` (`clustering_ordinal_position` column). Use `bq show dataset.table` as a quick CLI alternative.
