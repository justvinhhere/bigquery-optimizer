# MERGE DML

**Impact:** High

## What This Covers

The MERGE statement for atomic upserts, conditional updates, deletes, and SCD Type 2 patterns in BigQuery.

## Why It Matters

MERGE combines INSERT, UPDATE, and DELETE into a single atomic statement, reducing round-trips and ensuring consistency. It is the standard pattern for incremental loads and slowly changing dimensions.

## Example

```sql
-- Basic upsert pattern
MERGE INTO `project.dataset.customers` AS target
USING `project.dataset.staging_customers` AS source
ON target.customer_id = source.customer_id
WHEN MATCHED THEN
  UPDATE SET
    target.name = source.name,
    target.email = source.email,
    target.updated_at = CURRENT_TIMESTAMP()
WHEN NOT MATCHED THEN
  INSERT (customer_id, name, email, created_at, updated_at)
  VALUES (source.customer_id, source.name, source.email,
          CURRENT_TIMESTAMP(), CURRENT_TIMESTAMP());

-- SCD Type 2 (two-step pattern):
-- Step 1: Expire changed rows
MERGE INTO `project.dataset.customers_scd2` AS target
USING `project.dataset.staging_customers` AS source
ON target.customer_id = source.customer_id
   AND target.is_current = TRUE
WHEN MATCHED AND (target.name != source.name OR target.email != source.email) THEN
  UPDATE SET
    target.is_current = FALSE,
    target.valid_to = CURRENT_TIMESTAMP()
WHEN NOT MATCHED BY TARGET THEN
  INSERT (customer_id, name, email, is_current, valid_from, valid_to)
  VALUES (source.customer_id, source.name, source.email,
          TRUE, CURRENT_TIMESTAMP(), TIMESTAMP('9999-12-31'));

-- Step 2: Insert new current rows for changed records
-- Join staging to target where no current row exists (Step 1 just expired it).
INSERT INTO `project.dataset.customers_scd2`
  (customer_id, name, email, is_current, valid_from, valid_to)
SELECT s.customer_id, s.name, s.email, TRUE, CURRENT_TIMESTAMP(), TIMESTAMP('9999-12-31')
FROM `project.dataset.staging_customers` s
WHERE NOT EXISTS (
  SELECT 1 FROM `project.dataset.customers_scd2` t
  WHERE t.customer_id = s.customer_id
    AND t.is_current = TRUE
)
AND EXISTS (
  SELECT 1 FROM `project.dataset.customers_scd2` t
  WHERE t.customer_id = s.customer_id
);

-- Delete unmatched rows in target
MERGE INTO `project.dataset.products` AS target
USING `project.dataset.active_products` AS source
ON target.product_id = source.product_id
WHEN NOT MATCHED BY SOURCE THEN
  DELETE;
```

## Edge Cases / Pitfalls

- **Duplicate match keys:** If the source produces multiple rows matching the same target row, MERGE fails with a runtime error. Always deduplicate the source first.
- **DML concurrency:** BigQuery does not enforce a hard daily DML quota. However, at most 2 concurrent mutating DML statements (UPDATE/DELETE/MERGE) run per table, with up to 20 queued as PENDING. Plan accordingly for high-frequency merge pipelines.
- **Column-level lineage:** MERGE does not support `SELECT *` in the INSERT clause. You must list columns explicitly.
- **Partitioned tables:** MERGE on partitioned tables can modify multiple partitions in one statement.
- **Streaming buffer (legacy insertAll API):** MERGE fails with a runtime error if matched target rows are in the streaming buffer. Wait for the buffer to flush (~30 min). The `_PARTITIONTIME IS NOT NULL` filter only excludes buffered rows from source-side scans, not from target matches.
- **Storage Write API:** No streaming buffer restriction. MERGE on recently written rows is fully supported.
