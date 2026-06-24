# Chapter 11: Cluster Planning, Sizing, and Cost Optimization

Every cluster planning decision touches internals you now understand. The instance family you choose determines the buffer pool size, which determines whether your working set fits in memory — and whether `BufferCacheHitRatio` stays above the 95% threshold from Chapter 10. The reader instance size you select determines whether the writer's purge thread can keep up, or whether a single long-running `SELECT` cascades into a cluster-wide degradation spiral (Chapter 5). The I/O pricing model you select — Standard versus I/O-Optimized — depends on whether your workload's root cause is legitimate data growth or a fixable purge or buffer pool problem (Chapter 9).

This chapter connects architecture knowledge to operational decisions. It provides instance selection guidance, the reader sizing rule that most teams violate, storage and I/O cost modeling, scaling pattern decision trees, and the cost optimization strategies that actually work.

## 11.1 Instance Selection

### Instance Families

Aurora MySQL offers four memory-optimized instance families for production workloads, plus burstable T-series instances that should be restricted to development and testing. The R-series instances provide a 1:8 vCPU-to-RAM ratio suitable for most OLTP workloads, while the X2g family doubles memory per vCPU for workloads with very large working sets.

| Instance Family | Processor | Gen | vCPU:RAM (large) | On-Demand/hr (us-east-1) | Best For |
|:---|:---|:---|:---|:---|:---|
| db.r6g | Graviton2 | Previous | 2:16 GiB | $0.260 [^578^] | Stable workloads, proven track record |
| db.r7g | Graviton3 | Current | 2:16 GiB | ~$0.239 [^583^] | New deployments, best Graviton price-performance |
| db.r6i | Intel Ice Lake | Current | 2:16 GiB | ~$0.290 [^513^] | Workloads requiring x86, specific Intel optimizations |
| db.x2g | Graviton2 | Memory-opt | 2:32 GiB | $0.377 [^592^] | Large working sets, cache-heavy analytics |

The db.r7g family is the default recommendation for new clusters. AWS confirms that Graviton3 delivers up to 30% better performance and up to 27% better price-performance compared to Graviton2 for open-source databases on RDS [^583^], and Graviton2 itself provides 35% better price-performance on Aurora compared to equivalent Intel generations [^588^]. The db.x2g family justifies its premium only when the active dataset exceeds ~75% of available R-family RAM at the desired vCPU count — a condition detectable via `BufferCacheHitRatio` consistently below 95% even on an R-family instance.

Use this query to assess working set fit before choosing an instance family:

```sql
-- Measure buffer pool efficiency and working set pressure
SELECT
    ROUND(
        (1 - (
            (SELECT VARIABLE_VALUE FROM performance_schema.global_status
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_reads') /
            (SELECT VARIABLE_VALUE FROM performance_schema.global_status
             WHERE VARIABLE_NAME = 'Innodb_buffer_pool_read_requests')
        )) * 100, 2
    ) AS buffer_pool_hit_pct,
    ROUND(
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status
         WHERE VARIABLE_NAME = 'Innodb_buffer_pool_pages_data') * 16 / 1024 / 1024, 2
    ) AS data_pages_in_gb;
```

If `buffer_pool_hit_pct` is below 95% and `data_pages_in_gb` is near the instance's buffer pool limit (75% of RAM), move to the next larger R-family size or consider X2g. Do not guess — the CloudWatch `BufferCacheHitRatio` metric exposes this precisely [^596^].

### Graviton Migration: Compelling but Context-Dependent

Migrating from Intel to Graviton is the highest-ROI, lowest-risk optimization available. The process is a simple instance type change during a maintenance window — no data migration required [^536^]. However, the headline savings figures overstate the benefit for Aurora relative to RDS MySQL because Aurora runs three memory-consuming processes (`mysqld`, `csdd`, and HM) versus only two in RDS MySQL (`mysqld` and HM) [^101^]. The `csdd` (cluster storage daemon) and HM (health monitor) processes consume CPU and memory regardless of processor architecture, diluting the per-workload efficiency gains from Graviton's better instruction-per-clock performance. Expect savings at the lower end of the 20–35% range on smaller instances where fixed overhead represents a larger fraction of total resource consumption; the advantage increases with instance size.

Before migrating, verify that all application dependencies are compiled for ARM64. While pure Java, Python, and Go applications typically require no changes, any native C/C++ extension or client library with architecture-specific binaries must be tested.

### The Reader Sizing Rule: Match or Exceed Writer

This is the most consequential and most frequently violated sizing rule in Aurora. A reader instance smaller than the writer creates a purge liability that can increase total cluster cost rather than reducing it.

The mechanism, established in Chapter 5, works as follows. Because all Aurora instances share a single undo log, a read view opened on any reader blocks the writer's purge process for the entire cluster [^331^]. An under-provisioned reader — smaller buffer pool, fewer CPUs — is more likely to fall behind applying redo log records during write-heavy periods. When that reader finally releases an old read view, the writer initiates aggressive purge, generating a surge of I/O operations. The reader, already under-powered, may fail to keep up with the purge storm and be restarted by Aurora [^317^]. Each storage read I/O costs $0.20 per million requests, and a sustained purge storm can generate hundreds of millions of additional I/Os per month — easily exceeding the compute "savings" from running a smaller reader [^554^].

Operational guideline: reader instances should be the same class as the writer minimum, and one size larger if they serve heavy read traffic. A `db.r6g.xlarge` writer with a `db.r6g.large` reader saves approximately $190 per month in compute but risks thousands in I/O overages and operational incidents. The correct framing treats reader sizing as financial risk management, not just performance tuning.

For workload isolation, use custom endpoints to direct analytical queries to dedicated reader instances, preventing long-running SELECT statements from impacting OLTP readers [^336^]:

```bash
# Create a custom endpoint for reporting workloads
aws rds create-db-cluster-endpoint \
    --db-cluster-identifier my-cluster \
    --db-cluster-endpoint-identifier reporting-endpoint \
    --endpoint-type custom \
    --static-members '["reader-instance-1","reader-instance-2"]' \
    --region us-east-1
```

## 11.2 Storage and I/O Planning

### Auto-Scaling: Transparent but Not Free

Aurora storage auto-scales from 10 GB to 128 TB in 10 GB increments. Six copies of data exist across three Availability Zones, but billing counts only the logical data size — not the physical replicas. Standard storage costs $0.10 per GB-month in us-east-1. Backups are free up to 100% of the cluster's provisioned storage size; beyond that, backup storage costs $0.021 per GB-month.

This pricing model eliminates the storage provisioning guesswork required for standard RDS, but it also removes the cost ceiling. A workload with runaway data growth or unoptimized temporary tables can accumulate storage costs without any provisioning step to act as a brake. Monitor actual data size (not just allocated size) with this query:

```sql
-- Accurate per-table sizing using information_schema.files
SELECT file_name,
       ROUND(SUM(total_extents * extent_size) / 1024 / 1024 / 1024, 2) AS size_gb
FROM information_schema.files
WHERE file_name LIKE '%.ibd'
GROUP BY file_name
ORDER BY size_gb DESC
LIMIT 20;
```

### I/O Operations: The Most Common Billing Surprise

I/O is the line item that destroys Aurora cost models. Every read of a 16 KB page from storage and every 4 KB write unit counts as one I/O request at $0.20 per million [^554^]. This includes background I/O from buffer pool evictions, redo log writes, and checksum operations — not just user-query-generated I/O.

A single SQL UPDATE touching 50 pages generates 50 write I/O charges. At 1,000 transactions per second, each touching 50 pages, the cluster generates approximately 4.3 billion I/Os per day — over $860 per day in I/O charges, or more than $25,000 per month in I/O costs alone. This dwarfs compute costs for a `db.r6g.xlarge` instance at roughly $423 per month.

Before migrating any workload to Aurora from standard RDS, establish an I/O baseline. Use Performance Insights to identify queries with the highest physical I/O footprint, and estimate Aurora I/O using:

```
Estimated Monthly I/O Cost = (Peak IOPS_read + Peak IOPS_write) × 2,628,000 seconds/month × $0.20 / 1,000,000
```

The factor of 2,628,000 comes from 730 hours × 3,600 seconds. If this value exceeds 25% of the projected total Aurora bill, evaluate I/O-Optimized configuration.

### I/O-Optimized Pricing: Break-Even at 25%

Aurora I/O-Optimized, launched in mid-2023, eliminates per-I/O charges in exchange for higher storage and compute pricing. Storage increases from $0.10 to $0.225 per GB-month (2.25x), and instance pricing increases by 30%. There is zero per-I/O charge.

| Configuration | Storage Rate | Instance Premium | I/O Rate | Best When |
|:---|:---|:---|:---|:---|
| Standard | $0.10/GB-month | Base | $0.20/million | I/O < 25% of total bill |
| I/O-Optimized | $0.225/GB-month | +30% | $0 (included) | I/O > 25% of total bill [^554^] |

Switching is a cluster-level configuration change with no downtime, but it is reversible only after 30 days. Do not use I/O-Optimized to mask operational problems. If I/O spikes correlate with History List Length growth, fix the purge root cause first — otherwise you pay 30% more for compute and 125% more for storage indefinitely instead of resolving the issue at its source, which may require only terminating a long-running query.

A worked example for a mid-size OLTP workload running on `db.r6g.xlarge` with 500 GB storage:

```
Standard (500M I/Os/month):   $423 compute + $50 storage + $100 I/O  = $573/month
I/O-Optimized (500M I/Os):    $550 compute + $113 storage + $0 I/O   = $663/month
I/O-Optimized break-even:     ~750M I/Os/month (where I/O = $150 on Standard)
```

## 11.3 Scaling Patterns

### Scale-Up vs. Scale-Out Decision Tree

Aurora MySQL is fundamentally a single-master cluster [^539^]. Write throughput scales only vertically (larger instances). Read throughput scales horizontally (more replicas). This asymmetry is the defining constraint of all Aurora scaling decisions.

| Bottleneck Indicator | Scale Up (Larger Instance) | Scale Out (Add Replicas) | Notes |
|:---|:---|:---|:---|
| CPU saturated | Yes | Only for read load | Writer CPU cannot be distributed |
| Memory saturated (low `BufferCacheHitRatio`) | Yes | No | Each replica has its own buffer pool; adding replicas does not increase aggregate cache for the working set |
| Write throughput saturated | Yes — only option | No | Single writer is a hard limit |
| Read throughput saturated | Maybe (if CPU-bound) | Yes — preferred | Custom endpoints distribute load by query type |
| Connection count approaching practical limit | Yes | Partial — connection proxy needed | See Section 11.4.3 for practical limits |
| Replication lag increasing | Check reader sizing first | Yes, with same-size readers | Under-powered readers cause lag, not solve it |
| Purge lag (high HLL) | Yes (more CPU for purge threads) | No — more readers increase risk | Additional readers create more opportunities for old read views |

The critical operational rule: never add read replicas to solve a memory or write-throughput problem. Adding a `db.r6g.large` reader to a cluster whose writer has a 90% `BufferCacheHitRatio` produces another instance with a cold cache and identical memory pressure. Scale the writer up first, then add replicas of the same or larger size.

### Aurora Serverless v2: Not Always Cheaper

Aurora Serverless v2 scales compute capacity in 0.5 ACU increments, where each ACU provides approximately 2 GiB of memory with corresponding CPU and networking. Minimum capacity is 0.5 ACU; maximum is 256 ACU (512 GiB). Scaling from zero capacity (with auto-pause) resumed in approximately 15 seconds as of December 2024, acceptable for development but not for production with latency SLOs [^534^].

Pricing is $0.12 per ACU-hour for Aurora Standard in us-east-1. The cost comparison against provisioned instances is workload-dependent:

| Scenario | ACU Range | Avg ACUs | Monthly Compute Cost | Provisioned Equivalent |
|:---|:---|:---|:---|:---|
| Dev/staging (low usage) | 0.5–4 | 1 | ~$88 | db.t4g.medium (~$52) |
| Small SaaS (business hours) | 0.5–8 | 3 | ~$263 | db.r6g.large (~$190) |
| Mid-size OLTP (variable) | 2–16 | 8 | ~$701 | db.r6g.xlarge (~$423) |
| High-traffic API (sustained) | 8–32 | 24 | ~$2,102 | db.r6g.4xlarge (~$847) |

Serverless v2 is cheaper when the peak-to-average compute ratio exceeds 3x — dev/test environments with idle periods, unpredictable spiky workloads, or applications where capacity planning is genuinely difficult. It is more expensive for steady 24/7 production workloads, especially when Reserved Instances can be applied [^513^].

A hybrid pattern is common in production: a provisioned writer with Reserved Instances for the predictable baseline, plus Serverless v2 readers that auto-scale for variable read traffic. This captures RI discounts on the steady component while avoiding over-provisioning for variable read load.

Cold start considerations matter. When Serverless v2 scales from low capacity, the buffer pool is cold, the query plan cache is empty, and initial queries trigger storage I/O. Set `minimum_capacity` to at least 2 ACU for production workloads to keep the buffer pool warm. Note that Performance Insights requires a minimum of 2 ACU, and Global Database requires 8 ACU.

### Aurora Global Database: Cross-Region at a Multiplier

Aurora Global Database enables a single Aurora cluster to span multiple AWS regions with replication lag typically under one second. It serves two primary use cases: low-latency global reads (serving traffic from the region closest to users) and cross-region disaster recovery (RPO measured in seconds, RTO in minutes).

Cost components beyond standard Aurora pricing include: replicated write I/Os between primary and secondary regions at $0.20 per million; secondary region instances billed at normal regional rates; secondary region storage billed normally; and cross-region data transfer at standard AWS rates. Notably, even under I/O-Optimized configuration, replicated write I/O charges between regions still apply — the zero I/O charges apply only to local reads and writes.

Budget approximately 1.5–2x the cost of the primary cluster for each secondary region. A primary cluster in us-east-1 with two `db.r6i.large` instances plus a secondary in us-west-2 with one reader, 80 GB storage, and 45 million write I/Os per month totals approximately $674 per month [^578^]. Global Database is not recommended for write-heavy workloads where replication costs accumulate without proportional read benefit in the secondary region.

## 11.4 Cost Optimization Strategies

### Reserved Instances vs. On-Demand vs. Database Savings Plans

Three pricing models apply to Aurora compute. The optimal choice depends on commitment flexibility, workload stability, and instance generation.

| Pricing Model | Max Savings | Commitment | Flexibility | Best For |
|:---|:---|:---|:---|:---|
| On-Demand | None | Hourly | Full | Unpredictable, short-term, or evaluation workloads |
| Reserved Instances (1-year, No Upfront) | 40–45% | 1 year, specific family/region/engine | Size-flexible within family; no cross-family | Stable production on known instance family |
| Reserved Instances (3-year, All Upfront) | 63–66% | 3 years, specific family/region/engine | Size-flexible within family | Long-term stable production with capital available |
| Database Savings Plans (1-year) | Up to 35% | 1 year only, No Upfront only | Cross-region, cross-engine, cross-instance, cross-service [^533^] | Mixed database services, possible architecture changes |

Database Savings Plans, launched December 2025, cover Aurora, RDS, DynamoDB, ElastiCache (Valkey), DocumentDB, and Neptune [^533^]. They apply only to Gen 7+ instances (r7g, r7i). If running Gen 6 or older, upgrade first, then purchase commitments. The cross-service flexibility is valuable for organizations running multiple database engines, but the 1-year-only term and lower discount ceiling make 3-year RIs preferable for stable, single-engine workloads.

The break-even for a 1-year No Upfront RI occurs at approximately 55–60% utilization — if the instance runs fewer than ~4,400 hours in a year (out of 8,760), On-Demand is cheaper. For 3-year All Upfront RIs, the break-even is lower (~35% utilization over three years), but the capital is locked with AWS.

### Reader Auto-Scaling for Variable Workloads

Aurora Auto Scaling adds or removes read replicas based on average CPU utilization across all reader instances. The writer CPU is not included in the calculation, and at least one reader must exist for auto-scaling to function [^12^].

Configure with these parameters:

| Parameter | Recommended Value | Rationale |
|:---|:---|:---|
| Target CPU | 40–60% | Lower targets provide more headroom; higher targets reduce cost but increase latency risk |
| Minimum replicas | 1 | Required for HA; set to 2 for production if failover SLA is sub-minute |
| Maximum replicas | Based on peak read requirement × 1.5 | Prevents runaway scaling from query floods |
| Scale-out cooldown | 5 minutes | Prevents flapping; Aurora default |
| Scale-in cooldown | 15 minutes | Longer cooldown prevents premature removal of warmed-up replicas |

Reader auto-scaling adds compute cost only when needed, but each new replica starts with a cold buffer pool. The first queries after scale-out trigger storage I/O, contributing to the I/O bill. For workloads with frequent but short spikes, consider keeping one "warm standby" replica at minimum capacity rather than scaling to zero readers.

For dev/test environments, the most effective cost reduction is stopping instances during off-hours. Aurora clusters can be stopped for up to 7 days; no compute charges accrue during the stop period, though storage and backup charges continue. An instance running 8 hours per day, 5 days per week instead of 24/7 saves approximately 76% of compute costs [^578^]. Automate this with AWS Instance Scheduler or a Lambda function triggered by EventBridge.

### max_connections Reality Check

Aurora MySQL uses a unique formula for calculating `max_connections` that produces theoretical maximums far above practically sustainable limits [^101^]:

```
max_connections = GREATEST(
    {log2(DBInstanceClassMemory/805306368) * 45},
    {log2(DBInstanceClassMemory/8187281408) * 1000}
)
```

For a `db.r6g.large` with 16 GiB RAM, this formula yields approximately 1,000 connections. The practical sustainable limit is 180–200. Each MySQL connection consumes approximately 8 MB of memory, and thread contention degrades throughput well before the connection ceiling is reached [^569^]. Aurora's three-process architecture (`mysqld`, `csdd`, HM) and absence of swap further reduce available memory per connection, increasing OOM risk as connection count grows [^101^].

| Instance Type | RAM | Formula Output (max_connections) | Practical Sustainable Limit |
|:---|:---|:---|:---|
| db.r6g.large | 16 GiB | ~1,000 | 180–200 |
| db.r6g.xlarge | 32 GiB | ~2,000 | 350–400 |
| db.r6g.2xlarge | 64 GiB | ~3,000 | 600–700 |
| db.r6g.4xlarge | 128 GiB | ~4,000 | 1,000–1,200 |

Set CloudWatch alerts at 80% of the practical limit — approximately 150 connections for a `db.r6g.large` — not 80% of the formula output. If peak connections exceed this threshold, implement RDS Proxy or PgBouncer *before* the instance reaches capacity. Treat the formula output as a hard ceiling for crash safety, not a target for normal operation.

```sql
-- Monitor current connection pressure against practical limits
SHOW GLOBAL STATUS LIKE 'Threads_connected';
SHOW GLOBAL STATUS LIKE 'Max_used_connections';

-- Calculate connection memory overhead (approximate)
SELECT
    @@max_connections AS formula_max,
    (SELECT VARIABLE_VALUE FROM performance_schema.global_status
     WHERE VARIABLE_NAME = 'Threads_connected') AS current_connections,
    ROUND(
        (SELECT VARIABLE_VALUE FROM performance_schema.global_status
         WHERE VARIABLE_NAME = 'Threads_connected') * 8 / 1024, 2
    ) AS estimated_conn_memory_gb;
```

---

Planning sets the foundation. Parameters are the knobs you turn day-to-day. Chapter 12 separates the parameters that matter from the parameter delusion — building on the buffer pool insights from Chapter 3, the purge mechanics from Chapter 5, the locking behavior from Chapter 6, and the write path analysis from Chapter 7 to show exactly which tunables have production impact and which are architectural distractions.


## References

[^331^]: [AWS Documentation, "Aurora Custom Endpoints."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.CustomEndpoints.html)
