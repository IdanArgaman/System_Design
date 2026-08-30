# Kafka Partitioning & Key Hashing Strategy: Summary

This summary captures the core principles, guarantees, and trade-offs of using Kafka key-based partitioning for high-cardinality keys (e.g., stock market tickers).

---

## 1. Key-to-Partition Mapping

* **Partition Count vs. Key Count:** The number of Kafka partitions does **not** need to equal the total number of unique keys (e.g., stock tickers).
* **Hashing Mechanism:** Kafka hashes the message key and applies a modulo operation to determine the target partition:
  $$\text{partition} = \text{hash}(\text{key}) \pmod{\text{total\_partitions}}$$
* **Many-to-One Mapping:** Thousands or millions of unique keys can be cleanly distributed across a much smaller, manageable set of partitions (e.g., 10,000 tickers mapped across 100 partitions).

```
10,000 Tickers
     │
     ▼ hash(key) % 100
┌──────────┐   ┌──────────┐   ┌──────────┐         ┌──────────┐
│   P00    │   │   P01    │   │   P02    │   ...   │   P99    │
│ (~100)   │   │ (~100)   │   │ (~100)   │         │ (~100)   │
└──────────┘   └──────────┘   └──────────┘         └──────────┘
  AAPL, IBM      MSFT, AMD        NVDA              TSLA, META
```

---

## 2. Ordering Guarantees & Co-Location

* **Per-Partition Ordering:** Kafka strictly guarantees message ordering **within a single partition**, but provides no global ordering guarantee across different partitions.
* **Key Consistency:** Every message with the exact same key (e.g., `key = "AAPL"`) will always be routed to the exact same partition.
* **Mixed Keys in a Partition:** Multiple distinct keys can land on the same partition without compromising their individual order.

### Example Stream on Partition 2:

```
Partition 2 Log:
[AAPL $230.10] ──> [IBM $198.20] ──> [AAPL $230.15] ──> [IBM $198.25] ──> [AAPL $230.12]
```

* **AAPL Sequence:** `$230.10` $\rightarrow$ `$230.15` $\rightarrow$ `$230.12` *(Strictly Preserved)*
* **IBM Sequence:** `$198.20` $\rightarrow$ `$198.25` *(Strictly Preserved)*
* **Cross-Ticker Order:** Interleaved based on log position; no specific cross-key sequencing is enforced.

---

## 3. Hot Partitions & Sizing Rules

Partition count should **never** be based on the number of unique keys. Instead, base it on system performance metrics:

1. **Traffic Volume & Event Throughput:** Total messages/second and data rate (MB/sec).
2. **Hot Partition Risk:** High-volume keys (e.g., `AAPL` producing 500,000 events/sec while all other 9,999 stocks produce 500,000 events/sec combined) create a "hot partition" that bottlenecks consumer threads.
3. **Consumer Parallelism:** The maximum number of parallel consumers in a consumer group cannot exceed the total number of partitions.

---

## 4. Architectural Decision Guide

| System Requirement | Recommended Key Strategy | Architectural Impact |
| :--- | :--- | :--- |
| **Per-Ticker Processing** *(e.g., per-stock order books, price tracking)* | `key = ticker` | **Optimal Scalability:** High throughput, parallel consumption across nodes, per-stock sequential order guaranteed. |
| **Global Market Replay** *(e.g., exact sequence of all market events as received)* | Single partition or alternative timestamp keying | **Limited Scalability:** Bottlenecks ingestion into a single thread to preserve global arrival order. |
