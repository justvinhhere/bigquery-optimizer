# WhereOrder

**Severity:** Low (advisory -- BigQuery's cost-based optimizer reorders predicates independently)

## What It Detects
`WHERE` clauses with `AND`-connected predicates that are not ordered from most selective (cheapest) to least selective (most expensive).

## Optimal Predicate Order (Most to Least Selective)

| Priority | Operator | Cost Value |
|----------|----------|------------|
| 1 | `=` (equality) | 1 |
| 2 | `>`, `<` (comparison) | 2 |
| 3 | `>=`, `<=` (inclusive comparison) | 3 |
| 4 | `!=`, `<>` (not equal) | 5 |
| 5 | `LIKE` (pattern matching) | 6 |

## Before
```sql
SELECT repo_name, id, ref
FROM `bigquery-public-data.github_repos.files`
WHERE ref LIKE '%master%'
  AND repo_name = 'cdnjs/cdnjs'
```

## After
```sql
SELECT repo_name, id, ref
FROM `bigquery-public-data.github_repos.files`
WHERE repo_name = 'cdnjs/cdnjs'
  AND ref LIKE '%master%'
```

## Complex Example

**Before:**
```sql
SELECT col1 FROM tbl1
WHERE col4 LIKE '%a%'
  AND col1 = 1
  AND col2 > 1
  AND col6 >= 1
  AND col5 != 1
  AND col7 <= 1
  AND col3 < 1
  AND col5 <> 1
```

**After:**
```sql
SELECT col1 FROM tbl1
WHERE col1 = 1
  AND col3 < 1
  AND col2 > 1
  AND col6 >= 1
  AND col7 <= 1
  AND col5 != 1
  AND col5 <> 1
  AND col4 LIKE '%a%'
```

## Edge Cases
- This is a heuristic based on operator type, not actual data selectivity. Real-world impact varies -- a `!=` on a boolean may be more selective than `=` on a high-cardinality column. Use as a tiebreaker, not a strict rule.
- Only applies to predicates connected by `AND`. `OR`-connected predicates are not reorderable for this optimization.
- Does not apply to predicates on different tables in a JOIN condition.
- Function calls in predicates (e.g., `LOWER(col) = 'x'`) are not ranked by this heuristic.
