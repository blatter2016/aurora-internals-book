# Chapter 10: Monitoring, Alerting, and Observability

Most Aurora monitoring is backwards. Open the default CloudWatch dashboard for any Aurora MySQL cluster and you will see `CPUUtilization`, `DatabaseConnections`, and `FreeableMemory` in large line charts at the top. These are the metrics AWS surfaces most prominently, and they are the ones most teams alert on first. They are also, operationally, the *least* useful indicators for detecting the failure modes that actually matter in Aurora.

This is the **monitoring inversion**: the metrics that are easiest to watch are symptoms, not causes. A CPU spike tells you something is consuming compute, but not whether the root cause is a missing index, a buffer pool eviction storm (Chapter 3), or a purge backlog (Chapter 5). A connection count approaching `max_connections` tells you clients are piling up, but not *why* — whether the application leaked connections or queries slowed so much that the pool drained. These metrics are emotionally satisfying to watch but rarely guide you to the correct remediation step.

The metrics that predict Aurora outages are buried deeper. `RollbackSegmentHistoryListLength` — the undo log purge backlog that Chapter 5 identified as the single most important health indicator — exists as a CloudWatch metric that many DBAs do not know exists [^59^]. `BufferCacheHitRatio` determines whether your working set fits in memory, directly driving `VolumeReadIOPS`, latency, and cost [^1^]. `AuroraReplicaLag` is widely monitored but widely misunderstood: it measures page cache update propagation, not transaction apply lag, and a value under 20 ms does not guarantee healthy replicas (Chapter 8) [^351^].

This chapter inverts the inversion. It establishes a two-tier metric hierarchy — causes versus symptoms — provides nine essential CloudWatch alarms with production-tested thresholds, covers Enhanced Monitoring and CloudWatch Logs Insights queries, and maps third-party tool integration for teams building unified observability platforms.

## 10.1 Tier 1 Metrics: The Causes

**Tier 1 metrics** are leading indicators. They reveal root conditions before user-facing degradation occurs. These are the metrics that should occupy the top row of every production dashboard.

### RollbackSegmentHistoryListLength (HLL)

HLL represents undo log entries awaiting purge. In Aurora this is cluster-wide because all instances share a single undo log space [^317^]. A long-running transaction on any reader — even a single `SELECT` with `REPEATABLE READ` isolation — blocks the writer's purge thread and causes HLL to grow without bound [^331^]. Above 10,000, query performance becomes inconsistent. Above 100,000, the cluster enters a degradation spiral: queries slow, connections accumulate, CPU rises, and readers may restart [^59^].

The HLL threshold progression follows directly from Chapter 5's analysis. Below 1,000 is healthy. Between 1,000 and 10,000, purge is falling behind — investigate immediately. Between 10,000 and 100,000, queries traversing undo chains for MVCC version reconstruction degrade measurably. Above 100,000, the cluster approaches an operational crisis [^460^]. A rising trend — even below 10,000 — indicates purge threads falling behind and warrants intervention before the threshold crosses a danger line.

### BufferCacheHitRatio

`BufferCacheHitRatio` measures the percentage of read requests served from the InnoDB buffer pool. Aurora exposes this as a native CloudWatch metric — an advantage over standard RDS MySQL where it must be calculated manually [^122^]. Below 95%, the working set no longer fits in memory. Below 90%, queries read from the distributed storage layer, spiking `VolumeReadIOPS` and increasing latency [^1^].

This metric connects directly to Chapter 3's buffer pool analysis. The 75% default for `innodb_buffer_pool_size` works only when the working set fits within that allocation. When `BufferCacheHitRatio` drops below 95%, the response is not parameter tuning — it is scaling up to a larger instance class. No parameter change can compensate for a working set that exceeds available memory.

### AuroraReplicaLag

`AuroraReplicaLag` requires the careful interpretation that Chapter 8 established. Because all Aurora replicas read from the same shared storage volume, this metric tracks page cache update propagation time — not transaction apply lag [^351^]. Aurora replica lag is typically under 20 ms while traditional MySQL replication lag is measured in seconds. However, a *sustained* increase on a specific reader indicates buffer pool pressure: the reader is evicting pages to make room and falling behind in applying redo log records [^29^]. A lag spike during cache warm-up is benign; during steady-state, it signals memory pressure.

## 10.2 Tier 2 Metrics: The Symptoms

**Tier 2 metrics** are lagging indicators. They move in response to Tier 1 changes, often after performance has degraded. These metrics belong on dashboards — they are useful for capacity planning and as backstops — but they should not be primary operational triggers.

`CPUUtilization` rises when queries slow down, when purge backlog forces extra version traversal, or when buffer cache misses trigger storage waits. Sustained CPU above 80% with moderate I/O points to CPU-heavy query plans: complex sorts, JSON operations, or hash aggregations [^497^]. But high CPU is a *result*, not a cause. The remediation depends entirely on which Tier 1 metric triggered the CPU rise.

`DatabaseConnections` approaching `max_connections` is a *result* of queries taking longer, not a root cause. Aurora's default `max_connections` formula — `GREATEST(log(DBInstanceClassMemory/805306368)*45, log(DBInstanceClassMemory/8187281408)*1000)` with log base 2 [^101^] — produces ~1,000 connections for a db.r6g.large. The practical sustainable limit is ~180–200. Alerting on raw count without understanding the practical ceiling is a common pitfall [^503^].

`VolumeQueueLength` indicates I/O pressure at the storage layer. In a healthy cluster this hovers near 1; sustained values above 10 indicate a severe bottleneck [^1^]. The cause is typically buffer cache thrashing (Chapter 3) or a write burst exceeding storage capacity.

| Metric | Tier | Type | Why It Matters | Alert Threshold |
|---|---|---|---|---|
| `RollbackSegmentHistoryListLength` | 1 (Cause) | Leading | Cluster-wide purge health; reader transactions block writer [^59^] | > 10,000 warning; > 100,000 critical |
| `BufferCacheHitRatio` | 1 (Cause) | Leading | Working set fit in memory; drives I/O and latency [^1^] | < 95% warning; < 90% critical |
| `AuroraReplicaLag` | 1 (Cause) | Leading | Reader buffer pool pressure; cache-vs-transaction distinction [^351^] | > 1,000 ms warning; > 10,000 ms critical |
| `CPUUtilization` | 2 (Symptom) | Lagging | Consumption, not cause; requires wait event analysis [^497^] | > 80% for 5 min |
| `DatabaseConnections` | 2 (Symptom) | Lagging | Result of slow queries, not root cause [^503^] | > 80% of practical limit |
| `VolumeQueueLength` | 2 (Symptom) | Lagging | I/O queuing at storage; usually buffer cache miss [^1^] | > 10 sustained |

This table should guide dashboard construction. Tier 1 metrics belong at the top with prominent trend lines and growth-rate alarms. CPU and connection alerts serve as backstops: they catch issues that slip past Tier 1 monitoring, but should never be the only line of defense.

## 10.3 Nine Essential CloudWatch Alarms

Aurora MySQL publishes metrics to CloudWatch in the `AWS/RDS` namespace at 1-minute granularity by default, retained for 15 days (extendable via Enhanced Monitoring and Database Insights Advanced mode) [^122^]. The selection of metrics to alert on — and their thresholds — separates reactive firefighting from proactive operations.

Every production Aurora cluster should have these nine alarms. Thresholds assume an OLTP workload on db.r6g.xlarge or larger; adjust for smaller instances.

| # | Alarm Name | Metric | Threshold | Severity | Evaluation | Rationale |
|---|-----------|--------|-----------|----------|------------|-----------|
| 1 | `aurora-hll-critical` | `RollbackSegmentHistoryListLength` | > 10,000 | P2 (High) | 5 min | Purge falling behind; long-running transaction or reader blocking writer [^59^] |
| 2 | `aurora-buffer-cache-low` | `BufferCacheHitRatio` | < 95% | P2 (High) | 5 min | Working set exceeds memory; upgrade instance or optimize queries [^1^] |
| 3 | `aurora-replica-lag-high` | `AuroraReplicaLagMaximum` | > 1,000 ms | P2 (High) | 3 min | Worst replica falling behind; memory pressure or network issue [^351^] |
| 4 | `aurora-cpu-high` | `CPUUtilization` | > 80% | P3 (Medium) | 5 min | CPU saturation; investigate queries via Database Insights [^497^] |
| 5 | `aurora-memory-low` | `FreeableMemory` | < 20% of total RAM | P2 (High) | 5 min | Memory pressure causes cache evictions, then I/O, then latency [^233^] |
| 6 | `aurora-connections-high` | `DatabaseConnections` | > 80% of practical limit | P2 (High) | 3 min | Connection pool exhaustion; check for leaked connections or slow queries [^503^] |
| 7 | `aurora-disk-queue-high` | `VolumeQueueLength` | > 10 | P2 (High) | 5 min | I/O bottleneck at storage layer; usually cache miss amplification [^1^] |
| 8 | `aurora-login-failures` | `LoginFailures` | > 0 | P3 (Medium) | 1 min | Security or credential issue; immediate investigation [^122^] |
| 9 | `aurora-volume-read-iops-spike` | `VolumeReadIOPS` | > 3x baseline | P3 (Medium) | 5 min | Cache miss spike or full table scan; correlate with BufferCacheHitRatio [^347^] |

Alarms 1–3 are Tier 1 metrics detecting root causes before degradation. Alarms 4–7 are Tier 2, catching symptoms after performance degrades. Alarm 8 is a security control. Alarm 9 catches cache miss patterns that a gradual `BufferCacheHitRatio` decline might miss.

Create alarms via CLI:

```bash
# High HLL alarm (Alarm #1)
aws cloudwatch put-metric-alarm \
    --alarm-name "prod-aurora-hll-critical" \
    --alarm-description "HLL > 10000 indicates purge falling behind" \
    --metric-name RollbackSegmentHistoryListLength \
    --namespace AWS/RDS \
    --statistic Average --period 300 --evaluation-periods 1 \
    --threshold 10000 --comparison-operator GreaterThanThreshold \
    --dimensions Name=DBInstanceIdentifier,Value=writer-instance-id \
    --alarm-actions arn:aws:sns:us-east-1:123456789012:rds-critical-alerts \
    --ok-actions arn:aws:sns:us-east-1:123456789012:rds-critical-alerts

# Buffer cache alarm (Alarm #2)
aws cloudwatch put-metric-alarm \
    --alarm-name "prod-aurora-buffer-cache-low" \
    --metric-name BufferCacheHitRatio \
    --namespace AWS/RDS \
    --statistic Average --period 300 --evaluation-periods 1 \
    --threshold 95 --comparison-operator LessThanThreshold \
    --dimensions Name=DBInstanceIdentifier,Value=writer-instance-id \
    --alarm-actions arn:aws:sns:us-east-1:123456789012:rds-critical-alerts \
    --ok-actions arn:aws:sns:us-east-1:123456789012:rds-critical-alerts
```

The `--ok-actions` parameter ensures recovery notifications when metrics return to normal. Without it, on-call engineers must manually verify recovery, adding noise to incident response [^487^].

### VolumeReadIOPS: The Cache Miss Signal

`VolumeReadIOPS` measures billed read I/O from the cluster storage volume. In a healthy cluster this stays under 100 during normal operations [^347^]. Spikes indicate either buffer pool cache misses (working set exceeds memory, Chapter 3) or full table scans (queries reading entire tables without indexes).

When `VolumeReadIOPS` spikes, check `BufferCacheHitRatio`. If the hit ratio dropped simultaneously, the working set exceeds buffer pool capacity — scale up to a larger instance [^122^]. If the hit ratio remained stable, a specific query is performing a full table scan; find it via Database Insights Top SQL or the slow query log.

### VolumeQueueLength: The I/O Bottleneck Indicator

`VolumeQueueLength` measures pending I/O requests at the storage layer. Healthy databases maintain this near 1; sustained values above 10 indicate requests queuing faster than storage can service them [^1^].

Unlike `VolumeReadIOPS`, which spikes harmlessly during batch operations, sustained high `VolumeQueueLength` is always actionable. The most common cause is buffer pool thrashing: when the working set exceeds memory, every eviction triggers a storage read and the queue backs up. The immediate response is reducing load (kill long-running queries, throttle writes) while planning an instance upgrade.

## 10.4 Enhanced Monitoring and Logs

CloudWatch metrics at 1-minute granularity are sufficient for trend analysis and alerting, but miss transient spikes that resolve within seconds. Enhanced Monitoring provides the sub-minute granularity needed for diagnosing brief but impactful events.

### Enhanced Monitoring: OS-Level Metrics

Enhanced Monitoring collects OS metrics directly from an agent on the DB instance, bypassing the hypervisor layer that standard CloudWatch uses [^233^]. This produces more accurate CPU and memory measurements, especially on smaller instances.

Key characteristics:

- **Granularity**: configurable 1–60 seconds; production clusters should use 1–5 seconds.
- **Data destination**: JSON payloads in CloudWatch Logs under `RDSOSMetrics`.
- **Retention**: 30 days by default.
- **No reboot required**: enabling or changing granularity does not restart the instance [^462^].
- **Minimal overhead**: the agent runs on the host OS, not inside the database engine [^462^].

Enhanced Monitoring captures CPU breakdown (user, system, iowait, steal), memory (free, cached, buffers, active, inactive), disk I/O per device, network packets, swap usage, and per-process resource consumption [^459^]. The process list appears in Database Insights Advanced mode under the OS Process tab, distinguishing mysqld usage from background Aurora processes (csdd, HM) [^456^].

Enable via CLI:

```bash
aws rds modify-db-instance \
    --db-instance-identifier my-aurora-instance \
    --monitoring-interval 5 \
    --monitoring-role-arn arn:aws:iam::123456789012:role/rds-monitoring-role \
    --apply-immediately
```

The IAM role requires the `AmazonRDSEnhancedMonitoringRole` policy. Each instance creates a log stream named after its `resourceId` within `RDSOSMetrics` [^462^].

### Log Types and Their Use Cases

Aurora MySQL supports four log types. All can be published to CloudWatch Logs for centralized querying [^371^].

| Log Type | Default | CloudWatch Log Group | Primary Use Case | When to Enable |
|----------|---------|---------------------|------------------|----------------|
| Error log | Enabled | `/aws/rds/instance/<id>/error` | Crash diagnostics, startup/shutdown, failed connections | Always; no production impact |
| Slow query log | Disabled | `/aws/rds/instance/<id>/slowquery` | Query performance; identifies queries > `long_query_time` | Permanently in production |
| General log | Disabled | `/aws/rds/instance/<id>/general` | Complete SQL audit for debugging | Short windows only; high I/O overhead [^371^] |
| Audit log | Disabled | `/aws/rds/instance/<id>/audit` | Compliance and security via MariaDB Audit Plugin | For regulatory requirements [^474^] |

Enable the slow query log via parameter group:

```ini
slow_query_log = 1
long_query_time = 1
log_output = FILE
log_queries_not_using_indexes = 1
log_slow_admin_statements = 1
```

The slow query log is the most operationally valuable log for performance tuning. Each entry includes `Query_time` (execution time), `Lock_time` (lock acquisition time), `Rows_sent`, and `Rows_examined` [^8^]. A large gap between `Rows_examined` and `Rows_sent` signals a missing index or overly broad `WHERE` clause.

The audit log uses the MariaDB Audit Plugin, enabled via stored procedures:

```sql
CALL mysql.rds_set_configuration('server_audit_logging', 'ON');
CALL mysql.rds_set_configuration('server_audit_events', 'CONNECT,QUERY,TABLE');
CALL mysql.rds_set_configuration('server_audit_excl_users', 'rdsadmin');
```

Supported events: `CONNECT`, `QUERY` (all queries in plain text), `QUERY_DCL`, `QUERY_DDL`, `QUERY_DML`, and `TABLE` [^500^].

### CloudWatch Logs Insights: Sample Queries

Set log retention to 30 days for production; export to S3 for long-term archival [^371^].

Find slowest queries in the last hour:

```sql
fields @timestamp, @message
| filter @message like /Query_time/
| parse @message /Query_time: * Lock_time: * Rows_sent: * Rows_examined: */
| stats max(Query_time) as max_time, avg(Query_time) as avg_time by query
| sort max_time desc
| limit 20
```

Detect connection failures from the error log:

```sql
fields @timestamp, @message
| filter @message like /Access denied/
| stats count() as failed_attempts by bin(5m)
```

Identify excessive row scans (missing indexes):

```sql
fields @timestamp, @message
| filter @message like /Rows_examined/ and @message like /Rows_sent/
| parse @message /Rows_examined: * Rows_sent: */
| filter Rows_examined > Rows_sent * 1000
| stats count() as scan_count by bin(1h)
```

Detect mysqld restarts:

```sql
fields @timestamp, @message
| filter @message like /starting as process/
| stats count() as restarts by bin(1h)
```

## 10.5 Third-Party Monitoring Integration

CloudWatch and Database Insights provide the foundation, but third-party tools offer deeper analysis, longer retention, and unified observability. Three options stand out for production Aurora deployments.

### Datadog Database Monitoring

Datadog provides the deepest Aurora MySQL integration among commercial platforms [^502^]. Its Database Monitoring captures query-level metrics, execution plans, wait event analysis, InnoDB telemetry, and connection data. It autodiscovers Aurora endpoints and correlates database metrics with AWS infrastructure data.

Critical configuration: the Datadog Agent must connect to each **instance endpoint**, not the cluster endpoint. Connecting through the cluster endpoint collects data from a random replica each time, producing inconsistent metrics [^502^].

```yaml
# mysql.d/conf.yaml
init_config:
instances:
  - dbm: true
    host: '<AWS_INSTANCE_ENDPOINT>'  # NOT cluster endpoint
    port: 3306
    username: datadog
    password: 'ENC[datadog_user_password]'
    aws:
      instance_endpoint: '<AWS_INSTANCE_ENDPOINT>'
```

Datadog's wait event analysis breaks down time by event type (I/O, CPU, locks, mutexes), revealing whether high `CPUUtilization` is actual CPU work or CPU waiting on I/O. Query-level metrics include rows examined, rows returned, execution count, and latency percentiles.

### Prometheus and Grafana

The Prometheus `mysqld_exporter` collects MySQL metrics for scraping at port 9104, requiring a monitoring user with `SELECT` privileges [^490^].

```bash
docker run -d -p 9104:9104 \
    -e DATA_SOURCE_NAME="monitoring:password@(aurora-instance-endpoint:3306)/" \
    prom/mysqld-exporter
```

Grafana Cloud offers Database Observability via Grafana Alloy, which connects to Aurora instance endpoints and streams metrics to Grafana Cloud. Requirements: Aurora MySQL 8.0+, `performance_schema = ON`, `performance_schema_max_digest_length = 4096` [^495^]. The open-source stack requires more setup but provides full data ownership and avoids per-vCPU pricing.

### Percona Monitoring and Management (PMM)

PMM is a free, open-source tool supporting Aurora MySQL via AWS discovery [^510^]. It provides query analytics (QAN), alerting, and dashboards. OS-level metrics require Enhanced Monitoring and IAM permissions (`cloudwatch:GetMetricData`, `rds:DescribeDBInstances`). A common issue is missing IAM permissions causing blank OS metric graphs [^510^]. PMM's Query Analytics parses slow query logs and Performance Schema to identify top queries by time, frequency, and resource consumption.

## 10.6 Production Dashboard Specification

This three-tier dashboard implements the monitoring inversion: Tier 1 metrics (causes) receive most prominence; Tier 2 (symptoms) are available but de-emphasized.

**Dashboard: `aurora-mysql-production-overview`**

**Row 1 — Tier 1: Cluster Health (causes)**

| Widget | Metric | Visualization | Notes |
|--------|--------|--------------|-------|
| HLL Trend | `RollbackSegmentHistoryListLength` | Line chart, 1-hour window | Annotate with 10K and 100K threshold lines |
| Buffer Cache Efficiency | `BufferCacheHitRatio` | Line chart, 1-hour window | Target band 95–100% |
| Replica Lag Heatmap | `AuroraReplicaLag` per replica | Heatmap, 15-min window | Max lag determines failover RPO |
| Volume Read IOPS | `VolumeReadIOPS` (writer) | Line chart, 1-hour window | Spikes correlate with cache misses |

**Row 2 — Tier 2: Instance Resources (symptoms)**

| Widget | Metric | Visualization | Notes |
|--------|--------|--------------|-------|
| CPU (writer + readers) | `CPUUtilization` | Stacked area, 1-hour window | Overlay with vCPU count line |
| Freeable Memory | `FreeableMemory` | Line chart, 1-hour window | Annotate with 20% of total RAM threshold |
| Connections | `DatabaseConnections` | Line chart, 1-hour window | Overlay with practical limit |
| Volume Queue Depth | `VolumeQueueLength` | Line chart, 1-hour window | Threshold line at 10 |

**Row 3 — Query and Event Analysis**

| Widget | Source | Visualization | Notes |
|--------|--------|--------------|-------|
| AAS Load Chart | Database Insights | Bar chart over time | AAS exceeding vCPU count = queuing [^457^] |
| Top SQL by Time | Database Insights / slow log | Table, top 10 | Include wait event breakdown |
| Event Timeline | RDS Event Notifications | Annotation overlay | Failover, maintenance, parameter changes [^472^] |
| Slow Query Rate | CloudWatch Logs Insights | Line chart, count per 5 min | Derived from slow query log parsing |

Placing HLL, buffer cache, and replica lag at the top trains operators to check leading indicators first. The AAS load chart provides query-level detail for diagnosing root causes when Tier 1 metrics move outside healthy bands. The event timeline ensures operational events (failover, maintenance) are visible alongside metric changes, preventing false alarms from scheduled activities [^473^].

**Performance Insights to Database Insights Transition:** AWS Performance Insights reaches end of life on June 30, 2026 [^482^]. Clusters will default to Database Insights Standard mode, which lacks execution plans and on-demand analysis. Only Advanced mode (~$9/vCPU/month) provides these capabilities with 15-month retention and OS process telemetry [^491^]. Budget for Advanced mode on production clusters; the cost is small relative to the operational risk of diagnosing purge storms and lock contention without execution plan visibility.

---

Monitoring tells you something is wrong. Planning prevents things from going wrong in the first place. Chapter 11 covers instance selection, sizing methodology, and cost optimization — building on the buffer pool sizing principles from Chapter 3, the purge-reader interaction from Chapter 5, and the replication insights from Chapter 8 to design clusters that are correct from day one.
