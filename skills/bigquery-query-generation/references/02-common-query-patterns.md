# Common BigQuery Query Patterns

**Impact:** High

## What This Covers
BigQuery-idiomatic patterns for common analytical tasks, using BigQuery-specific syntax and avoiding known anti-patterns.

## Why It Matters
Generic SQL patterns often perform poorly in BigQuery. These idiomatic alternatives leverage the columnar engine and built-in functions for optimal performance.

## Best Practice

### Deduplication -- use ARRAY_AGG, not ROW_NUMBER
```sql
SELECT event.* FROM (
  SELECT ARRAY_AGG(t ORDER BY t.updated_at DESC LIMIT 1)[OFFSET(0)] event
  FROM `project.dataset.events` t
  GROUP BY t.user_id)
```

### Running totals
```sql
SELECT order_date, daily_revenue,
  SUM(daily_revenue) OVER (ORDER BY order_date) AS cumulative_revenue
FROM `project.dataset.daily_sales`
ORDER BY order_date LIMIT 1000
```

### Gap-and-island detection
```sql
SELECT user_id, MIN(event_date) AS island_start, MAX(event_date) AS island_end
FROM (
  SELECT *, DATE_SUB(event_date, INTERVAL CAST(ROW_NUMBER() OVER (
    PARTITION BY user_id ORDER BY event_date) AS INT64) DAY) AS grp
  FROM `project.dataset.user_activity`)
GROUP BY user_id, grp
```

### PIVOT / UNPIVOT
```sql
SELECT * FROM `project.dataset.sales`
PIVOT(SUM(amount) FOR quarter IN ('Q1', 'Q2', 'Q3', 'Q4'))
-- UNPIVOT: UNPIVOT(metric_value FOR metric_name IN (revenue, cost, profit))
```

### Sessionization (30-min gap threshold)
```sql
SELECT *, SUM(new_session) OVER (PARTITION BY user_id ORDER BY event_time) AS session_id
FROM (
  SELECT *, IF(TIMESTAMP_DIFF(event_time, LAG(event_time) OVER (
    PARTITION BY user_id ORDER BY event_time), MINUTE) > 30
    OR LAG(event_time) OVER (PARTITION BY user_id ORDER BY event_time) IS NULL, 1, 0) AS new_session
  FROM `project.dataset.clickstream`)
```

### Funnel analysis (open funnel -- does not enforce step order)
```sql
SELECT COUNTIF(step >= 1) AS views, COUNTIF(step >= 2) AS carts, COUNTIF(step >= 3) AS purchases
FROM (
  SELECT user_id, MAX(IF(event='page_view',1,0)) + MAX(IF(event='add_to_cart',2,0))
    + MAX(IF(event='purchase',3,0)) AS step
  FROM `project.dataset.events` GROUP BY user_id)
-- Note: this open funnel counts users who completed each action independently.
-- Users who skip steps (e.g., purchase without add_to_cart) are still counted.
-- For a strict sequential funnel, use event timestamps to enforce step ordering.
```

### Time-series bucketing
```sql
SELECT TIMESTAMP_TRUNC(event_timestamp, HOUR) AS hour_bucket, COUNT(*) AS event_count
FROM `project.dataset.events`
WHERE event_timestamp BETWEEN TIMESTAMP('2024-01-01') AND TIMESTAMP('2024-02-01')
GROUP BY hour_bucket ORDER BY hour_bucket LIMIT 1000
```

## Edge Cases / Pitfalls
- `PIVOT`/`UNPIVOT` require literal `IN` values. For dynamic pivots, use `EXECUTE IMMEDIATE`.
- Gap-and-island assumes no duplicate dates per user. Deduplicate first if needed.
- Sessionization threshold is domain-specific. Always note the chosen value in assumptions.
