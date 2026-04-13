---
title: Lizard Transaction System
---

# Lizard Transaction System

Lizard is the SCN-based transaction system in PolarDB-X Standard Edition, replacing InnoDB's native transaction visibility mechanism with a high-performance MVCC implementation.

## Prerequisites

- PolarDB-X 2.0 Standard Edition or Enterprise Edition.
- Engine version: MySQL 8.0.

## How It Works

Lizard introduces **SCN (System Commit Number)** to express transaction commit order, and **Transaction Slots** to persist the commit version (SCN) for each transaction.

### Write Transaction

1. On transaction start, allocate a Transaction Slot; record its address as **UBA**.
2. During the transaction, fill modified records with `(SCN=NULL, UBA)`.
3. On commit, obtain a commit SCN, write it back to the Transaction Slot, finalize transaction state, and return success to the client.

### Read Transaction

1. On query start, obtain the current SCN from the SCN generator as the query's **Vision**.
2. During the query, follow each record's UBA to its Transaction Slot to get the transaction state and commit SCN.
3. Compare the record's SCN with the Vision SCN to determine visibility.

## Advantages over InnoDB

- **No global structure contention**: Eliminates read-write conflict on shared data structures, significantly reducing contention.
- **Lightweight Vision**: The read view is a single SCN number instead of an active transaction ID array, making it easy to propagate.
- **FlashBack Query**: Supports custom point-in-time queries using SCN.

## Performance Optimization

Since the commit only writes to the Transaction Slot (row records keep SCN=NULL), every visibility check requires a UBA lookup. Two Cleanout mechanisms mitigate this overhead:

### Commit Cleanout

During commit, the system collects a subset of modified records and backfills their SCN based on system load, keeping commit latency low.

### Delayed Cleanout

During queries, after looking up a Transaction Slot and finding the transaction committed, the system opportunistically backfills the row's SCN so future queries can skip the slot lookup.

### Transaction Slot Reuse

Transaction Slots are recycled via a free_list. A cache layer avoids the overhead of traversing multiple data pages during allocation.

## Performance Benchmark

Sysbench Read Write, 512 concurrency, Intel 8269CY 104C, 16M rows:

| System | QPS | TPS | 95% Latency (ms) |
|--------|-----|-----|-------------------|
| Lizard | 636,087 | 31,804 | 16.07 |
| MySQL 8.0.32 | 487,579 | 24,379 | 34.33 |
| MySQL 8.0.18 | 311,400 | 15,577 | 41.23 |

Compared to MySQL 8.0.32: **30% higher throughput, 53% lower latency**.
