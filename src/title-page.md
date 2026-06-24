# Aurora MySQL Internals

## A Production Engineer's Deep Guide

---

**From Storage Architecture to Production Battlefield**

---

This book is written for engineers who operate Aurora MySQL at scale — database administrators managing terabyte-class clusters, SREs who wake up to replica lag alerts at 3 AM, application developers who need to understand why their queries behave differently on Aurora than on standard MySQL, and architects evaluating whether Aurora is the right fit for their workload.

It assumes you know SQL, understand basic indexing, and have at least some production MySQL experience. You do not need a PhD in distributed systems, but you will need patience for acronyms — Aurora has many, and each one names a concept that can cause outages when misunderstood.

### Scope and Conventions

This book covers Amazon Aurora MySQL — the MySQL-compatible version of AWS's cloud-native relational database. While Aurora's compute layer runs a fork of community MySQL with InnoDB, its storage layer is a fundamentally different system. The book focuses on Aurora-specific behaviors: the distributed storage fleet, the quorum-based replication protocol, the redo log stream that replaces binlog shipping, and the operational patterns that emerge from these architectural choices.

Where Aurora preserves standard InnoDB semantics — B+ tree indexes, MVCC, row-level locking, transaction isolation — the book covers these concepts as they manifest in Aurora, with attention to behaviors that differ from standard MySQL. General InnoDB concepts are included when they form necessary context for understanding Aurora-specific features. Citations reference AWS documentation, the Aurora SIGMOD papers, and the MySQL/InnoDB source code where applicable.

All SQL queries and commands have been tested against Aurora MySQL 2.x (MySQL 5.7-compatible) and 3.x (MySQL 8.0-compatible) unless otherwise noted. Version-specific features are called out explicitly.
