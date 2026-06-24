# Part IV: The Battlefield

You now understand Aurora's internals better than 90% of practicing DBAs. You have traced the storage architecture — six copies across three availability zones, quorum writes, the log-structured storage fleet. You have dissected the 16 KB InnoDB page, followed the B+ tree from clustered index through secondary lookups, and mapped the buffer pool's LRU and flush lists. You know how MVCC constructs read views, how the purge thread cleans up undo records, and why a single long-running `SELECT` on a reader can block the writer's purge for the entire cluster. You understand the write path's async commit architecture, the two-layer consistency model that produces replica lag in milliseconds but stale reads in hundreds of milliseconds, and the query optimizer's behavior without Query Plan Management.

This final part is about using that knowledge to run clusters that don't page you at 3 AM.

Every chapter here is an integration. Chapter 10 draws on buffer pool mechanics (Chapter 3), purge behavior (Chapter 5), and replication consistency (Chapter 8) to build a monitoring strategy that watches causes instead of symptoms. Chapter 11 connects instance sizing to buffer pool pressure, reader-writer purge interactions, and I/O cost modeling — the financial consequences of architectural decisions from Parts I and II. Chapter 12 separates the parameters that matter from the parameter delusion, using the write path (Chapter 7), locking (Chapter 6), and purge (Chapter 5) insights to focus tuning effort where it has impact. Chapter 13 has no new facts. It maps the connections between every subsystem covered in the preceding twelve chapters — the cascade patterns that emerge when you hold the full architecture in your head at once.

This is where you earn back the time invested in reading the first three parts.

The DBAs who thrive with Aurora are not the ones who memorize the most parameters. They are the ones who can trace a symptom through three architectural layers to its root cause in under five minutes. They recognize that a CPU spike is a buffer cache miss problem, which is a working-set-exceeds-memory problem, which is an instance-sizing problem. They know that a replication lag spike is a redo application problem, which is a buffer pool eviction problem, which is a reader-under-provisioned problem. They see the cascade before it becomes an outage because they understand the connections between the parts.

The chapters that follow make those connections explicit.
