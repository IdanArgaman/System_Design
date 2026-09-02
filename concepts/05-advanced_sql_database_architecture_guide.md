# Advanced SQL & Relational Database Architecture: Interview Study Guide

---

## Table of Contents
1. [Transaction Isolation & Locking Mechanisms](#1-transaction-isolation--locking-mechanisms)
   - Read Anomalies vs. Isolation Levels
   - Concurrency Control Strategies
2. [Physical Index Architecture (B+ Trees vs. LSM Trees)](#2-physical-index-architecture)
   - B+ Tree Mechanics & Traversal
   - Clustered vs. Non-Clustered Indexes
   - B+ Trees vs. Log-Structured Merge-Trees (LSM Trees)
3. [Phantom Reads & Next-Key Locking](#3-phantom-reads--next-key-locking)
   - Non-Repeatable Read vs. Phantom Read
   - Concrete Execution Scenario
   - Prevention via Next-Key Locking & Snapshot Isolation
4. [Distributed Transactions & Two-Phase Commit (2PC)](#4-distributed-transactions--two-phase-commit-2pc)
   - Two-Phase Commit Protocol Architecture
   - Edge Cases & Failure Modes (In-Doubt States, Partitions)
   - Mitigations & Modern Alternatives (3PC, Saga, Raft/Paxos)
5. [Advanced Concurrency Control & Write Skew](#5-advanced-concurrency-control--write-skew)
   - Write Skew Anomaly Explained
   - Serializable Snapshot Isolation (SSI) & SIREAD Locks
6. [Storage Engine Mechanics & Memory Layout](#6-storage-engine-mechanics--memory-layout)
   - Write-Ahead Logging (WAL) & Crash Recovery
   - Slotted Page Architecture
7. [Query Engine Mechanics & Join Execution Algorithms](#7-query-engine-mechanics--join-execution-algorithms)
   - Nested Loop Join
   - Hash Join
   - Sort-Merge Join
8. [Table Partitioning Strategies](#8-table-partitioning-strategies)
   - Range, List, and Hash Partitioning
   - Partition Pruning Mechanics
9. [Distributed Database Theorems](#9-distributed-database-theorems)
   - CAP Theorem & PACELC Theorem

---

## 1. Transaction Isolation & Locking Mechanisms

Database transactions adhere to **ACID** properties (Atomicity, Consistency, Isolation, Durability). Isolation defines how changes made by concurrent operations become visible to one another.

### Read Anomalies vs. Isolation Levels

| Isolation Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|---|:---:|:---:|:---:|
| **Read Uncommitted** | Allowed | Allowed | Allowed |
| **Read Committed** | Prevented | Allowed | Allowed |
| **Repeatable Read** | Prevented | Prevented | Allowed* |
| **Serializable** | Prevented | Prevented | Prevented |

*\*Note: Some MVCC engines (such as PostgreSQL) prevent phantom reads in plain `SELECT` queries even under `Repeatable Read` via point-in-time snapshots, though write anomalies like Write Skew can still occur.*

#### Definitions of Read Anomalies
* **Dirty Read:** Reading uncommitted data modified by another transaction that might subsequently roll back.
* **Non-Repeatable Read:** Re-reading an existing row within the same transaction yields altered data because a concurrent committed transaction modified or deleted it.
* **Phantom Read:** Re-executing a range predicate query yields a different set of rows because a concurrent committed transaction inserted or deleted records matching the predicate condition.

### Concurrency Control Strategies

* **Pessimistic Locking:** Assumes conflicts will occur frequently. Transactions acquire explicit locks before operating on records.
  - **Shared Locks (S-Lock):** Allows multiple concurrent readers; blocks exclusive locks.
  - **Exclusive Locks (X-Lock):** Granted to a single writer; blocks all other readers and writers.
  - *Risk:* Can introduce deadlocks, requiring background graph cycle detection or lock timeouts.
* **Multi-Version Concurrency Control (MVCC):** Used by PostgreSQL, InnoDB (MySQL), and Oracle. Writers do not overwrite existing data pages directly; instead, they create point-in-time tuple versions.
  - **Key Benefit:** Readers never block writers, and writers never block readers.

---

## 2. Physical Index Architecture

### B+ Tree Mechanics & Traversal

Relational engines heavily rely on **B+ Trees** for on-disk index structures. Unlike standard binary search trees or general B-Trees, B+ Trees are multi-way self-balancing search trees optimized for block storage I/O.

```
                  [ 50 ]
                 /      \
         [ 20 | 35 ]    [ 65 | 80 ]
        /     |    \   /     |    \
     [10..] [20..] [35..] [50..] [65..] [80..]  <-- Leaf Nodes (Doubly Linked List)
```

1. **Internal Nodes:** Store key values and child pointers strictly for navigation/routing. They contain no actual row data.
2. **Leaf Nodes:** Store actual data rows (clustered index) or pointers/Row IDs (non-clustered index). All leaf nodes are connected via a **doubly linked list**, making sequential range scans extremely fast.
3. **Time Complexity:** Search, insertion, and deletion operate in $O(\log N)$ time. Range queries execute in $O(\log N)$ to find the initial boundary, followed by sequential traversal across the leaf linked list.

### Clustered vs. Non-Clustered Indexes

* **Clustered Index:** Defines the physical sequential order of table records on disk. 
  - Only **one** clustered index per table (typically the Primary Key).
  - Leaf nodes contain the full data row payload.
* **Non-Clustered Index:** A distinct, secondary index structure sorted independently.
  - Leaf nodes store the indexed key column values alongside a pointer (Row ID or Primary Key value) pointing back to the primary/clustered data page.
  - Query execution requiring non-indexed columns causes a secondary lookup ("Bookmark Lookup" / "Index Sweep").

### B+ Trees vs. Log-Structured Merge-Trees (LSM Trees)

| Feature | B+ Tree | LSM Tree |
|---|---|---|
| **Primary Workload Optimization** | Fast Reads ($O(\log N)$), Range Scans | Heavy Write Throughput |
| **Write Pattern** | In-place updates causing random disk I/O | Out-of-place sequential appends |
| **Core Structure** | Tree of fixed-size blocks/pages | MemTable (RAM) + SSTables (Disk) |
| **Write Amplification** | High (page rewrites for minor row changes) | Lower initially; managed via background compaction |
| **Read Amplification** | Low (direct tree traversal) | Higher (must check MemTable + multiple SSTable levels) |

---

## 3. Phantom Reads & Next-Key Locking

### Non-Repeatable Read vs. Phantom Read

* **Non-Repeatable Read:** Concerns **existing rows being modified or deleted**.
* **Phantom Read:** Concerns **new rows being inserted** (or matching rows deleted) into a range predicate space.

### Concrete Execution Scenario

```
  Transaction A (Reporting)              Transaction B (Registration)
      |                                        |
  T1: BEGIN;                                   |
  T2: SELECT COUNT(*) FROM users               |
      WHERE tier = 'VIP';                      |
      (Returns: 10)                            |
  T3:                                      BEGIN;
  T4:                                      INSERT INTO users (name, tier)
                                           VALUES ('Alex', 'VIP');
  T5:                                      COMMIT;
  T6: SELECT COUNT(*) FROM users               |
      WHERE tier = 'VIP';                      |
      (Returns: 11)  <-- PHANTOM ROW           |
  T7: COMMIT;                                  |
```

### Prevention Mechanics

1. **Next-Key Locking (Pessimistic - MySQL InnoDB):**
   - **Index-Record Lock:** Locks the specific index records matching the predicate.
   - **Gap Lock:** Locks the empty space ("gap") before and after matching index records.
   - **Next-Key Lock:** Combination of an index record lock and a gap lock on the space immediately preceding it. This prevents concurrent sessions from inserting new records into any gap covered by the scanned range.
2. **Snapshot Isolation (Optimistic - MVCC):**
   - Reads evaluate records against a consistent snapshot timestamp generated at transaction start (`TransactionReadView`).
   - Inserts committed after the snapshot creation timestamp are invisible to the transaction's read queries.

---

## 4. Distributed Transactions & Two-Phase Commit (2PC)

A distributed transaction guarantees atomic commits across separate physical node databases connected over a network.

### Two-Phase Commit Protocol Architecture

```
  Coordinator                Participant 1           Participant 2
      |                           |                       |
      |--- 1. Prepare (TxId) ---->|                       |
      |-------------------------- 1. Prepare (TxId) ----->|
      |                           |                       |
      |<-- "VOTE_COMMIT" ---------|                       |
      |<-------------------------- "VOTE_COMMIT" ---------|
      |                                                   |
      | [All Voted YES -> Decide COMMIT]                  |
      |                                                   |
      |--- 2. Global Commit ----->|                       |
      |-------------------------- 2. Global Commit ------>|
      |                           |                       |
      |<-- ACK -------------------|                       |
      |<-------------------------- ACK -------------------|
```

#### Phase 1: Prepare (Voting Phase)
1. The Coordinator generates a global $\text{TxId}$ and sends `PREPARE` to all Participants.
2. Participants execute local transaction operations up to commit, log operations to their local Write-Ahead Log (WAL), acquire resource locks, and vote **VOTE_COMMIT** (YES) or **VOTE_ABORT** (NO).

#### Phase 2: Commit / Abort (Execution Phase)
1. If **all** participants vote YES, the Coordinator writes `GLOBAL_COMMIT` to its log and broadcasts `COMMIT`.
2. If **any** participant votes NO or times out, the Coordinator writes `GLOBAL_ABORT` and broadcasts `ROLLBACK`.
3. Participants execute the command, release locks, and reply with an `ACK`.

### Edge Cases & Failure Modes

2PC is a **blocking protocol**. Node failures can lock database resources indefinitely.

* **Coordinator Crash After Phase 1 (In-Doubt State):** Participants voted YES and are holding local locks, but haven't received the final decision. They are stuck in an **in-doubt state** and cannot unilaterally abort because another node may have received a `COMMIT`. They must block until the Coordinator recovers and replays its WAL.
* **Network Partition During Phase 2:** Partial message delivery causes some nodes to receive `COMMIT` while others remain blocked in the prepared state.
* **Participant Failures Prior to Voting:** Safe failure mode; Coordinator times out, assumes a NO vote, and triggers global abort.

### Architectural Mitigations & Alternatives

* **3PC (Three-Phase Commit):** Adds a `Pre-Commit` state and explicit timeout rules to prevent indefinite blocking, but fails under network partitions.
* **Saga Pattern:** Replaces global multi-node locks with localized transactions. If step $N$ fails, compensating transactions are executed in reverse ($N-1 \dots 1$). Provides eventual consistency.
* **Consensus-Driven Systems (Spanner / CockroachDB):** Runs 2PC across shards where each individual shard is a replicated consensus group using **Raft** or **Paxos**, combined with bounded clock sync (Google TrueTime API).

---

## 5. Advanced Concurrency Control & Write Skew

### Write Skew Anomaly

Write Skew occurs under **Snapshot Isolation** when two concurrent transactions read overlapping datasets, make decisions based on that data, and update disjoint sets of records that jointly violate a domain invariant.

#### Concrete Example: On-Call Doctors Invariant
* **Rule:** A clinic must have at least **1 doctor on call** at all times.
* **Current State:** Doctor A and Doctor B are both `ON_CALL`.

```
  Transaction 1 (Doctor A requesting off)    Transaction 2 (Doctor B requesting off)
      |                                          |
  T1: BEGIN (Snapshot T1);                     BEGIN (Snapshot T1);
  T2: SELECT COUNT(*) WHERE status='ON_CALL';  SELECT COUNT(*) WHERE status='ON_CALL';
      (Returns 2 -> Invariant safe)              (Returns 2 -> Invariant safe)
  T3: UPDATE doctors SET status='OFF'          UPDATE doctors SET status='OFF'
      WHERE name = 'Doctor A';                   WHERE name = 'Doctor B';
  T4: COMMIT;                                  COMMIT;
```
* **Result:** Both transactions commit successfully because they modified **different rows**. However, 0 doctors remain on call, violating the system invariant.

### Serializable Snapshot Isolation (SSI)

Modern relational databases (e.g., PostgreSQL) prevent write skew using **Serializable Snapshot Isolation (SSI)**:
* SSI tracks **SIREAD locks** (ephemeral locks tracking read dependencies).
* It maintains a dynamic dependency graph for active transactions.
* If a cycle of read-write dependencies (a `rw-antidependency` overlap) is detected between concurrent transactions, the engine aborts one of the transactions with a `serialization failure`.

---

## 6. Storage Engine Mechanics & Memory Layout

### Write-Ahead Logging (WAL) & Crash Recovery

To optimize throughput, databases delay writing modified data pages from memory to disk. Durability is maintained via **Write-Ahead Logging (WAL)**:

1. **Log First Principle:** Any modification to a data page must first be written sequentially to the WAL file on disk before the dirty page in the buffer pool can be written to data files.
2. **Commit Criteria:** A transaction commit is acknowledged once its WAL record hits non-volatile disk storage.
3. **Checkpointing:** Periodically, dirty memory pages are flushed to disk, and a checkpoint record is written to the log.
4. **Crash Recovery Algorithm (ARIES model):**
   - **Analysis Pass:** Scans log forward from last checkpoint to identify active transactions and dirty pages at crash time.
   - **Redo Phase:** Replays log records forward to restore the system state exactly to the instant of crash.
   - **Undo Phase:** Reverses the actions of all transactions that were uncommitted at the time of the crash.

### Slotted Page Architecture

Disk files are partitioned into uniform pages (e.g., 8KB in PostgreSQL). Within each page, data is managed using a **Slotted Page** layout:

```
+-------------------------------------------------------+
| Page Header | Pointer Slot 1 | Pointer Slot 2 | ...    |
+-------------------------------------------------------+
|                   <-- Free Space -->                  |
+-------------------------------------------------------+
| ... | Tuple 2 Data             | Tuple 1 Data         |
+-------------------------------------------------------+
```

* **Header & Pointer Slots:** Grow from top to bottom. Each slot contains a byte offset and length pointing to the corresponding tuple.
* **Tuple Payload:** Written from the bottom of the page upward.
* **Indirection Advantage:** Deleting or shifting tuple data physically within the page only requires updating the offset pointer in the slot array. External indexes reference the static tuple ID `(PageID, SlotIndex)`, avoiding index updates when rows shrink or shift internally.

---

## 7. Query Engine Mechanics & Join Execution Algorithms

When processing joins, the database execution planner selects one of three core algorithms based on data size, index availability, and memory parameters (`work_mem`).

### 1. Nested Loop Join
* **Mechanism:** Iterates through every row of the outer table and scans the inner table for matching rows.
* **Time Complexity:** $O(M \times N)$ unindexed; $O(M \log N)$ indexed inner table.
* **Best Use Case:** Small outer table dataset paired with a indexed inner table join key.

### 2. Hash Join
* **Mechanism:**
  1. **Build Phase:** Reads the smaller table into memory and constructs an in-memory hash table based on the join key.
  2. **Probe Phase:** Scans the larger table, hashes its join key, and probes the hash table for matches.
* **Time Complexity:** $O(M + N)$.
* **Best Use Case:** Large, unindexed datasets evaluating equality conditions (`=`).
* **Memory Spill:** If the hash table exceeds available memory (`work_mem`), the planner partitions data into disk-backed batch files.

### 3. Sort-Merge Join
* **Mechanism:**
  1. **Sort Phase:** Both tables are sorted on their join keys (if not already pre-sorted by an index).
  2. **Merge Phase:** Scans both sorted inputs in parallel, matching identical key values.
* **Time Complexity:** $O(M \log M + N \log N)$ for sorting; $O(M + N)$ for merging.
* **Best Use Case:** Large datasets, range/inequality joins (`<`, `>=`, `<=`), or inputs already pre-sorted by B+ Tree indexes.

---

## 8. Table Partitioning Strategies

Partitioning splits a massive logical table into smaller physical sub-tables while presenting a unified interface to applications.

```
                      +-------------------+
                      |   Logical Table   |
                      +-------------------+
                                |
             +------------------+------------------+
             |                  |                  |
    +-----------------+ +-----------------+ +-----------------+
    | Partition: 2024 | | Partition: 2025 | | Partition: 2026 |
    +-----------------+ +-----------------+ +-----------------+
```

### Partitioning Types

1. **Range Partitioning:** Maps rows based on column value boundaries (e.g., `created_at BETWEEN '2026-01-01' AND '2026-02-01'`). Ideal for time-series, log data, and archiving.
2. **List Partitioning:** Maps rows based on explicit value sets (e.g., `region IN ('EU', 'US')`). Ideal for multi-tenant or multi-region data routing.
3. **Hash Partitioning:** Applies a hash function to the partition key modulo $N$ partitions (`hash(id) % N`). Ideal for evenly balancing high-throughput tables lacking natural range boundaries.

### Partition Pruning

The query optimizer analyzes `WHERE` predicate filters during execution planning. If a query filters on the partition key, **Partition Pruning** excludes non-relevant sub-tables from execution entirely, drastically reducing disk I/O and index scan overhead.

---

## 9. Distributed Database Theorems

### CAP Theorem
In a distributed data store, you can only simultaneously provide **two** of the following three guarantees during a network failure:
* **Consistency (C):** Every read receives the most recent write or an error.
* **Availability (A):** Every non-failing node returns a non-error response (without guarantee that it contains the latest write).
* **Partition Tolerance (P):** The system continues operating despite dropped or delayed messages between nodes.

*In practice, network partitions ($P$) are inevitable, so distributed systems must choose between **CP** (Consistency over Availability) or **AP** (Availability over Consistency).*

### PACELC Theorem
Extends CAP to model system trade-offs during normal (non-partitioned) operation:

$$\text{If } \mathbf{P} 	ext{ (Partition): Choose } \mathbf{A} 	ext{ (Availability) vs. } \mathbf{C} 	ext{ (Consistency)}$$
$$\text{Else } \mathbf{E} 	ext{: Choose } \mathbf{L} 	ext{ (Latency) vs. } \mathbf{C} 	ext{ (Consistency)}$$

* **Example Systems:**
  - **MongoDB / HBase:** PC/EC (Favors consistency both during partitions and normal execution).
  - **Cassandra / DynamoDB:** PA/EL (Favors availability during partitions and low latency during normal execution).
  - **Spanner / CockroachDB:** CP/EC (Strictly consistent, synchronous replication via Paxos/Raft).
