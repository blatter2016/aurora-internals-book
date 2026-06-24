# Chapter 2: InnoDB Table Structure and Row Format

Aurora's storage fleet manages 16KB pages — Chapter 1 showed how those pages materialize from redo logs at the storage layer. But what IS a page? What lives inside it?

This chapter descends one level into the physical structures that InnoDB places on top of Aurora's distributed storage. Every concept here is standard InnoDB: tablespaces, page layout, B+ tree indexes, row formats, and the hidden system columns that make MVCC possible. Aurora does not change these structures. What Aurora changes is how pages are written and reconstructed. Understanding both layers — the InnoDB structures in this chapter and the storage machinery beneath them — is prerequisite work for debugging query slowdowns, purge blocking, and I/O spikes.

## 2.1 Tablespaces and the Physical Layer

### 2.1.1 InnoDB Tablespace Hierarchy

Every InnoDB table and index lives inside a *tablespace* — a logical container for pages, extents, and segments. Aurora MySQL 3.x (MySQL 8.0 compatible) supports four tablespace types, each with distinct operational implications.

| Tablespace Type | File Location | Contents | Aurora-Specific Notes |
|---|---|---|---|
| **System tablespace** (`ibdata1`) | Shared cluster volume | Data dictionary, doublewrite buffer (disabled in Aurora), change buffer, undo logs (pre-8.0) | In Aurora, change buffering is disabled (`innodb_change_buffering = none`) because changes must be immediately visible to shared storage |
| **File-per-table** (`*.ibd`) | Shared cluster volume | Individual table data and indexes | Default for user tables; enables `ROW_FORMAT=DYNAMIC`; each table's clustered index and secondary indexes reside here |
| **General tablespaces** (`*.ibd`) | Shared cluster volume | Multiple tables in a single tablespace file | Useful for grouping related tables; configurable with `CREATE TABLESPACE` |
| **Undo tablespaces** (`*.ibu`) | Shared cluster volume | Rollback segments and undo log records | Two default tablespaces (`innodb_undo_001`, `innodb_undo_002`) in Aurora 3.x; shared across all cluster instances |

The system tablespace is the oldest design: tables cannot be truncated individually, and the file grows monotonically. Aurora mitigates some of this by disabling the change buffer and doublewrite buffer, but `ibdata1` still houses the data dictionary and internal structures. File-per-table tablespaces (enabled by default) isolate each table's storage, making `TRUNCATE TABLE` near-instant and enabling transportable tablespaces.

Undo tablespaces merit particular attention in Aurora because they are **shared across the entire cluster**. In standard MySQL each instance maintains its own undo logs; in Aurora, the writer and all readers reference the same undo tablespaces on the shared storage volume [^331^]. This architecture enables sub-100ms replica lag but creates cluster-wide MVCC coupling: a read view opened on any reader blocks the writer's purge process, causing the History List Length to grow unbounded [^16^]. We will return to this coupling repeatedly in Part II.

### 2.1.2 Page Structure

The InnoDB page is the fundamental I/O unit — 16,384 bytes (16KB) by default. Aurora uses this size universally. Every read, write, and cache operation works in whole pages.

An index page (`FIL_PAGE_INDEX`) has the following layout:

```text
Offset Size Section
------ ---- -----------
0 38 File Header (FIL Header)
38 56 Page Header
94 13 Infimum Record (system minimum)
107 13 Supremum Record (system maximum)
120 var User Records (row data, variable length)
... var Free Space (shrinks during inserts)
... var Page Directory (slot array, grows upward from bottom)
16376 8 File Trailer (FIL Trailer)
```

The **File Header** contains the page number, tablespace ID, page type, LSN of last modification, and pointers to the previous and next pages at the same B+ tree level — forming a doubly-linked list that enables efficient range scans. The **Page Header** tracks heap top position, record count, garbage space, last insert position, and insert direction.

The **Page Directory** is an array of 16-bit offsets pointing to every 4th to 8th record, plus entries for `infimum` and `supremum`. When InnoDB searches within a page, it performs a binary search on directory slots first, then follows the singly-linked record list for at most 4-8 records — transforming an O(n) scan into O(log n) followed by a short linear probe.

In Aurora, pages are materialized on demand by the storage layer. The "log is the database" architecture means pages on storage are always consistent — there is no concept of a torn page or doublewrite buffer recovery [^15^]. The doublewrite buffer exists in the system tablespace for historical reasons but is effectively a no-op because the storage layer guarantees atomic page writes [^16^].

### 2.1.3 Extent Management

An **extent** is a contiguous allocation of 64 pages (1MB at 16KB page size). Extents are the unit of space allocation for segments, which back indexes and rollback segments. When a table grows, InnoDB allocates extents from the tablespace's free list.

This allocation strategy has visible performance implications. For sequential insert workloads — time-series data with an `AUTO_INCREMENT` primary key — InnoDB fills extents sequentially, producing contiguous reads during range scans and efficient read-ahead (controlled by `innodb_read_ahead_threshold`, default 56). For random inserts — UUID keys or hash-distributed sharding — records scatter across pages and extents, causing more frequent page splits, under-filled pages, and fragmented allocation. On Aurora, fragmentation amplifies storage costs because every page read counts toward I/O billing, even if half-utilized.

## 2.2 B+ Tree Indexes

InnoDB organizes all table data and secondary indexes as B+ trees. Every B+ tree node is exactly one page (16KB), and all leaf nodes sit at the same depth — ensuring predictable O(log N) lookup time.

### 2.2.1 Clustered Index Organization

The **clustered index** is the table. InnoDB stores the entire row data in the leaf pages of the clustered index, ordered by the primary key. If a table defines no explicit primary key, InnoDB selects the first `UNIQUE` index with all non-null columns; if none exists, it generates a hidden 6-byte `DB_ROW_ID` and builds the clustered index around it.

Leaf pages contain the full row — all user columns, the three hidden system columns (Section 2.3.2), and the record header. Non-leaf pages contain only the primary key value and a page pointer to the child. A primary key lookup traverses from root to leaf — typically 2-4 page reads for millions of rows — and the full row is found at the leaf.

The clustered index's physical ordering has critical write implications. Sequential inserts (monotonically increasing primary key) append to the rightmost leaf, minimizing splits. Random inserts (UUID, hashed values) scatter across the tree, triggering frequent splits and rebalancing that cascade up to the root — CPU-intensive and latch-contention-heavy operations. Aurora's storage layer handles split durability, but the split itself executes on the compute instance. In high-concurrency workloads, split contention on the rightmost page can limit throughput — one reason Aurora's async commit architecture is essential [^22^].

### 2.2.2 Secondary Indexes

Secondary indexes are separate B+ trees. Their leaf pages store the indexed column values plus the primary key value — **not** the full row data. When a query uses a secondary index, InnoDB performs a two-step lookup: first traverse the secondary index B+ tree to find the primary key, then traverse the clustered index to fetch the full row.

This "bookmark lookup" is a source of significant performance variation. If the clustered index page is in the buffer pool, the lookup is fast; if not, it triggers a storage read — and on Aurora, every storage read is billable. A query returning 1,000 rows via a secondary index may require 1,000 separate clustered index lookups. Covering indexes — indexes that include all columns needed by the query — eliminate this entirely.

Secondary indexes also carry an MVCC penalty. Their leaf pages do not store `DB_TRX_ID` or `DB_ROLL_PTR`; instead, each page stores `PAGE_MAX_TRX_ID`, the maximum transaction ID that modified any record on it. If this value is newer than the read view's up-limit, InnoDB must look up the primary key and traverse the version chain to determine visibility. Under old read views (high History List Length), a secondary index scan degenerates into a clustered index lookup plus undo log read for every row — explaining why queries become 10-100x slower when HLL spikes [^16^].

### 2.2.3 Index Page Splits and Merges

A page split occurs when a page is full and a new record must be inserted. InnoDB allocates a new page, redistributes records roughly 50/50, and inserts a new key-pointer pair into the parent. If the parent is also full, it splits too, potentially cascading to the root.

Monitor split rates:

```sql
SET GLOBAL innodb_monitor_enable = 'index_page_split';

SELECT NAME, COUNT, SUBSYSTEM, COMMENT
FROM information_schema.INNODB_METRICS
WHERE NAME LIKE '%split%';
```

Sustained high split rates indicate random insert patterns. In Aurora, splits cause storage I/O for new page allocation, plus CPU and latch contention on the compute instance [^15^].

Page merges are the inverse: when a page falls below 50% occupancy, InnoDB attempts to merge it with a neighbor. Merges become significant during bulk purge operations — the scenario that unfolds when a long-held read view is released and the purge thread aggressively cleans old versions.

## 2.3 Row Format and Hidden Columns

### 2.3.1 DYNAMIC Row Format (Default in Aurora 3.x)

Aurora MySQL 3.x defaults to `ROW_FORMAT=DYNAMIC` for all new tables, matching MySQL 8.0's `innodb_default_row_format`. DYNAMIC is a variation of the COMPACT format with a critical difference in how large variable-length columns are stored.

In the older COMPACT and REDUNDANT formats, variable-length columns store their first 768 bytes inline in the clustered index page, with overflow on separate pages. For wide rows, this fills B-tree nodes with column data rather than key values, reducing rows per page and making the index less efficient.

DYNAMIC stores long variable-length column values **entirely off-page**, with the clustered index record containing only a 20-byte pointer to the overflow page. Columns are chosen for off-page storage in order of length until the row fits. `TEXT` and `BLOB` columns of 40 bytes or less remain inline. This keeps clustered index leaf pages dense, improving cache efficiency and range scan performance.

| Characteristic | REDUNDANT | COMPACT | DYNAMIC (Aurora 3.x default) |
|---|---|---|---|
| Compact storage | No | Yes | Yes |
| Off-page storage threshold | 768-byte prefix | 768-byte prefix | Entire value off-page |
| Max index key prefix | 767 bytes | 767 bytes | 3,072 bytes |
| Barracuda file format required | No | No | Yes (default in 8.0) |

The practical impact is most visible for tables with large `VARCHAR` or `JSON` columns. With DYNAMIC, a table storing a 4KB JSON document keeps only a 20-byte pointer in the clustered index page — roughly 500-800 rows per 16KB page. With COMPACT, each row embeds 768 bytes of the JSON, reducing row density by an order of magnitude.

Check and convert row formats:

```sql
-- Check current row format for all tables
SELECT table_name, row_format, engine
FROM information_schema.tables
WHERE table_schema = 'production_db';

-- Convert a table to DYNAMIC (requires table rebuild)
ALTER TABLE large_events ROW_FORMAT=DYNAMIC;
```

### 2.3.2 Hidden System Columns

Every InnoDB row carries three hidden system columns invisible to `SELECT *` but essential to MVCC, transaction rollback, and clustered index construction:

| Column | Size | Purpose | Visible When |
|---|---|---|---|
| `DB_TRX_ID` | 6 bytes (48-bit) | Transaction ID of the last transaction that inserted or updated the row | `innodb_page_info` tool, `innodb_ruby` gem, or page-level inspection |
| `DB_ROLL_PTR` | 7 bytes | Rollback pointer: encodes undo tablespace ID, page number, and offset of the previous version's undo log record | Same as above |
| `DB_ROW_ID` | 6 bytes | Monotonically increasing row ID; used only when InnoDB generates the clustered index automatically | Same as above |

These 13 bytes are the physical foundation of everything in Part II.

`DB_TRX_ID` is a 48-bit unsigned integer. At 20,000 transactions per second, it would take approximately 446 years to exhaust the ID space — wraparound is not a practical concern. The value is updated on every INSERT or UPDATE (DELETE is internally an update that sets a delete flag). Read-only transactions receive a pseudo-ID derived from the transaction structure's memory address.

`DB_ROLL_PTR` encodes a three-part address: undo tablespace space ID, page number, and byte offset. This 7-byte pointer is the link in the version chain — each row version points to the undo log record of the previous version, forming a singly-linked list from newest to oldest. When purge removes an undo log record, it must also update the corresponding roll pointer — another reason high HLL increases write activity.

`DB_ROW_ID` is only relevant for tables without an explicit primary key. When InnoDB generates the clustered index automatically, it uses `DB_ROW_ID` as the key. Production tables should always define an explicit primary key.

### 2.3.3 How Hidden Columns Enable MVCC

The hidden columns transform abstract MVCC theory into a concrete, byte-level mechanism. When a transaction modifies a row, InnoDB:

1. Writes the old version to an undo log record (previous column values, previous `DB_TRX_ID`, previous `DB_ROLL_PTR`)
2. Updates the row in place in the clustered index leaf page, setting `DB_TRX_ID` to the current transaction and `DB_ROLL_PTR` to the newly created undo record
3. If the modification affects indexed columns, updates secondary indexes (which may involve delete-marking the old entry and inserting a new one)

When another transaction reads the row, it checks `DB_TRX_ID` against its read view using the `changes_visible()` algorithm:

```c
// Simplified visibility check
if (row_trx_id < read_view.m_up_limit_id ||
 row_trx_id == read_view.creator_trx_id)
 return VISIBLE; // Committed before oldest active, or self

if (row_trx_id >= read_view.m_low_limit_id)
 return NOT_VISIBLE; // Created after this read view

// Binary search in active transaction list
if (binary_search(read_view.m_ids, row_trx_id))
 return NOT_VISIBLE; // Still uncommitted at snapshot time

return VISIBLE;
```

If the current version is not visible, InnoDB follows `DB_ROLL_PTR` to the undo log, reconstructs the previous version using `trx_undo_prev_version_build()`, and checks again. This loop continues until a visible version is found or the chain ends. Each step requires reading and parsing an undo log record, then applying the update vector to reconstruct the historical version — CPU-intensive work that explains why high HLL causes slower SELECTs [^16^].

In Aurora, this mechanism operates at **cluster scope**. Because all instances share the same undo logs, a read view opened on any reader can block the writer's purge process. The read view's `m_low_limit_no` represents the oldest committed transaction number that must be preserved; the purge thread cannot remove any undo log with `trx_no < m_low_limit_no`. When a reader holds an old read view for hours, the undo log grows unbounded — a single `SELECT` on a reader can balloon HLL to millions, degrading every query on every instance [^16^].

To identify which instance holds the oldest read view:

```sql
-- Run on the writer instance
SELECT
 server_id,
 IF(session_id = 'master_session_id', 'writer', 'reader') AS role,
 replica_lag_in_msec,
 oldest_read_view_trx_id,
 oldest_read_view_lsn
FROM mysql.ro_replica_status;
```

The instance with the lowest `oldest_read_view_trx_id` is blocking purge. If it is a reader, terminate the long-running query or session — not the instance itself, which would lose the survivable page cache and force cold reads during purge recovery [^59^].

Every row carries its own transaction lineage in 13 bytes. `DB_TRX_ID` identifies who last modified the row; `DB_ROLL_PTR` locates how to reconstruct earlier versions. The History List Length is not a mysterious counter — it is the number of undo log records protecting historical row versions that some transaction, somewhere in the cluster, might still need to see.

---

Pages are the unit of storage. The buffer pool (Chapter 3) is where those pages live in memory.


## References

[^15^]: [AWS Documentation, "Doublewrite Buffer Elimination in Aurora."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Performance.html)
[^16^]: [AWS Documentation, "Checkpointing in Standard MySQL vs Aurora."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Monitoring.html)
[^22^]: [AWS Documentation, "Aurora Commit Thread Behavior."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Performance.html)
[^59^]: [AWS Documentation, "Aurora MySQL Wait Events."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/AuroraMySQL.Managing.Monitoring.html)
[^331^]: [AWS Documentation, "Aurora Custom Endpoints."](https://docs.aws.amazon.com/AmazonRDS/latest/AuroraUserGuide/Aurora.CustomEndpoints.html)
