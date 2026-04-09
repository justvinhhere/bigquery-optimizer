# Pricing Models

**Impact:** High

## What This Covers
BigQuery compute pricing models: on-demand, editions (Standard, Enterprise, Enterprise Plus), and free-tier operations.

## Why It Matters
Choosing the wrong pricing model can double your BigQuery spend. On-demand charges per TB scanned; editions charge per slot-hour consumed. The optimal choice depends on query volume, concurrency, and usage patterns.

## Best Practice

### On-Demand ($6.25/TB)
- First 1 TB/month free across all projects in a billing account.
- Cost scales linearly with bytes billed.
- Best for: sporadic workloads, exploration, low-volume pipelines.

### Editions
| Tier | Rate | Key Features |
|------|------|-------------|
| Standard | $0.04/slot-hr | Autoscaling, no commitment required |
| Enterprise | $0.06/slot-hr | CMEK, VPC Service Controls, BQML |
| Enterprise Plus | $0.10/slot-hr | 99.99% SLA, advanced DR, compliance |

- Autoscaling adjusts slot count to demand (pay only for what you use).
- Capacity commitment discounts: 20% (1-yr), 40% (3-yr) for Enterprise/Enterprise Plus. Spend-based CUDs (a separate product) offer ~10% (1-yr), ~20% (3-yr) on eligible compute spend.

### Break-Even Analysis
```
monthly_on_demand = total_TB_scanned * 6.25
monthly_editions  = avg_slots * hours_in_month * rate_per_slot_hr

-- Example: 100 slots sustained on Enterprise
-- 100 * 730 * 0.06 = $4,380/month
-- Equivalent on-demand: $4,380 / 6.25 = ~700 TB/month break-even
```

### Free Operations
- **Cached queries**: identical query text within 24 hrs (on-demand only).
- **Dry runs**: `bq query --dry_run` -- always free.
- **Metadata queries**: `INFORMATION_SCHEMA` views have no storage cost, but queries are billed normally (10 MB minimum on on-demand; slots on editions). Results are never cached.
- **Loading data**: batch loads from GCS are free.

## Edge Cases / Pitfalls
- Flat-rate (legacy) pricing is deprecated. Migrate to editions.
- On-demand uses dynamic concurrency with query queuing (up to 1,000 interactive queries queued per project).
- Editions autoscaling has a ramp-up delay; sudden spikes may queue.
- Cached results bypass cost on-demand but still consume slots in editions.
- Free tier resets monthly; unused free TB does not roll over.
