# JoinOrder

**Severity:** Low (advisory -- BigQuery's optimizer usually handles join ordering automatically)

## What It Detects
JOIN operations where a smaller table is on the left (first position) and a larger table is on the right. BigQuery's best practice is to place the largest table first.

## Why It Matters
In broadcast joins, BigQuery sends the smaller table to every worker processing the larger table. The query optimizer uses table position as a hint about relative sizes when statistics are unavailable. Placing the largest table first helps the optimizer choose the most efficient join strategy.

## Before
```sql
SELECT t1.station_id, COUNT(1) num_trips_started
FROM `bigquery-public-data.austin_bikeshare.bikeshare_stations` t1  -- small: 100 rows
JOIN `bigquery-public-data.austin_bikeshare.bikeshare_trips` t2     -- large: 1M+ rows
  ON t1.station_id = t2.start_station_id
GROUP BY t1.station_id
```

## After
```sql
SELECT t1.station_id, COUNT(1) num_trips_started
FROM `bigquery-public-data.austin_bikeshare.bikeshare_trips` t2     -- large: 1M+ rows
JOIN `bigquery-public-data.austin_bikeshare.bikeshare_stations` t1  -- small: 100 rows
  ON t1.station_id = t2.start_station_id
GROUP BY t1.station_id
```

## Edge Cases
- **Table size must be known or estimated.** If table sizes are unknown, flag this as advisory.
- Applies to `JOIN`, `LEFT JOIN`, `RIGHT JOIN`, `FULL JOIN`.
- For `LEFT JOIN`, the left table drives the output -- reordering may change semantics. In that case, recommend switching to `RIGHT JOIN` or restructuring.
- Multiple JOINs: the largest table should be leftmost, with subsequent tables ordered by decreasing size.
