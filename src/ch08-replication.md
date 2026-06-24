# Chapter 8: Replication, Read Consistency, and the Failover Continuum

Aurora's replicas don't receive binlog events. They share the same storage volume (Chapter 1) and apply redo logs independently to their own buffer pools (Chapter 3). When the writer generates a redo log record for a committed transaction (Chapter 7), that record is durably persisted to the distributed storage layer through a 4-out-of-6 quorum — and then every reader in the cluster reads that same record from storage and applies it to its own local buffer pool.

This architecture is Aurora's most radical departure from standard MySQL. Where conventional replication ships logical binlog events to independent storage volumes, Aurora forwards physical redo log records to readers that share the same distributed storage volume as the writer [^29^]. This yields replica lag in milliseconds rather than seconds, but introduces a two-layer consistency model that every production DBA must internalize. This chapter covers that consistency model, replica lag diagnosis, failover mechanics and endpoint management, and the role of RDS Proxy in eliminating the most common source of application downtime during failovers.

## 8.1 Aurora's Two-Layer Consistency Model

Aurora separates consistency into two distinct layers: the storage layer, which is strongly consistent, and the buffer pool layer, which is eventually consistent. Understanding the boundary between them is essential for building correct applications and for interpreting the `AuroraReplicaLag` metric.

### Storage Layer: Strong Consistency

When a transaction commits on the writer, its redo log records are durably persisted to the Aurora storage layer using a 4-out-of-6 write quorum across three Availability Zones [^334^]. Any instance that reads a page directly from storage receives the latest committed version. The SIGMOD 2017 paper establishes the invariant: "No pages are ever written from the database tier, not for background writes, not for checkpointing, and not for cache eviction" [^29^]. If a reader queries a page that is **not** in its buffer pool, the read goes to shared storage and returns strongly consistent data [^11^]. There is no replication delay at the storage layer.

### Buffer Pool Layer: Eventual Consistency

The buffer pool layer is where eventual consistency appears. Each reader maintains its own independent buffer pool (as established in Chapter 3, the survivable page cache is instance-local). When the writer modifies a page that a reader has cached, that reader must receive and apply the corresponding redo log record to update its cached copy [^29^]. The `AuroraReplicaLag` CloudWatch metric measures precisely this delay — "the lag for the data cache of the Aurora Replica compared to the writer DB instance" [^6^]. It does **not** measure storage-level lag, which is effectively zero.

The practical implication is critical: a reader can see stale data even when `AuroraReplicaLag` reports a value near zero, because the metric is an average across all cached pages. One production report documented exactly this: "The AWS dashboard shows the replica lag to consistently be about 20ms. However, we are seeing old results on the reader more than 90ms after a commit on the master and at least up to 170ms in some cases" [^168^]. The only guaranteed way to avoid stale read-after-write data is to direct such queries to the cluster endpoint (the writer) rather than a reader.

### Read View Propagation

Aurora readers support snapshot isolation for local read-only transactions by receiving continuous transaction state information from the writer [^29^]. A reader will never see uncommitted writer data; it respects the writer's transaction boundaries through MVCC and the shared undo log system. Readers follow two strict redo log application rules: they only apply records with LSN <= the current Volume Durable LSN (VDL), and they apply mini-transactions atomically to maintain consistent B-tree structures [^29^].

The shared undo log architecture creates a critical cross-instance dependency: a long-running transaction on **any** reader blocks the writer's purge process, causing History List Length (HLL) to grow cluster-wide [^15^]. As HLL grows, query performance degrades on all instances, and readers may fall behind in redo application — potentially triggering automatic restart [^157^]. The diagnostic query for this condition belongs in every runbook:

```sql
-- Identify which instance holds the oldest read view, blocking cluster-wide purge
SELECT 
    server_id,
    IF(session_id = 'master_session_id', 'writer', 'reader') AS role,
    replica_lag_in_msec,
    oldest_read_view_trx_id,
    oldest_read_view_lsn
FROM mysql.ro_replica_status
ORDER BY oldest_read_view_trx_id;
```

This query connects directly to Chapter 5's coverage of purge. The History List Length is not a single-instance metric in Aurora — it is a cluster-wide concern because all instances share the same undo log storage.

## 8.2 Replica Lag: Root Causes and Diagnostics

Aurora replica lag measures the time between when a transaction is durably committed to storage and when the reader has applied the corresponding redo log to its buffer pool [^6^]. While typical lag is under 20 milliseconds, production incidents routinely see spikes to seconds or minutes. The root causes fall into four categories, each with distinct diagnostic signatures.

**Under-provisioned readers.** If a reader lacks sufficient CPU or memory to process incoming redo log entries at the writer's production rate, lag accumulates monotonically. AWS recommends that "all DB instances in the Aurora cluster have the same specification" [^6^]. A db.r6g.large reader saving ~$190/month versus a db.r6g.xlarge can trigger lag-driven I/O amplification costing far more than the savings. Diagnostic signature: `CPUUtilization` > 70% sustained with steadily increasing `AuroraReplicaLag`, while writer `WriteThroughput` is normal.

**Heavy queries on readers.** Long-running analytical queries can consume all CPU on a reader, leaving insufficient cycles for the redo log applicator. The key diagnostic is Performance Insights showing a specific query consuming 80%+ of CPU while `AuroraReplicaLag` climbs. Remediation: isolate analytical queries via custom endpoints so OLTP readers remain responsive to redo application.

**Buffer pool pressure.** When the reader's buffer pool is full and must evict pages to make room for new data, redo application slows [^2^]. This creates a compounding feedback loop: more lag means more redo to catch up on, which increases buffer pool pressure, which slows redo application further. Diagnostic signature: `BufferCacheHitRatio` < 90% with non-zero `Innodb_buffer_pool_wait_free`. CloudWatch will show declining `FreeableMemory` correlating with rising `AuroraReplicaLag`.

**High write throughput.** Bulk loads or large batch updates can generate redo faster than readers can apply it. The diagnostic signature is a `WriteThroughput` spike on the writer with `AuroraReplicaLag` increasing across **all** readers simultaneously — this pattern distinguishes writer-driven lag from reader-specific contention. Remediation: break large transactions into smaller batches, or scale up readers temporarily during bulk operations.

| Root Cause | Diagnostic Signature | Key Metrics / Queries | Remediation |
|:---|:---|:---|:---|
| Under-provisioned reader | `CPUUtilization` > 70% sustained; lag increases monotonically; writer throughput normal | CloudWatch `CPUUtilization` + `AuroraReplicaLag` trend; `REPLICA_HOST_STATUS` CPU column [^6^] | Scale reader to match writer instance class; AWS recommends identical specs across cluster |
| Heavy queries consuming CPU | Specific query dominates CPU in Performance Insights; lag correlates with query runtime | Performance Insights top SQL by CPU; `SHOW PROCESSLIST` on reader | Isolate analytical queries via custom endpoints; add covering indexes; kill or reschedule long queries |
| Buffer pool pressure | `BufferCacheHitRatio` < 90%; `Innodb_buffer_pool_wait_free` > 0; `FreeableMemory` declining | `SHOW STATUS LIKE 'Innodb_buffer_pool%'`; CloudWatch `BufferCacheHitRatio` [^2^] | Scale up to larger instance class; reduce `innodb_buffer_pool_size` if OOM risk exists; optimize queries to touch fewer pages |
| High writer throughput | `WriteThroughput` spike on writer; lag increases on **all** readers simultaneously | CloudWatch `WriteThroughput` + `AuroraReplicaLagMaximum`; correlate writer spikes with lag onset | Break bulk operations into smaller batches; scale up readers temporarily; schedule bulk loads during low-traffic periods |
| Blocked purge / HLL growth | `RollbackSegmentHistoryListLength` > 10,000; oldest read view on a reader; lag correlates with HLL | `SELECT ... FROM mysql.ro_replica_status ORDER BY oldest_read_view_trx_id` [^15^]; `SELECT COUNT FROM information_schema.innodb_metrics WHERE name = 'trx_rseg_history_len'` | Kill long-running transactions on any instance; tune `innodb_purge_threads`; monitor HLL as P1 metric |

The table above maps each root cause to its observable signature, the specific metrics or SQL queries that confirm it, and the corresponding remediation. In production, multiple causes often compound: an under-provisioned reader running heavy queries during a bulk load will exhibit all three local symptoms simultaneously. The first step in any lag incident should be the `mysql.ro_replica_status` query to check for blocked purge, because HLL-driven lag is the most dangerous — it affects the entire cluster, not just one reader.

CloudWatch monitoring for replica lag should use tiered thresholds: alarm at 1,000ms (warning, indicating a reader is falling behind), and page at 5,000ms (critical, failover time is being extended). The `AuroraReplicaLagMaximum` metric is particularly useful because it reveals the worst lag across all readers in a single data point, catching individual readers that may be hidden by averages. For Global Database deployments, monitor `AuroraGlobalDBReplicationLag` as the live cross-region RPO indicator [^370^].

```bash
# CloudWatch: replica lag trend across all readers over the last hour
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name AuroraReplicaLag \
  --dimensions Name=DBClusterIdentifier,Value=prod-cluster Name=Role,Value=READER \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
  --period 60 \
  --statistics Average Maximum
```

## 8.3 Failover Mechanics and High Availability

Aurora's failover mechanism is fundamentally different from standard MySQL because readers share the writer's storage volume. There is no data to copy, no binlog position to align, and no relay log to replay. The promoted reader simply starts accepting write connections against the same storage volume it was already reading from.

### Failover Process

The failover sequence has four phases. **Detection**: Aurora health monitors check the writer through heartbeats and storage-layer connectivity checks; when the writer fails to respond, Aurora declares it unhealthy [^134^]. **Promotion**: Aurora selects the most up-to-date reader from the highest available promotion tier (0 to 15, where 0 is highest priority) [^334^]. If no reader exists, Aurora must create a new instance — extending failover from seconds to up to 15 minutes [^211^]. **DNS Update**: The cluster endpoint CNAME is flipped to the promoted reader's IP; clients that re-resolve route to the new writer [^211^]. **Epoch Fencing**: Aurora increments a volume epoch in its metadata service and records it in a write quorum. Storage nodes reject requests at stale epochs, boxing out the old writer and preventing split-brain [^28^].

In Aurora MySQL 2.10+, in-region readers do **not** reboot on writer failover [^334^]. Only the old writer and promoted reader restart; other readers continue serving traffic uninterrupted.

### RTO and RPO Guarantees

Aurora in-region failover typically completes within 30 seconds: 0-5 seconds for 30% of failovers, 5-10 seconds for 40%, 10-20 seconds for 25%, and 20-30 seconds for 5% [^334^]. An independent benchmark measured 7 seconds of application downtime [^211^]. The Recovery Point Objective (RPO) is zero for in-region failures because all committed writes are already durably persisted across six copies in three AZs [^333^].

Cross-region failover via Aurora Global Database has different characteristics: typical RPO is 1 second and RTO is 1 minute [^134^]. However, actual RPO during an unplanned outage depends on `AuroraGlobalDBReplicationLag` at the moment of failure. Because Global Database replication is asynchronous, any data not yet replicated to the secondary region is lost.

The most common cause of slow failover is the easiest to prevent: running a single-instance cluster with no reader. Without a reader, Aurora must recreate the primary — taking minutes instead of seconds. Running at least one reader in a different AZ is the single highest-leverage HA decision in Aurora [^329^].

### Endpoint Types

Aurora provides four endpoint types, each with distinct behavior during failover. Using the wrong endpoint in application connection strings is one of the most common operational mistakes.

| Endpoint Type | DNS Pattern | Behavior During Failover | Use Case | Caveat |
|:---|:---|:---|:---|:---|
| Cluster (writer) | `*.cluster-*.*.rds.amazonaws.com` | Automatically updates to new writer; CNAME flip within seconds | All write operations; any query requiring read-after-write consistency | Applications must re-resolve DNS; JVMs may cache IP past TTL [^340^] |
| Reader | `*.cluster-ro-*.*.rds.amazonaws.com` | Round-robin across healthy readers; excludes failed/unavailable readers | Read-only SELECT queries; read scaling | Uneven distribution if client caches DNS; may briefly route to promoted writer during failover [^329^] |
| Custom | User-defined | Static members (explicit include list) or excluded members (dynamic include-all-except) | Workload isolation (analytics vs OLTP); routing by instance size; dedicated failover tiers | Must be managed; new instances only join dynamic endpoints automatically [^331^] |
| Instance | `*.*.rds.amazonaws.com` | Points to one specific DB instance; does **not** follow failover | Diagnostic queries; tuning; debugging specific instance issues | Never use in application code; will break during failover [^329^] |

The cluster endpoint always resolves to the current writer; the reader endpoint load-balances read-only connections across available Aurora Replicas using DNS round-robin [^329^]. Custom endpoints enable workload isolation: define an "analytics" endpoint targeting only your largest readers, preventing long-running reporting queries from consuming CPU on a failover candidate [^329^].

Instance endpoints exist for operational necessity only. They point to a single DB instance and do not change during failover. Applications that hardcode instance endpoints will fail during failover — either with write failures (if the instance is a reader) or connection drops (if the instance restarts).

Failover testing should be performed regularly using the AWS CLI:

```bash
# Force failover to a specific reader (recommended for testing)
aws rds failover-db-cluster \
  --db-cluster-identifier prod-cluster \
  --target-db-instance-identifier reader-az-b

# Or failover to the highest-priority reader (default behavior)
aws rds failover-db-cluster --db-cluster-identifier prod-cluster
```

Always initiate failover tests during low-traffic periods, monitor application error rates and recovery time, and verify that the cluster endpoint resolves to the new writer afterward [^373^]. Subscribe to RDS event notifications via SNS for real-time awareness:

```bash
# Create event subscription for failover events
aws rds create-event-subscription \
  --subscription-name aurora-failover-alerts \
  --sns-topic-arn arn:aws:sns:us-east-1:123456789012:dba-alerts \
  --source-type db-cluster \
  --event-categories 'failure,failover,recovery'
```

## 8.4 RDS Proxy and Connection Management

Even a 10-second database failover can translate to minutes of application downtime if connections are not handled correctly. RDS Proxy addresses this by maintaining a pool of warm database connections and transparently rerouting them during failover.

### How RDS Proxy Reduces Failover Impact

RDS Proxy sits between application servers and the Aurora cluster, maintaining a pool of warm database connections reused across application connections. During failover, the proxy monitors cluster topology and routes connections to the new writer without waiting for DNS propagation [^345^]. This eliminates two major downtime sources: DNS caching latency (where JVMs may cache the old IP for minutes) and connection storms (where hundreds of servers simultaneously connect to a freshly promoted instance).

AWS reports failover times reduced by up to 66%, with client-recovery improvements of up to 79% for Aurora MySQL [^329^]. Proxy endpoints do not change during failovers — the same IP continues accepting connections while the proxy internally redirects [^345^]. When demand exceeds limits, the proxy queues new connections, adding latency rather than letting the database fall over [^329^].

### Session Pinning

RDS Proxy for MySQL pins sessions to specific database connections when certain SQL operations occur that prevent safe connection reuse. Pinned sessions dramatically reduce connection pool efficiency and can cause the proxy to exhaust its connection limit. Operations that trigger pinning include `SET` statements (which modify session state), prepared statements, transactions with `autocommit = 0`, user-defined variables, and any statement with text size greater than 16 KB [^368^] [^364^].

One production team observed `DatabaseConnections` exceeding 4,000 despite a configured proxy limit of 1,600, traced to an API generating query text sizes exceeding 100 KB, which caused every session to be pinned [^364^]. The diagnostic approach is to monitor the `ClientConnections` (application connections to proxy) versus `DatabaseConnections` (proxy connections to database) metrics in CloudWatch. A large and growing gap between them indicates excessive pinning.

To minimize pinning: move `SET` statements into the proxy's initialization query configuration rather than issuing them per-session; keep individual SQL statements under 16 KB; and use simple autocommit transactions where possible. When a workload cannot avoid pinning (for example, an application that relies heavily on prepared statements), consider splitting the workload across multiple proxies.

### Best Practices

The following practices minimize failover downtime and ongoing operational risk:

**DNS TTL**: Set DNS time-to-live to less than 30 seconds for all Aurora endpoints. The JVM caches hostname-to-IP mappings far longer than the record's TTL by default — set `networkaddress.cache.ttl` to no more than 60 seconds in `$JAVA_HOME/jre/lib/security/java.security` [^340^].

**Connection Pool Configuration**: Enable `test-on-borrow` with a lightweight validation query so stale connections are discarded. Set a bounded maximum connection lifetime (e.g., 10 minutes) so the pool periodically recycles connections and re-resolves DNS. Keep connection-acquire timeout short to fail fast and retry [^329^].

**AWS JDBC Driver**: For Java applications, the AWS Advanced JDBC Driver provides topology-aware failover that can reduce recovery from minutes to seconds [^359^]. The `failover` plugin detects failover events and reroutes; `efm2` provides background socket probing. When using RDS Proxy, drop the `failover`, `efm2`, and `readWriteSplitting` plugins because the proxy handles topology automatically; use the `srw` plugin with two explicit proxy endpoints instead [^359^].

**Retry Logic**: Catch connection errors, reconnect through the same endpoint, and use bounded retries with exponential backoff and jitter to prevent a thundering herd. Design writes to be idempotent — in-flight transactions fail during failover. Keep transactions short; long-running transactions lengthen the replay set and extend failover time [^329^].

**Endpoint Discipline**: Always use the cluster endpoint for writes and the reader endpoint for reads. Never use instance endpoints in application code. The combination of cluster endpoint + reader endpoint + RDS Proxy provides the most resilient connection architecture for production Aurora deployments [^340^].

```bash
# Verify endpoint resolution after failover (should point to new writer)
dig +short prod-cluster.cluster-xxx.us-east-1.rds.amazonaws.com

# Verify reader endpoint excludes the new writer
dig +short prod-cluster.cluster-ro-xxx.us-east-1.rds.amazonaws.com
```

The failover continuum in Aurora spans from the storage layer's synchronous quorum (guaranteeing zero RPO) through the buffer pool's asynchronous redo application (creating the eventual consistency visible to applications) to the DNS-based endpoint propagation (determining how quickly applications find the new writer). Understanding each layer's timing, failure mode, and diagnostic signature is what separates a DBA who can survive a 3 AM page from one who can prevent it.

---

We've traced writes from buffer pool to storage to replicas. But what about the queries that READ this data? Chapter 9 examines how Aurora executes queries, optimizes plans, and surfaces performance problems — bringing together everything we have learned about storage, transactions, and the pipeline to diagnose and fix the queries that drive your application.

[^2^]: AWS Documentation, "Amazon Aurora Storage Overview."
[^6^]: AWS Documentation, "Monitoring Aurora Replication with Amazon CloudWatch."
[^11^]: AWS Documentation, "Aurora Read Consistency Model."
[^15^]: AWS Documentation, "Doublewrite Buffer Elimination in Aurora."
[^28^]: AWS re:Invent 2018, Aurora Deep Dive Session.
[^29^]: AWS Documentation, "VDL Truncation and Epoch Fencing."
[^59^]: AWS Documentation, "Aurora MySQL Wait Events."
[^134^]: AWS Documentation, "Aurora Global Database RTO/RPO."
[^157^]: AWS Documentation, "Reader Restart and HLL Growth."
[^168^]: AWS re:Post, "Replica Lag Metric vs Actual Staleness."
[^211^]: AWS Documentation, "Aurora Failover Benchmarks."
[^329^]: AWS Documentation, "Aurora Best Practices — Endpoints and Failover."
[^331^]: AWS Documentation, "Aurora Custom Endpoints."
[^333^]: AWS Documentation, "Aurora RPO Guarantees."
[^334^]: AWS Documentation, "Aurora Failover Timing Statistics."
[^340^]: AWS Documentation, "DNS and JVM Caching with Aurora."
[^345^]: AWS Documentation, "RDS Proxy for Aurora."
[^359^]: AWS Documentation, "AWS Advanced JDBC Driver for Aurora."
[^364^]: AWS re:Post, "RDS Proxy Session Pinning Diagnosis."
[^368^]: AWS Documentation, "RDS Proxy Session Pinning Causes."
[^370^]: AWS Documentation, "Aurora Global DB Replication Lag."
[^373^]: AWS Documentation, "Failover Testing Best Practices."
[^481^]: AWS Documentation, "CloudWatch Database Insights — Standard vs Advanced."
[^482^]: AWS Documentation, "Performance Insights End of Life Notice."
[^491^]: AWS Documentation, "Migrating from Performance Insights to Database Insights."
[^497^]: Aurora MySQL 3.x Release Notes and Optimizer Documentation.
