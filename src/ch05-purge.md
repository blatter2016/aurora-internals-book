# 5. The Purge System: Aurora's Silent Killer

MVCC creates version chains. Every `UPDATE` and `DELETE` generates an update undo record that preserves the previous row version, and those 7 bytes of `DB_ROLL_PTR` on every row link them together into a singly-linked list stretching from newest to oldest. Someone must clean up those versions when no transaction could possibly need them anymore. That someone is the purge system.

Purge lag is the most destructive and most common failure mode in Aurora MySQL production. It begins silently — a single long-running `SELECT` on a reader, an uncommitted transaction on the writer, an ETL job that opens a read view and forgets to close it. Hours later, the History List Length (HLL) has grown from a healthy few hundred to hundreds of thousands. Queries slow from milliseconds to seconds. CPU climbs. Storage bloats. Readers restart in a cascade, each reboot triggering another purge storm. This is the reader death spiral, the signature failure mode of Aurora MySQL [^317^].

Understanding the purge system is not academic — it is a survival skill.

## 5.1 Purge Architecture

The InnoDB purge system is a background garbage collector. Its job is to remove old row versions that are no longer needed by any active transaction. Every `UPDATE` and `DELETE` in InnoDB creates a historical version in the undo log. These versions must be preserved as long as any transaction might need them for a consistent read. Once all transactions are guaranteed to never need a particular version, the purge system physically removes it.

### 5.1.1 Purge Threads: Coordinator and Workers

The purge subsystem runs as a dedicated thread pool with a specific hierarchy. One **purge coordinator thread** orchestrates the work, while one or more **purge worker threads** execute the physical removal of records. The coordinator is responsible for scanning rollback segments, deciding which undo records are eligible for purging, and assigning batches of work to the worker threads. After workers complete a batch, the coordinator attempts to truncate undo tablespaces that have been fully emptied [^317^].

The number of purge threads is controlled by `innodb_purge_threads`. The default is 1 on instances with 16 or fewer logical processors, and 4 on larger instances. The maximum allowed value is 32 [^317^]. In Aurora, this parameter requires an instance restart to change — there is no online adjustment. For write-heavy workloads, increasing this to 4–8 can meaningfully improve purge throughput, though it does nothing if the real problem is a blocked purge horizon (Section 5.2).

The coordinator dynamically adjusts active worker count based on system load: it scales up when HLL exceeds `innodb_max_purge_lag`, and scales down when idle. Every 128 purge batches — configurable via `innodb_purge_rseg_truncate_frequency` — the coordinator attempts to free undo segments marked `TRX_UNDO_TO_PURGE` [^317^]. The batch size is controlled by `innodb_purge_batch_size` (default 300 undo log pages); increasing to 500–1000 helps under heavy write load but delays truncation checks.

### 5.1.2 History List Length (HLL)

The **History List Length** (`trx_rseg_history_len`) is the single most important metric for purge health. It represents the total number of committed transactions whose undo records still reside in the rollback segment history lists across all undo tablespaces. When a transaction commits after performing updates, HLL increments. When purge removes those old versions, HLL decrements [^317^].

A healthy cluster maintains HLL below 1,000. When HLL exceeds 10,000, purge has fallen behind. Above 100,000, queries traversing undo chains for MVCC version reconstruction degrade measurably [^OneUptime^]. Values in the millions predict imminent operational failure [^317^].

Check HLL on the writer via `SHOW ENGINE INNODB STATUS`:

```sql
-- Run on the writer instance
SHOW ENGINE INNODB STATUS\G
```

Look for these two lines in the output:

```
Purge done for trx's n:o < 2125991086 undo n:o < 0 state: running but idle
History list length 3462570
```

When HLL is elevated, `Purge done for trx's n:o` may be hours behind the current `Trx id counter`.

HLL is also available via `INFORMATION_SCHEMA.INNODB_METRICS`:

```sql
-- Enable purge metrics (one-time, survives restart)
SET GLOBAL innodb_monitor_enable = 'purge_%';
SET GLOBAL innodb_monitor_enable = 'trx_rseg_history_len';

-- Query current HLL and purge activity
SELECT NAME, COUNT, SUBSYSTEM, COMMENT
FROM information_schema.INNODB_METRICS
WHERE NAME IN (
  'trx_rseg_history_len',
  'purge_del_mark_records',
  'purge_upd_exist_or_extern_records',
  'purge_undo_log_pages',
  'purge_invoked',
  'purge_triggered_log_purges'
);
```

### 5.1.3 The Purge Process Step by Step

When a transaction commits, its INSERT undo records are marked as finished and can be freed immediately. Its UPDATE undo records — which contain the before-image of modified rows — are moved to the rollback segment's history list and tagged with the transaction's commit number (`trx_no`). The global `rseg_history_len` counter is incremented [^317^].

The purge system maintains a priority queue ordered by `trx_no`, ensuring undo logs are processed in commit order across all rollback segments. For each undo record, the purge worker performs one of two operations:

1. **For delete-marked records** (`TRX_UNDO_DEL_MARK_REC`): Physically remove the row from all secondary indexes, then remove it from the clustered index. The record was logically deleted at commit time; purge performs the physical removal.
2. **For update records** (`TRX_UNDO_UPD_EXIST_REC`): Clean up secondary index entries if the update modified columns that affect index ordering. If indexed columns changed, the old index entries must be removed.

After all workers complete a batch, the coordinator attempts to truncate freed undo segments. In Aurora MySQL 3.x, `innodb_undo_log_truncate` defaults to ON. When an undo tablespace exceeds `innodb_max_undo_log_size` (default 1GB), it is marked for truncation: its rollback segments are made inactive, existing transactions using those segments are allowed to finish, the purge system empties the segments, and the tablespace is truncated to its initial size of 16MB [^317^].

The following table summarizes the two undo log types and their purge behavior:

| Undo Log Type | Generated By | Contents | Purge Behavior After Commit |
|--------------|-------------|----------|----------------------------|
| **Insert undo** | `INSERT` | Primary key of inserted row only | Freed immediately on commit; no history list entry created [^317^] |
| **Update undo** | `UPDATE`, `DELETE` | Transaction ID, roll pointer to previous version, delta of modified columns | Moved to rollback segment history list; preserved until no read view needs it; removed by purge worker in commit order [^317^] |

Insert-heavy workloads generate far less purge pressure because insert undo never enters the history list. Update- and delete-heavy workloads drive HLL growth in direct proportion to transaction rate and purge lag duration.

## 5.2 How Long-Running Transactions Block Purge

The purge system cannot remove arbitrary old versions. Its progress is tied directly to the oldest active read view in the system.

### 5.2.1 The low_limit_no Mechanism

When a transaction opens a read view under `REPEATABLE READ` isolation, InnoDB captures the current transaction system state. One field in particular — `m_low_limit_no` — defines the **purge horizon**. This value is set to the smallest committed transaction number (`trx_no`) at the time the view was created [^317^]. All undo logs with `trx_no` less than `low_limit_no` can be safely purged, because every transaction that committed before the view was created is invisible to this reader. Conversely, **no undo log with `trx_no >= low_limit_no` can be purged** as long as this read view remains open.

The purge coordinator maintains a special "purge view" (`purge_sys->view`) that is a clone of the oldest active read view in the system. The purge process cannot remove any undo records with `trx_no` greater than or equal to this view's `low_limit_no` [^317^]. This is the fundamental constraint that makes long-running transactions so dangerous.

### 5.2.2 The Cascade Effect

The critical insight is that **one old read view prevents ALL subsequent undo log cleanup**. Consider a transaction that started 6 hours ago and holds a read view with `low_limit_no = 1000000`. Since then, 500,000 additional transactions have committed. Every single one of those 500,000 transactions has undo records sitting in the history list, consuming space and slowing queries — even though the old transaction may not touch any of those rows. The purge system is blocked at transaction 1,000,000 and cannot advance [^317^].

This is not a gradual degradation. It is a hard wall. The moment that old read view is released, the purge system can sprint forward through all 500,000 transactions. But until then, HLL grows without bound.

Under `READ COMMITTED` isolation, this problem is dramatically reduced because each `SELECT` statement gets a fresh read view. The view is released at the end of each statement, so the purge horizon advances continuously. This is why AWS recommends `READ COMMITTED` for reader instances [^15^].

### 5.2.3 Consequences of High HLL

When HLL grows, queries accessing recently modified rows must traverse longer undo chains for MVCC version reconstruction, increasing CPU and latency [^317^]. Undo tablespaces swell — in extreme cases to hundreds of gigabytes — consuming buffer pool space and driving storage I/O. Buffer pool saturation with undo pages creates OOM risk [^317^]. Readers with old read views see delayed data (apparent "stale" reads), which is an MVCC artifact distinct from replication lag.

The following table provides a severity scale with operational actions:

| HLL Range | Severity | Observable Symptoms | Required Actions |
|-----------|----------|---------------------|-----------------|
| < 1,000 | Normal | No performance impact | Continue routine monitoring; verify trend is flat or slowly decreasing |
| 1,000 – 10,000 | Elevated | Slight query latency increase on hot rows; undo tablespace growth | Identify blocking transaction via `mysql.ro_replica_status` and `innodb_trx`; set P2 alert [^OneUptime^] |
| 10,000 – 100,000 | Warning | Measurable SELECT slowdown; elevated CPU; growing undo files | Execute HLL diagnosis workflow (Figure 5.1); kill blocking transaction if safe; set P1 alert at 10,000 threshold [^59^] |
| 100,000 – 1,000,000 | Critical | Severe query degradation across all instances; readers falling behind | Immediate kill of blocking transaction; prepare for purge storm after release; monitor reader restart cycle [^317^] |
| > 1,000,000 | Emergency | Near-complete performance collapse; imminent reader restarts; OOM risk | Emergency transaction termination; consider read-only mode to stop HLL growth; page on-call engineer immediately |

The progression from "elevated" to "emergency" can occur in hours if the write rate is high and the blocking transaction remains open. A production incident at this scale typically begins with a developer running an analytical query on a reader or an uncommitted DML transaction on the writer — both entirely routine operations that become catastrophic only in Aurora's shared-undo architecture.

## 5.3 The Aurora-Specific Critical Behavior: Cross-Instance Purge Blocking

If the material in Section 5.2 applies to all InnoDB deployments, what follows is unique to Aurora and represents the most consequential architectural difference between Aurora MySQL and vanilla MySQL.

### 5.3.1 One Undo Log for the Entire Cluster

In standard MySQL replication, the primary and each replica maintain separate, independent undo logs. A long-running transaction on a replica blocks purge only on that replica's local undo log; the primary continues purging its own undo space unaffected [^331^].

In Aurora, the writer and all reader instances share a single storage volume. They share **one set of undo logs** [^15^]. A read view opened on any reader instance pins the purge horizon at its `low_limit_no`, and this blocks the purge system for the **entire cluster** — including the writer [^16^].

This means a data analyst running a 4-hour `SELECT` query on a `db.r6g.large` reader can degrade performance on the production writer. There is no access control mechanism that prevents a read-only user from triggering this cascade. The read-only credential that was granted for BI reporting becomes, in effect, a cluster-wide denial-of-service vector [^317^].

Vanilla MySQL isolates the blast radius of long-running reads to the instance where they execute. Aurora concentrates it across all 16 possible instances.

### 5.3.2 The Reader Death Spiral

The cascade is vicious and self-reinforcing. A reader holds an old read view, blocking purge. HLL grows; queries slow as undo chains lengthen. Slow queries consume more CPU on the reader. When the read view finally closes, the writer purges aggressively, generating a burst of redo log activity. The reader must apply all accumulated writes plus purge redo, falls behind, and is **automatically restarted** by the Aurora control plane [^317^]. After restart, the reader reconnects with a cold buffer pool. If another reader still holds an old view, the cycle repeats. Clusters can enter a state of continuous reader restarts.

Production incidents have documented HLL reaching 4,500,000 from a single long-running `SELECT` on a reader, degrading query latency 10x or more [^317^].

### 5.3.3 DDL: The Hidden Query Killer

Aurora enforces an additional behavior that surprises most operators: when DDL executes on the writer, Aurora **forcefully kills long-running transactions on readers** to acquire metadata locks [^Percona DDL^]. This is not a graceful timeout — it is explicit termination by the Aurora control plane. A schema migration during business hours silently kills analytical queries or ETL jobs on readers. Logical backups (`mysqldump`, `mydumper`) using long-running transactions are terminated mid-backup. The DBA running DDL sees no error; killed queries simply disappear from the reader's process list.

The only protection is to run all DDL during maintenance windows and check `mysql.ro_replica_status` for long-running transactions before any schema change.

## 5.4 Monitoring and Mitigation

Effective purge management in Aurora requires three capabilities: detecting HLL growth, identifying the blocking transaction, and remediating without making the problem worse. This section provides the operational tools for all three.

### 5.4.1 Essential Diagnostic Queries

**Query 1: Current HLL and purge state (run on writer)**

```sql
-- Shows HLL and how far behind purge is
SHOW ENGINE INNODB STATUS\G
-- Look for:
--   Purge done for trx's n:o < N
--   History list length NNNN
```

**Query 2: Purge metrics from INNODB_METRICS**

```sql
-- Requires enabling purge metrics first:
-- SET GLOBAL innodb_monitor_enable = 'purge_%';

SELECT NAME, COUNT, SUBSYSTEM, COMMENT
FROM information_schema.INNODB_METRICS
WHERE NAME LIKE 'purge_%' OR NAME = 'trx_rseg_history_len'
ORDER BY NAME;
```

**Query 3: Long-running transactions on the writer**

```sql
-- Find all transactions open longer than 5 minutes on the current instance
SELECT 
    a.trx_id,
    a.trx_state,
    a.trx_started,
    TIMESTAMPDIFF(SECOND, a.trx_started, NOW()) AS trx_seconds_open,
    a.trx_rows_modified,
    a.trx_isolation_level,
    b.USER,
    b.host,
    b.db,
    b.command,
    b.time AS processlist_time_sec,
    b.state,
    b.INFO AS current_query
FROM information_schema.innodb_trx a
JOIN information_schema.processlist b 
    ON a.trx_mysql_thread_id = b.id
WHERE TIMESTAMPDIFF(SECOND, a.trx_started, NOW()) > 300
ORDER BY a.trx_started;
```

**Query 4: Identify which cluster instance blocks purge (run on writer)**

```sql
-- This is THE query for purge diagnosis in Aurora.
-- Compare oldest_read_view_trx_id across instances.
-- The instance with the lowest value is blocking purge.
SELECT 
    server_id,
    IF(session_id = 'master_session_id', 'writer', 'reader') AS role,
    replica_lag_in_msec,
    oldest_read_view_trx_id,
    oldest_read_view_lsn
FROM mysql.ro_replica_status
ORDER BY oldest_read_view_trx_id;
```

When a reader shows a significantly lower `oldest_read_view_trx_id` than the writer, that reader is the purge blocker. Its `replica_lag_in_msec` will typically also be elevated. This single query should be in every Aurora DBA's runbook [^59^].

### 5.4.2 CloudWatch Alerting

Aurora publishes the `RollbackSegmentHistoryListLength` CloudWatch metric from the writer instance. Treat this as a **Tier 1 operational signal**.

| CloudWatch Metric | Namespace | Alert Threshold | Severity |
|-------------------|-----------|----------------|----------|
| `RollbackSegmentHistoryListLength` | `AWS/RDS` | > 10,000 for 2 consecutive periods | P1 (page on-call) [^59^] |
| `RollbackSegmentHistoryListLength` | `AWS/RDS` | > 100,000 | P0 (emergency response) |
| `AuroraReplicaLag` | `AWS/RDS` | > 1,000 ms | P2 (investigate) |
| `AuroraReplicaLagMaximum` | `AWS/RDS` | > 5,000 ms | P1 (reader in distress) |

The HLL alert at 10,000 provides sufficient lead time to identify and kill a blocking transaction before the situation escalates. Do not wait for query latency alerts to fire — by the time queries are visibly slow, HLL is already in the danger zone.

The following Mermaid diagram shows the HLL diagnosis workflow that should be followed when the P1 alert fires:

```mermaid
flowchart TD
    A[HLL alert fires: >10,000] --> B[Connect to writer: SHOW ENGINE INNODB STATUS]
    B --> C{HLL trending up?}
    C -->|Yes| D[Query mysql.ro_replica_status]
    C -->|No / stable| Z[Monitor; alert resolves itself]
    D --> E{Lowest oldest_read_view_trx_id on reader?}
    E -->|Yes| F[Connect to blocking reader]
    E -->|No (writer has oldest)| G[Query innodb_trx on writer]
    F --> H{Find trx >5 min in innodb_trx?}
    H -->|Yes| I[Kill transaction: CALL mysql.rds_kill_query(id)]
    H -->|No| J[Check application connection pool for idle transactions]
    G --> K{Blocking transaction found?}
    K -->|Yes| I
    K -->|No| L[Check for application bug: uncommitted DML, ORM session leak]
    I --> M[Monitor HLL: should drop within 10-30 minutes]
    J --> M
    L --> M
    M --> N{HLL drops?}
    N -->|Yes| O[Post-incident: implement READ COMMITTED on readers]
    N -->|No after 30 min| P[Investigate: high write rate overwhelming purge threads]
    P --> Q[Tune innodb_purge_threads + innodb_purge_batch_size]
```

**Figure 5.1**: HLL diagnosis and remediation workflow. The critical decision point is `mysql.ro_replica_status`: a reader with the lowest `oldest_read_view_trx_id` is the purge blocker. Do not reboot instances at high HLL — this destroys the survivable page cache and forces slower storage-level purge with additional I/O billing costs [^59^].

### 5.4.3 Mitigation Strategies

**Kill the blocking transaction.** This is the first and usually most effective response. Use `mysql.rds_kill_query(query_id)` or `KILL <query_id>` on the instance where the blocking transaction runs. Be aware that rolling back a large transaction can itself take significant time — a transaction that modified 10 million rows may spend 10–30 minutes in rollback state, during which it continues to hold its read view. Monitor `SHOW ENGINE INNODB STATUS` for rollback progress.

**Use `READ COMMITTED` on readers.** This is AWS's recommended configuration change for preventing purge blocking [^15^]. Under `READ COMMITTED`, each statement receives a fresh read view and releases it at statement end. No old view is held across statements, so the purge horizon advances continuously. Configure this at the session level for applications that can tolerate non-repeatable reads:

```sql
-- Set for current session on reader
SET SESSION TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

Or set it as the instance default via a custom DB parameter group (`tx_isolation = READ-COMMITTED`). Note that Aurora allows readers to have different isolation levels than the writer, so the writer can remain on `REPEATABLE READ` while readers use `READ COMMITTED` [^15^].

**Tune `innodb_purge_threads`.** For write-intensive workloads where HLL grows even without blocking transactions, increase purge parallelism. This requires a parameter group change and instance restart:

```
innodb_purge_threads = 8       -- For db.r6g.xlarge and above on write-heavy workloads
innodb_purge_batch_size = 500  -- Increase batch size to reduce coordination overhead
```

The maximum is 32, but values above 8 rarely provide additional benefit and can increase CPU contention. More threads help when undo records are spread across many rollback segments; they do nothing if a single old read view blocks the purge horizon.

**Never reboot to "fix" HLL.** This is the most common operational mistake. When HLL is high, the buffer pool contains hot undo pages that purge can access quickly from memory. A reboot destroys the survivable page cache. After restart, purge must read undo pages from the cluster volume, which is significantly slower and incurs additional I/O billing costs [^59^]. If the reboot was an attempt to clear a blocking transaction, that transaction will simply reconnect and open a new read view. Always identify and kill the blocking transaction explicitly.

**Offload long-running queries.** Use **binlog replicas** (external MySQL with independent undo logs) for analytical workloads [^317^], **Aurora database clones** (independent storage volume), or **S3 export** for one-time analysis via Athena.

**Break up large transactions.** Process updates in batches of 1,000–10,000 rows with explicit `COMMIT` between batches. This keeps undo logs small and allows purge to interleave cleanup.

**Let it purge before maintenance.** Before planned shutdowns, reduce write workload and allow HLL to drain. Purge after a cold restart is substantially worse with an empty buffer pool [^317^].

---

The purge system gives no warning until the damage is severe — a healthy HLL of 500 can become 500,000 in the span of one long-running report query. The defense is threefold: alert on HLL before it becomes symptomatic, diagnose with `mysql.ro_replica_status` to find the exact blocking instance, and remediate by killing the blocking transaction — never by rebooting. The DBA who internalizes this workflow avoids the most common and most destructive failure mode in Aurora MySQL.

Purge failures are a specific kind of concurrency problem. But they're not the only one. Chapter 6 examines the broader landscape of locks and deadlocks — the everyday concurrency control that every production DBA must understand.
