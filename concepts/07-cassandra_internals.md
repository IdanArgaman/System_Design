# Apache Cassandra: Concepts, Internals & Storage Mechanics — Comprehensive Reference Guide

## Table of Contents
- [Executive Summary & Design Philosophy](#executive-summary--design-philosophy)
- [1. Cluster Architecture & Data Partitioning](#1-cluster-architecture--data-partitioning)
  - [1.1 Partition Keys](#11-partition-keys)
  - [1.2 Clustering Keys](#12-clustering-keys)
  - [1.3 Composite Partition Keys](#13-composite-partition-keys)
  - [1.4 Consistent Hashing & Murmur3 Partitioner (Token Hashing)](#14-consistent-hashing--murmur3-partitioner-token-hashing)
  - [1.5 Virtual Nodes (vnodes)](#15-virtual-nodes-vnodes)
  - [1.6 Replication Factor (RF)](#16-replication-factor-rf)
  - [1.7 Gossip Protocol & Failure Detection](#17-gossip-protocol--failure-detection)
- [2. Write Path Mechanics](#2-write-path-mechanics)
  - [2.1 Commit Log](#21-commit-log)
  - [2.2 MemTable](#22-memtable)
  - [2.3 SSTable (Sorted String Table)](#23-sstable-sorted-string-table)
  - [2.4 Step-by-Step Write Sequence](#24-step-by-step-write-sequence)
- [3. Storage Engine & On-Disk Data Structures](#3-storage-engine--on-disk-data-structures)
  - [3.1 Components of an SSTable Set](#31-components-of-an-sstable-set)
  - [3.2 Key Multi-Level Data Structures](#32-key-multi-level-data-structures)
  - [3.3 Secondary Indexes (2i / SAI)](#33-secondary-indexes-2i--sai)
  - [3.4 Tombstones](#34-tombstones)
  - [3.5 TTLs (Time-To-Live)](#35-ttls-time-to-live)
- [4. Read Path Mechanics & Optimization](#4-read-path-mechanics--optimization)
- [5. Compaction Strategies](#5-compaction-strategies)
  - [5.1 Size-Tiered Compaction Strategy (STCS)](#51-size-tiered-compaction-strategy-stcs)
  - [5.2 Leveled Compaction Strategy (LCS)](#52-leveled-compaction-strategy-lcs)
  - [5.3 Time-Window Compaction Strategy (TWCS)](#53-time-window-compaction-strategy-twcs)
- [6. Tunable Consistency & Data Repair](#6-tunable-consistency--data-repair)
  - [6.1 Consistency Levels](#61-consistency-levels)
  - [6.2 Achieving Strong Consistency (R + W > N)](#62-achieving-strong-consistency-r--w--n)
  - [6.3 Anti-Entropy Mechanisms (Data Repair)](#63-anti-entropy-mechanisms-data-repair)
- [7. Architecture Comparison: Cassandra vs. RDBMS vs. DynamoDB](#7-architecture-comparison-cassandra-vs-rdbms-vs-dynamodb)

---

## Executive Summary & Design Philosophy

Apache Cassandra is a distributed, wide-column NoSQL database designed to handle massive volumes of data across commodity servers with **no single point of failure** (SPOF), high availability, and linear scalability. Originally developed at Facebook (combining concepts from Google's Bigtable and Amazon's Dynamo) and later open-sourced, Cassandra relies on a peer-to-peer masterless architecture.

### Key Architectural Pillars
- **Masterless / Peer-to-Peer:** Every node in the cluster plays an identical role. Any node can accept read or write requests for any piece of data (acting as a Coordinator Node).
- **LSM-Tree Storage Engine:** Optimized for fast, append-only writes by buffering writes in memory (Memtable) and persisting sequentially to immutable disk files (SSTables).
- **Tunable Consistency:** Allows developers to trade off consistency for performance/availability per query (ranging from `ONE` and `QUORUM` to `ALL`).
- **Decentralized Cluster Management:** Utilizes the Gossip protocol for node discovery/health and Consistent Hashing (Virtual Nodes) for data distribution.

---

## 1. Cluster Architecture & Data Partitioning

Cassandra's data model dictates how data is routed, partitioned, and replicated across physical cluster nodes.

```
                     Consistent Hash Ring (Token Range: 0 to 2^127 - 1)
                                      
                                      Node A (Tokens: 0, 25, 50)
                                      /                        \
                                     /                          \
        Node D (Tokens: 15, 40, 65) <                            > Node B (Tokens: 10, 35, 60)
                                     \                          /
                                      \                        /
                                      Node C (Tokens: 20, 45, 70)
```

### 1.1 Partition Keys
* **Definition:** The primary component of a table's primary key structure used to determine data distribution across the cluster.
* **Function:** Cassandra applies a hash function (default: `Murmur3Partitioner`) to the partition key value to produce a token. This token determines which specific node(s) in the cluster ring will physically store the row.
* **Access Pattern Impact:** Queries that include the partition key are routed directly to the specific replica nodes holding that partition, enabling $O(1)$ query routing.

### 1.2 Clustering Keys
* **Definition:** Additional columns specified in the primary key definition after the partition key.
* **Function:** Controls the internal physical sort order of data *within* a single partition on disk.
* **Access Pattern Impact:** Allows efficient range queries (e.g., `>` or `<`), grouping, and ordered retrieval of rows inside a known partition without scanning other partitions.

### 1.3 Composite Partition Keys
* **Definition:** A partition key made up of multiple combined columns, defined using nested parentheses in CQL: `PRIMARY KEY ((col1, col2), col3)`.
* **Function:** Cassandra concatenates the values of `col1` and `col2` together before generating the token hash.
* **Use Case:** Used when a single column value does not offer enough cardinality or leads to excessively large partitions (e.g., combining `tenant_id` and `date_bucket` to prevent hotspotting).

### 1.4 Consistent Hashing & Murmur3 Partitioner (Token Hashing)
Cassandra maps row keys to physical nodes using **Consistent Hashing**:
1. When a row is written, its **Partition Key** is hashed using a partitioner (default: `Murmur3Partitioner`).
2. The hash output is a 64-bit integer token ranging from $-2^{63}$ to $2^{63}-1$.
3. The token space is conceptualized as a continuous ring. Each physical node is assigned one or more tokens on this ring.
4. Data is stored on the node whose assigned token range covers the row's hash token.

### 1.5 Virtual Nodes (vnodes)
In early versions, a physical node owned a single contiguous token range. Modern Cassandra uses **Virtual Nodes (vnodes)**:
- A single physical node hosts multiple virtual nodes (typically 128 or 256 vnodes per physical machine).
- **Advantages:**
  - **Balanced Data Distribution:** Prevents hotspots by scattering token ranges evenly across the cluster.
  - **Faster Bootstrapping & Repair:** When a new node joins or an existing node fails, stream repair operations occur concurrently across all machines in parallel rather than hitting a single neighboring node.

### 1.6 Replication Factor (RF)
* **Definition:** The total number of replica nodes across the cluster that must store a copy of each row.
* **Configuration:** Defined per Keyspace (e.g., `RF=3` means 3 independent nodes hold a copy of the data).
* **Topologies:**
  * `SimpleStrategy`: Places replicas sequentially around the hash ring (used for single data centers / testing).
  * `NetworkTopologyStrategy`: Allows defining specific replication factors per Data Center (DC) and distributes replicas across different racks to survive rack-level outages.

### 1.7 Gossip Protocol & Failure Detection
Cassandra does not rely on a master node (like Zookeeper in HDFS/HBase). Instead, it uses a peer-to-peer **Gossip Protocol**:
- **Mechanism:** Every second, each node selects a small number of random neighbors (typically 1–3) and exchanges state information (heartbeats, application state, schema updates).
- **Gossip Convergence:** Information spreads exponentially across the cluster ($O(\log N)$ rounds).
- **Phi Accrual Failure Detector:** Rather than a binary up/down timeout, Cassandra calculates a probabilistic value ($\Phi$) reflecting historical network latencies and heartbeat history. If $\Phi$ exceeds a threshold (e.g., $\Phi > 8$), the node is suspected to be down, preventing false positives caused by transient network hiccups or GC pauses.

---

## 2. Write Path Mechanics

Cassandra writes are **append-only**, making write latency extremely low (sub-millisecond) because no in-place disk reads or updates are required.

```
Incoming Write Request (Coordinator)
          │
          ├──> 1. CommitLog (Append-only to Disk for Data Recovery)
          │
          └──> 2. Memtable (In-Memory Data Structure)
                     │
                     └── (When Full / Flushed) ──> 3. SSTable (Immutable Disk File)
```

### 2.1 Commit Log
* **Type:** On-disk append-only log file.
* **Purpose:** Ensures **Durability** (the 'D' in ACID).
* **Behavior:** Every write mutation is written sequentially to the CommitLog on disk *before* or concurrently with memory insertion. Because it is append-only, disk write latency is extremely fast. If a node loses power, it replays un-flushed segments of the CommitLog upon reboot to recover lost state.

### 2.2 MemTable
* **Type:** In-memory sorted data structure (off-heap or heap-allocated, typically implemented as a skip-list or sorted map).
* **Purpose:** Serves write operations instantly in RAM while keeping data sorted by Partition Key and Clustering Keys.
* **Lifecycle:** Accepts continuous writes until memory thresholds (or age limits) are reached, after which its contents are flushed sequentially to disk as a new SSTable, and the corresponding CommitLog segment is truncated.

### 2.3 SSTable (Sorted String Table)
* **Type:** Immutable on-disk data file set.
* **Characteristics:**
  * Writes are append-only; SSTables are **never modified in-place**.
  * Data inside `Data.db` is strictly sorted by partition key token and clustering keys.
  * Overwrites or updates result in *new* SSTables containing newer cell timestamps rather than modifying old SSTables.

### 2.4 Step-by-Step Write Sequence

1. **Coordinator Selection:** A client sends a write request to any cluster node. This node acts as the **Coordinator** for that operation.
2. **Routing:** The Coordinator hashes the Partition Key to locate the replica nodes for that partition and forwards the payload to them.
3. **CommitLog Append:** On each replica node, the write is first appended sequentially to the **CommitLog** on disk. This guarantees durability in the event of power loss or crash before memory is flushed.
4. **Memtable Insertion:** Concurrently, the payload is written into the **Memtable** (an in-memory sorted data structure, typically a skip-list or sorted map).
5. **Acknowledgment:** Once written to Memtable and CommitLog on a sufficient number of replicas (determined by the requested Consistency Level), the Coordinator returns a success response to the client.
6. **Memtable Flush to SSTable:**
   - When the Memtable reaches capacity (or time-based flush limits), it is flushed sequentially to disk as a new **SSTable** (Sorted String Table).
   - Once flushed, the corresponding segment of the CommitLog is truncated.

---

## 3. Storage Engine & On-Disk Data Structures

Cassandra's storage engine relies on immutable files known as **SSTables** (Sorted String Tables). Because SSTables are immutable, updates and deletes never modify existing files; they write new versions or markers.

### 3.1 Components of an SSTable Set
Each SSTable flush generates a set of related binary files on disk:

| File Component | Description |
| :--- | :--- |
| **Data.db** | The actual row data sorted by Partition Key and Clustering Columns. |
| **Index.db** | Maps Partition Keys to byte offsets within `Data.db`. |
| **Summary.db** | An in-memory sampling index of `Index.db` used for quick lookup offsets without reading the full index. |
| **Filter.db** | A **Bloom Filter** kept in memory to quickly check if a partition key exists in the SSTable. |
| **TOC.txt** | Table of Contents listing all files belonging to the specific SSTable generation. |
| **Digest.crc** | Checksum verification file for data integrity. |

### 3.2 Key Multi-Level Data Structures

#### 1. Bloom Filter
- Probabilistic, in-memory (off-heap) data structure associated with each SSTable.
- **Purpose:** Answers the question: *"Does this SSTable definitely NOT contain this partition key?"* — eliminating unnecessary disk I/O during read operations.
- **Behavior:** Before reading an SSTable from disk, Cassandra checks its Bloom Filter. It guarantees either:
  * *"Key definitely does NOT exist in this SSTable"* → Skip disk read entirely.
  * *"Key MIGHT exist in this SSTable"* → Proceed to check key cache/partition index (potential false positive, never a false negative).

#### 2. Partition Summary
- Sampling of `Index.db` loaded directly into RAM (typically every $N$-th key, e.g., every 128th key).
- Used to quickly locate the approximate position in `Index.db`.

#### 3. Partition Index
- On-disk index structure (`Index.db`) mapping partition keys to the exact byte offset of the partition start in `Data.db`.

### 3.3 Secondary Indexes (2i / SAI)
* **Definition:** Built-in or Storage-Attached Indexes created on non-primary-key columns to query attributes without knowing the partition key.
* **Trade-off:** Requires index lookups across nodes, since the queried column isn't part of the routing token.

### 3.4 Tombstones

* **Definition:** A special deletion marker record written during delete operations or column updates.
* **Why It Exists:** Because SSTables are immutable, data cannot be erased in-place. Deleting a record appends a Tombstone containing a deletion timestamp.
* **Lifecycle:** During reads, tombstones hide deleted rows from query results. Tombstones remain on disk for a configurable window (`gc_grace_seconds`, default 10 days) to allow offline replicas to receive the delete marker before compaction permanently purges the marker and underlying data.

In an LSM-tree storage engine with immutable SSTables, a tombstone is a deletion marker written into a *new* SSTable rather than a deletion performed in-place on disk.

**How Deletions Create Tombstones**
* **Append-Only Write Path:** When a row or column is deleted, the database treats the deletion like a regular write — it logs the request, adds it to the in-memory MemTable, and eventually flushes a timestamped marker (the tombstone) to a new SSTable on disk.
* **Immutability:** Because SSTables are immutable, existing data cannot be overwritten or erased immediately when a delete command arrives.

**How Reads Handle Tombstones**
* **Shadowing Data:** During a read, the database scans SSTables in reverse chronological order (newest to oldest).
* **Filtering Out Records:** If a tombstone is encountered before the actual data record, it "shadows" that older value — the engine knows the key is deleted and ignores any older matching entries.

**How Tombstones Are Cleared (Compaction)**
* **Background Merging:** Physical removal happens later, during the background compaction process that merges multiple SSTables into a new, consolidated file.
* **Grace Periods:** To prevent "zombie data" (where a missed replica revives a deleted record during cluster repair), tombstones must age past a safe threshold (`gc_grace_seconds`) before they can be purged.
* **Dropping the Marker:** Once the grace period expires and compaction confirms no overlapping older versions of the data exist in un-participating SSTables, the tombstone and the shadowed data are dropped permanently to reclaim disk space.

![Tombstone Lifecycle & Compaction Diagram](Assets/Tombstone.png)
> Note: the actual deletion doesn't happen until the tombstone survives the grace period *and* compaction confirms no older shadowed data remains elsewhere — up until then, it's just a marker.

### 3.5 TTLs (Time-To-Live)
* **Definition:** An optional expiration period assigned to individual rows or columns at insertion time (e.g., `USING TTL 86400`).
* **Behavior:** When a TTL expires, Cassandra automatically converts the expired cell into a Tombstone without requiring an explicit `DELETE` command. The tombstone is then purged during standard compaction after `gc_grace_seconds`.

---

## 4. Read Path Mechanics & Optimization

Read operations in Cassandra are more complex than write operations because data for a single partition key may reside across multiple SSTables and the Memtable.

```
Incoming Read Request
          │
          ├──> Check Memtable (In-Memory)
          │
          └──> For each candidate SSTable:
                     │
                     ├── 1. Check Key Cache (RAM) ── [Hit] ──> Jump to Data.db Offset
                     │                                 │
                     │                              [Miss]
                     │                                 │
                     ├── 2. Check Bloom Filter (RAM) ── [No] ──> Skip SSTable
                     │                                   │
                     │                                [Yes]
                     │                                   │
                     ├── 3. Check Partition Summary (RAM)
                     │                                   │
                     ├── 4. Read Partition Index (Disk)
                     │                                   │
                     └── 5. Fetch Data Block from Data.db (Disk)
                                         │
                                         ▼
                           Merge & Reconcile Timestamps
                                         │
                                         ▼
                               Return Result to Client
```

### Read Sequence Execution

1. **Coordinator Routing:** The Coordinator identifies replica nodes responsible for the partition key.
2. **Local Node Lookup:** On each replica node processing the read:
   - **Memtable Search:** Checks active in-memory Memtables.
   - **Key Cache Check:** If enabled, checks the key cache for direct offsets into `Data.db`.
   - **Bloom Filter Verification:** If key cache misses, the node checks the Bloom Filter for candidate SSTables. If negative, the SSTable is skipped completely.
   - **Summary & Index Search:** Searches `Summary.db` to locate the offset in `Index.db`, then reads `Index.db` to get the exact position in `Data.db`.
   - **Data Extraction:** Reads data blocks from `Data.db`.
3. **Data Reconciliation (Last-Write-Wins):**
   - The coordinator merges results from all SSTables and Memtable.
   - Cassandra uses **Cell-Level Timestamps**. If conflicts exist (e.g., overlapping column values), the data point with the highest client timestamp wins (**Last-Write-Wins / LWW**).

---

## 5. Compaction Strategies

Over time, writing to disk creates many small SSTables. **Compaction** is the background process of merging multiple immutable SSTables into a single new SSTable. It:
  * Reconciles multiple versions of rows across SSTables using cell timestamps (**Last-Write-Wins**).
  * Hard-deletes tombstoned data whose garbage collection grace period (`gc_grace_seconds`) has expired.
  * Reclaims disk space and reduces read amplification (fewer SSTables to search during reads).

### 5.1 Size-Tiered Compaction Strategy (STCS)
- **How it works:** Groups SSTables of roughly equal size into sets. When a threshold (e.g., 4 SSTables) of similar size is reached, they are merged into one larger SSTable.
- **Best for:** Write-heavy workloads (e.g., logging, metrics, append-only streams).
- **Drawbacks:** High disk space overhead (can require up to 50% free disk space for compaction of huge SSTables).

### 5.2 Leveled Compaction Strategy (LCS)
- **How it works:** Inspired by Google Bigtable. SSTables are divided into exponential levels ($L_0, L_1, L_2 \dots$). $L_1$ has a fixed size (e.g., 10MB per file, 100MB total). Higher levels ($L_n$) hold $10\times$ the data of $L_{n-1}$.
- **Key Feature:** Guarantees that within $L_1$ and above, partition keys do not overlap across SSTables within the same level.
- **Best for:** Read-heavy workloads or transactional access patterns with frequent updates/deletes.
- **Drawbacks:** Higher I/O overhead due to frequent background compaction.

### 5.3 Time-Window Compaction Strategy (TWCS)
- **How it works:** Groups SSTables into time-based windows based on data insertion timestamp. Once a time window expires, its SSTables are compacted together once and left untouched.
- **Best for:** Time-series data, log storage, and workloads utilizing TTL (Time-To-Live).

---

## 6. Tunable Consistency & Data Repair

Cassandra delivers tunable consistency according to the **CAP Theorem** (prioritizing Availability and Partition Tolerance, trading strict consistency when configured). The consistency level is specified per query (read or write) and dictates how many replica nodes must acknowledge the request before returning a success signal to the client.

### 6.1 Consistency Levels

Common Consistency Levels:
- **`ONE` / `TWO` / `THREE`:** Requires acknowledgment from 1, 2, or 3 replicas.
- **`QUORUM`:** Requires acknowledgment from a strict majority of replicas:
  $$\text{QUORUM} = \left\lfloor \frac{\text{Replication Factor}}{2} \right\rfloor + 1$$
- **`LOCAL_QUORUM`:** Requires majority of replicas in the local Data Center (prevents cross-DC WAN latency).
- **`ALL`:** Requires acknowledgment from every single replica node.

### 6.2 Achieving Strong Consistency (R + W > N)
To achieve linearizable / strong consistency across reads and writes:
$$\text{Read Consistency (R)} + \text{Write Consistency (W)} > \text{Replication Factor (N)}$$

*Example:* With $N = 3$, using $W = \text{QUORUM} (2)$ and $R = \text{QUORUM} (2)$: $2 + 2 = 4 > 3$. This guarantees that at least one replica in the read set overlaps with the write set, returning the most recent update.

### 6.3 Anti-Entropy Mechanisms (Data Repair)

#### 1. Read Repair
- When a read request occurs with consistency $> \text{ONE}$, the coordinator compares digest checksums from replicas.
- If a mismatch is detected, a full data read is triggered from all replicas.
- The coordinator resolves the newest data using timestamps and asynchronously writes the updated value back to out-of-date replicas.

#### 2. Hinted Handoffs
- If a write target replica is unreachable/down, the Coordinator stores a local "hint" on its own disk.
- Hints contain the row mutation payload and timestamp.
- When the target node recovers, the coordinator streams stored hints to bring it up to date. (Default hint window: 3 hours).

#### 3. Anti-Entropy Active Repair (Nodetool Repair)
- A background process using **Merkle Trees** (hash trees of ranges of data).
- Nodes compare Merkle Trees over identical token ranges. If hashes mismatch, only the differing ranges are streamed between nodes.

---

## 7. Architecture Comparison: Cassandra vs. RDBMS vs. DynamoDB

| Architectural Axis | Apache Cassandra | Relational DB (e.g. PostgreSQL) | AWS DynamoDB |
| :--- | :--- | :--- | :--- |
| **Cluster Topography** | Peer-to-Peer Ring (No Master) | Primary - Standby / Master-Replica | Managed Distributed (Hidden Master/Storage) |
| **Write Model** | Append-Only (Memtable + SSTable) | B-Tree In-Place / Write-Ahead Log | B-Tree / SSD Log-Structured (Managed) |
| **Scaling** | Linear Horizontal (Add nodes to ring) | Vertical (Hardware), Read Scaling | Horizontal Auto-scaling |
| **Consistency Control** | Tunable per Query (`ONE`, `QUORUM`) | Strict ACID / Immediate | Eventual or Strong (Read flag) |
| **Single Point of Failure**| None | Yes (Primary node failover overhead) | None (Managed high-availability) |
