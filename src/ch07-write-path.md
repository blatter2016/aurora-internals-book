# Chapter 7: Write Path, Redo Logs, and Crash Recovery

A transaction modifies a page in the buffer pool (Chapter 3) while holding the appropriate locks (Chapter 6). The row versions are managed through undo logs, the read view is established, and the B-tree page has been updated in memory. Now what? How does that change become durable? And how does Aurora make it durable differently from every other MySQL deployment you have managed?

The write path is the most consequential architectural difference between Aurora MySQL and standard MySQL. Aurora's decision to send *only* redo log records to storage — letting storage nodes reconstruct data pages asynchronously — eliminates the majority of network I/O and removes the bottlenecks that constrain standard MySQL throughput. This chapter dissects that write path, explains which tuning parameters Aurora manages automatically (and which ones migrating DBAs waste time on), and covers the crash recovery mechanism that makes Aurora startups effectively instant.

## 7.1 The Standard MySQL Write Path: Seven Steps of Amplification

To appreciate what Aurora removes, you must first understand what standard MySQL/InnoDB does on every write.

In a traditional database, a single application write results in multiple actual I/Os: redo log writes, binary log writes, modified data page writes, doublewrite buffer writes, and metadata file writes. [^1^] The sequence is:

1. **Modify the buffer pool page** — The data page is changed in memory and marked dirty.
2. **Write the redo log (WAL)** — Redo log records are written to the log buffer, then flushed to disk on commit via the `innodb_flush_log_at_trx_commit` path.
3. **Write the doublewrite buffer** — Before writing a dirty page to its final location, InnoDB writes it to a contiguous doublewrite buffer to protect against torn pages. Every dirty page write becomes two physical writes. [^15^]
4. **Flush the dirty page** — The modified page is written to its final location in the tablespace file during background cleaning, LRU eviction, or checkpoint.
5. **Checkpoint** — InnoDB must periodically write all dirty pages older than a certain LSN to bound recovery time. When the redo log fills, a forced checkpoint creates I/O spikes that can stall foreground queries. [^16^]
6. **Write the binary log** — If replication is enabled, the change is also written to the binary log.
7. **Purge** — Old undo log records are cleaned up. With long-running transactions, purge can fall behind and the history list grows.

In a mirrored MySQL configuration across availability zones, each of these writes is amplified further by replication, creating what AWS terms an "untenable performance" situation for cloud-scale workloads. [^1^]

### Why Write Amplification Matters

Write amplification is not an abstract concern. A single `UPDATE` of a 100-byte row can generate megabytes of I/O: the 16 KB data page is written twice (doublewrite buffer + final location), the redo log and binary log are each synced, and a checkpoint may force hundreds of additional dirty pages to disk. This caps throughput in two ways. First, the database node must sustain all this I/O, which quickly saturates local disk or network. Second, the synchronous nature of these writes means worker threads spend significant time waiting for `fsync` rather than processing the next transaction. [^1^]

## 7.2 Aurora's Optimized Write Path

Aurora eliminates most of this amplification by changing one fundamental assumption: the compute node no longer writes data pages.

### Log-Only Writes

In Aurora, the only writes that cross the network are redo log records. No pages are ever written from the database tier — not for background writes, not for checkpointing, and not for cache eviction. [^2^] The log applicator is pushed to the storage tier where it generates database pages in the background or on demand. The central innovation: "the log is the database." [^3^] Aurora's distributed storage engine receives redo log records and applies them asynchronously on the storage nodes themselves. The compute layer ships only redo log records across the network. It does not write full data pages or maintain a doublewrite buffer. [^3^]

The Aurora write sequence is:

1. **Modify buffer pool page** — Same as standard MySQL.
2. **Generate redo log records** — Redo log records capturing the changes are created.
3. **Batch and shard log records** — Records are organized into batches, sharded by Protection Group (PG).
4. **Send to storage service** — Only redo log records cross the network.
5. **Storage applies quorum protocol** — 4 of 6 storage nodes must acknowledge.
6. **Asynchronous page materialization** — Storage nodes reconstruct pages in the background.
7. **Acknowledge commit** — Commit is acknowledged asynchronously.

This architecture is resilient to AZ+1 failures, reduces network traffic by writing only log records, and enables fast recovery because storage nodes continuously apply logs in the background. [^3^]

### Async Commit Architecture

In standard MySQL, when a transaction commits, the worker thread must wait for the redo log to be flushed to disk before returning to the client. Each commit is a potential serialization point.

In Aurora, transaction commits are completed asynchronously. When a client commits, the worker thread records the transaction's "commit LSN" on a commit queue and immediately returns to the task pool. [^21^] A separate driver thread advances the Volume Complete LSN (VCL) as storage acknowledgments arrive from the 4/6 quorum. When VCL advances, a dedicated commit thread scans the queue and acknowledges to clients all transactions whose commit LSN is at or below the new VCL. [^22^] Worker threads never pause for commits. There is no induced latency from group commits and no idle time for worker threads. [^22^]

### Group Commit and Boxcar Batching

While Aurora's native commit mechanism does not use traditional group commit, two batching optimizations remain. First, Aurora handles write batching through a "boxcar" mechanism: it submits the asynchronous network operation when it receives the first redo log record, but continues filling the buffer until the network operation executes. This packs records together to minimize network packets without adding boxcar latency. [^24^]

Second, when binary logging is enabled, the binary log layer still uses MySQL's traditional group commit (Flush, Sync, Commit stages). This is why enabling the binlog adds measurable latency to Aurora commits even though the redo log path is fully async. Because Aurora already batches log record writes between transactions, the parameter `innodb_flush_log_at_trx_commit` has limited impact for high-throughput write workloads. [^23^]

The following table compares the full write path between standard MySQL and Aurora MySQL across every stage:

| Stage | Standard MySQL/InnoDB | Aurora MySQL | Implication |
|---|---|---|---|
| What crosses network | Data pages + redo logs + doublewrite | Redo log records only | Aurora generates ~7.7x fewer I/Os per transaction [^18^] |
| Page writes | Engine writes dirty pages to data files | Engine never writes pages; storage materializes them | Eliminates doublewrite buffer entirely [^15^] |
| Doublewrite buffer | Required for torn page protection | Eliminated; storage handles consistency via log-structured design | Removes 1 redundant write per dirty page [^15^] |
| Checkpointing | Engine-driven, periodic, causes I/O spikes | Continuous, distributed, storage-driven | No checkpoint stalls or spikes [^16^] |
| Redo log durability | `fsync` to local disk | 4/6 quorum across distributed storage | Survives AZ+1 failure [^10^] |
| Commit model | Synchronous: worker waits for log flush | Asynchronous: worker continues, dedicated thread acks | Worker threads never block on commit [^22^] |
| Page materialization | All dirty pages flushed at checkpoint | Only pages with long modification chains materialized | Background work proportional to per-page churn [^17^] |
| Write I/O unit | Full InnoDB page (16 KB) | 4 KB log record units | Finer granularity, better batching [^20^] |

The practical impact is substantial. In benchmark tests using SysBench write-only workload, Aurora with replicas sustained 35 times more transactions than mirrored MySQL over a 30-minute period. The number of I/Os per transaction in Aurora was 7.7 times fewer than in mirrored MySQL, despite amplifying writes six times for Aurora replication. [^18^] Each storage node sees unamplified writes since it is only one of the six copies, resulting in 46 times fewer I/Os requiring processing at the storage tier. [^18^]

## 7.3 Redo Log in Aurora

The redo log is the central nervous system of Aurora's architecture. Understanding its format, its chain structures, and what parameters do *not* control it is essential for production operations.

### Redo Log Format

The redo log format in Aurora preserves the same fundamental structure as standard InnoDB redo logs but adds Aurora-specific metadata for distributed storage coordination. Each log record in Aurora contains a backlink that identifies the previous log record for its Protection Group. These backlinks enable tracking the point of completeness of log records at each segment to establish a Segment Complete LSN (SCL). [^5^]

Each log record stores three LSN values: the LSN of the preceding log record in the volume (for global log ordering), the previous LSN for the segment (for per-segment completeness tracking, yielding the SCL), and the previous LSN for the block being modified (for on-demand page materialization). The block chain is used by the storage node to materialize individual blocks on demand when a read request arrives for a page that has not yet been coalesced. The segment chain is used by each storage node to identify records it has not received and fill holes by gossiping with other storage nodes.

At a higher level, Aurora separates the concepts of *completeness* and *durability*. The Volume Complete LSN (VCL) represents the highest LSN for which all prior log records have met write quorum across all PGs. The Volume Durable LSN (VDL) represents the last LSN at or below VCL that marks a mini-transaction completion. Even if data is complete to LSN 1007, if only Consistency Point LSNs (CPLs) at 900, 1,000, and 1,100 were declared, Aurora truncates at 1,000. The system is complete to 1007, but only durable to 1,000. [^40^]

Mini-transactions (MTRs) — internal InnoDB constructs that group operations that must execute atomically, such as B-tree page splits — map directly to this chain structure. The redo log records representing the changes that must be executed atomically in each MTR are organized into batches that are sharded by the PGs each log record belongs to. The final log record of each MTR is tagged as a consistency point. [^7^] This ensures that structural operations like B-tree splits are applied atomically at storage nodes and at read replicas.

### Parameters That Don't Matter in Aurora

A significant trap for DBAs migrating from standard MySQL to Aurora is continuing to tune parameters that Aurora manages internally. The following parameters are either not applicable, not exposed, or have negligible impact in Aurora MySQL:

| Parameter | Standard MySQL Role | Aurora Status | Why It Doesn't Matter |
|---|---|---|---|
| `innodb_log_file_size` | Controls redo log capacity; critical for write throughput | Not applicable; not exposed in parameter groups [^25^] | Aurora manages log storage internally via distributed storage |
| `innodb_log_files_in_group` | Number of redo log files; defines total log space | Not applicable; not exposed [^25^] | Same reason — storage handles log management |
| `innodb_flush_log_at_trx_commit` (set to 0/2) | Major durability/performance tradeoff; often set to 2 for throughput | Limited impact due to async commits and batched log writes [^23^] | Aurora's async commit architecture already batches and optimizes log flushes |
| `innodb_doublewrite` | Protects against torn page writes | Eliminated; parameter does not apply [^15^] | Only redo log records (not pages) are sent to storage |
| `innodb_io_capacity` | Controls InnoDB background I/O rate | Locked at 200; not modifiable | Aurora storage auto-scales I/O capacity |
| `innodb_io_capacity_max` | Maximum I/O capacity for bursts | Locked at 2000; not modifiable | Same — storage manages throttling internally |
| `innodb_flush_method` | How InnoDB flushes data (O_DIRECT, fsync, etc.) | Not applicable | Aurora doesn't write to local filesystems |
| `innodb_flush_neighbors` | Whether flushing a page also flushes adjacent pages | Not applicable | Distributed storage doesn't benefit from neighbor flushing |
| `innodb_read_io_threads` | Number of read I/O threads | Set to 64; not modifiable | Aurora uses a different threading model for distributed storage |
| `innodb_write_io_threads` | Number of write I/O threads | Set to 4; not modifiable | Write I/O is handled asynchronously by storage |
| `innodb_adaptive_flushing` | Dynamically adjusts flush rate based on workload | Not applicable | Storage manages page materialization |
| `innodb_max_dirty_pages_pct` | Threshold to trigger aggressive flushing | Not applicable | No traditional dirty page flushing in Aurora |

The operational takeaway is straightforward: do not spend time tuning these parameters in Aurora. The parameter `innodb_log_file_size` is not applicable for Aurora MySQL and doesn't need to be tuned. In MySQL, increasing this variable might be helpful to increase the performance of write operations, but it is not valid for Aurora as it uses different storage to handle all write operations effectively. [^25^] Similarly, `innodb_flush_log_at_trx_commit` has limited impact in Aurora for high-throughput write workloads because of asynchronous commits and batched log writes. [^23^] Keep it at `1` (the default). Setting it to `0` or `2` provides minimal benefit and introduces unnecessary durability risk.

The one log-related parameter that *can* matter is `innodb_log_buffer_size`. If `SHOW STATUS LIKE 'Innodb_log_waits'` returns a non-zero value, the log buffer is too small and transactions are waiting for buffer space. The default 16 MB is usually sufficient; increase it only if you see log waits under sustained heavy write load.

### The Async Commit Paradox

Aurora's documentation claims that async commits mean "there is no induced latency from group commits and no idle time for worker threads." [^22^] Yet production workloads regularly show `io/aurora_redo_log_flush` as a top wait event in Performance Insights. These observations are not contradictory — they reveal a design paradox that every Aurora DBA should understand.

Async commits eliminate worker thread blocking, but the flush work still happens and blocks a *different* thread: the dedicated commit thread. The `wait/io/aurora_redo_log_flush` event occurs during write I/O operations for data persistence. Its increase could be due to increased write activity, storage performance issues, instance type mismatch, or background processes. When this commit thread becomes saturated, it manifests as the `io/aurora_redo_log_flush` wait event. In standard MySQL, the same class of wait event indicates disk I/O saturation. In Aurora, it indicates that the single commit thread cannot keep up with the rate of completed transactions, not that storage is slow. [^AWS re:Post^]

The remediation is different too. Batching writes into larger transactions reduces commit frequency, which is the correct response. Adding CPU will not help because the bottleneck is a single thread, not core capacity. Consider larger instance types with more I/O capacity if batching is insufficient. [^AWS re:Post^] This is a subtle but critical distinction: in Aurora, `io/aurora_redo_log_flush` is a throughput signal, not a storage latency signal. We will revisit this paradox in Chapter 9, where we examine how it appears in Performance Insights and what actions to take when it dominates your wait event profile.

## 7.4 Crash Recovery

The most dramatic operational benefit of Aurora's log-only architecture is crash recovery. Where standard MySQL must replay redo logs from the last checkpoint — a process that can take minutes for large databases — Aurora starts instantly because recovery is continuous and distributed.

### Continuous Distributed Recovery

In traditional databases, crash recovery requires starting from the most recent checkpoint and replaying redo logs from that checkpoint forward. In MySQL, this replay is single-threaded, and checkpoint intervals are typically 5 minutes — meaning up to 5 minutes of logs must be replayed sequentially before the database can accept connections. [^27^]

Aurora performs crash recovery asynchronously on parallel threads, so the database is open and available immediately after a crash. The underlying storage replays redo records continuously, whether in recovery or not. Coalescing is parallel, distributed, and asynchronous. [^27^] In Aurora, durable redo record application happens at the storage tier, continuously, asynchronously, and distributed across the fleet. Any read request for a data page may require some redo records to be applied if the page is not current. As a result, the process of crash recovery is spread across all normal foreground processing. Nothing is required at database startup. [^28^]

When a database instance starts after a crash, the recovery process is: (1) storage service self-recovery ensures a uniform view; (2) the database contacts each PG to establish a read quorum (3/6 segments); (3) from reported SCLs, the database recalculates PGCLs, VCL, and VDL; (4) log records above VDL are truncated via truncation ranges versioned with epoch numbers and written durably to storage; [^29^] (5) undo recovery runs for in-flight transactions; (6) Aurora increments an epoch to box out old instances with stale connections. Aurora also implements fast recovery for binlog by keeping events in memory initially, then writing to durable storage as a temporary binlog file that is simply dropped on recovery, enabling near-constant-time recovery regardless of transaction size. [^30^]

### Recovery Speed

Amazon Aurora recovers up to 97% faster than traditional databases. [^27^] In standard MySQL, recovery is dominated by single-threaded log replay since the last checkpoint — typically up to 5 minutes of logs. In Aurora, there is no replay at startup because storage nodes continuously apply log records in parallel. [^27^]

Failover statistics confirm this: 5-10 seconds for 40% of failovers, 20-30 seconds for 5%. Overall failover times are typically under 30 seconds. [^31^] Note that the 97% figure refers to crash recovery time (effectively zero at startup), while failover time includes detection, promotion, and DNS propagation.

### Reader Recovery After Failover

When a failover occurs in Aurora, a read replica is promoted to become the new writer. Because all instances share the same storage volume, the promoted instance does not need to replay logs — the storage is already consistent. The concern is buffer pool warmth, not storage consistency.

Aurora read replicas reuse the redo log stream and perform NO writes since storage is shared, fundamentally different from MySQL read replicas that maintain independent storage. [^32^] At read replicas, only log records with LSN ≤ VDL are applied, atomically in MTR chunks. In practice, each replica typically lags the writer by 20 ms or less. [^33^]

In Aurora, each DB instance's page cache is managed in a separate process from the database, allowing it to survive restarts. [^34^] For Aurora MySQL 2.10 and higher, in-Region readers do not reboot on writer failover, and a promoted reader retains its warm cache. [^35^]

The post-failover risk is a cold buffer pool. If the promoted instance was a small reader used for light queries, its cache may not contain the hot pages needed for OLTP workloads. Expect 10-30 minutes of degraded performance while the pool warms. Production deployments should maintain at least one read replica of the same instance class as the writer in failover tier 0.

---

The write path explains how a single instance survives. But Aurora is a distributed system. Chapter 8 follows the same write as it reaches read replicas — tracing how log records propagate, how the two-layer consistency model affects what applications see, and what happens when the writer fails and a reader must take its place.

[^1^]: [Verbitski et al., "Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases," SIGMOD 2017.](https://dl.acm.org/doi/10.1145/3035918.3056101)
[^2^]: [AWS Documentation, "Amazon Aurora Storage Overview."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Storage.html)
[^3^]: [AWS Documentation, "Amazon Aurora: How It Works — Storage."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/CHAP_Aurora.html)
[^5^]: [Aurora Architecture Deep Dive, "Segment Complete LSN and Log Chains."](https://aws.amazon.com/blogs/database/introducing-the-aurora-storage-engine/)
[^7^]: [AWS Documentation, "Mini-Transactions and Consistency Points in Aurora."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Storage.html)
[^10^]: [AWS Documentation, "Aurora Quorum Model — 4/6 Writes Across 3 AZs."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Storage.html)
[^15^]: [AWS Documentation, "Doublewrite Buffer Elimination in Aurora."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Performance.html)
[^16^]: [AWS Documentation, "Checkpointing in Standard MySQL vs Aurora."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Monitoring.html)
[^17^]: [AWS Documentation, "Page Materialization in Aurora Storage."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Storage.html)
[^18^]: [Verbitski et al., SIGMOD 2017 — Benchmark Results.](https://dl.acm.org/doi/10.1145/3035918.3056101)
[^20^]: [Aurora Storage Internals, "4 KB Log Record Units."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Storage.html)
[^21^]: [AWS Documentation, "Aurora Async Commit Architecture."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Storage.html)
[^22^]: [AWS Documentation, "Aurora Commit Thread Behavior."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Performance.html)
[^23^]: [AWS Documentation, "innodb_flush_log_at_trx_commit in Aurora."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.Parameters.html)
[^24^]: [AWS Documentation, "Boxcar Batching Mechanism."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Storage.html)
[^25^]: [AWS Documentation, "Parameters Not Applicable to Aurora MySQL."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Reference.ParameterDifferences.html)
[^27^]: [AWS Documentation, "Aurora Crash Recovery — 97% Faster."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Storage.html)
[^28^]: [AWS re:Invent 2018, Aurora Deep Dive Session.](https://www.youtube.com/watch?v=3PshvYmTv9M)
[^29^]: [AWS Documentation, "VDL Truncation and Epoch Fencing."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Storage.html)
[^30^]: [AWS Documentation, "Fast Binlog Recovery in Aurora."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Replication.html)
[^31^]: [AWS Documentation, "Aurora Failover Timing Statistics."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Reliability.html)
[^32^]: [AWS Documentation, "Aurora Read Replica Architecture."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Replication.html)
[^33^]: [AWS Documentation, "Typical Replica Lag — 20ms."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Replication.html)
[^34^]: [Aurora Architecture, "Survivable Page Cache."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Storage.html)
[^35^]: [AWS Documentation, "Reader Behavior During Failover (Aurora MySQL 2.10+)."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Replication.html)
[^40^]: [Aurora Storage Internals, "Completeness vs Durability — VCL vs VDL."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.Overview.Storage.html)
