---
title: Panda Index — Deadlock-Free Unique Key Index
---

# Panda Index — Deadlock-Free Unique Key Index

A next-generation MVCC-based unique key index provided by PolarDB-X storage engine. It optimizes global range constraint checks to row-level lock granularity, effectively eliminating Gap locks and the resulting deadlocks/lock waits under RC isolation level.

## Prerequisites

- **Edition**: Standard Edition or Enterprise Edition.
- **Engine version**: MySQL 8.0.
- **Storage node version**: xcluster8.4.20-20250527 or later.
- **No instance?** Use `polardbx-zero` skill to create a free temporary instance (2C4G Standard Edition).

## Key Benefits

- **Eliminates unnecessary Gap locks**: Under RC isolation, regular unique keys produce Gap locks on delete-then-insert operations, blocking non-conflicting concurrent writes. Panda Index reduces constraint checks to row-level lock granularity.
- **Reduces deadlocks**: Gap lock range expansion is the primary cause of MySQL unique key deadlocks. Panda Index eliminates this class of deadlocks.
- **No performance overhead**: Benchmarks show equal or better performance than regular unique keys. The only cost is 28 extra bytes per unique index record (storage growth typically < 5%).

## How to Enable

`opt_index_format_panda_enabled` supports session-level and global-level settings:

```sql
-- Session level: current connection only
SET opt_index_format_panda_enabled = ON;

-- Global level: all new connections
SET GLOBAL opt_index_format_panda_enabled = ON;
```

Can also be set via console: **Configuration & Management** > **Parameter Settings** > **Storage Layer**. No restart required, takes effect immediately.

- Enabled by default for instances created after 2025-06-04.
- When ON, new tables and new unique indexes automatically use Panda Index format.
- When OFF, new unique indexes revert to standard MySQL format.

## Upgrading Existing Indexes

Existing unique keys are not automatically converted. Rebuild manually:

```sql
-- 1. Create new Panda Index unique key
ALTER TABLE t1 ADD UNIQUE INDEX uk_new(c1);

-- 2. Rename indexes
ALTER TABLE t1 RENAME INDEX uk TO uk_old, RENAME INDEX uk_new TO uk;

-- 3. Drop old index
ALTER TABLE t1 DROP INDEX uk_old;
```

## Usage Example

### Data Setup

```sql
CREATE TABLE t1(
  id int,
  c1 int,
  PRIMARY KEY(id),
  UNIQUE KEY uk1(c1)
);

INSERT INTO t1 VALUES (1,1);
INSERT INTO t1 VALUES (100,100);
```

### Concurrency Comparison

Session 1 (delete-then-insert):

```sql
BEGIN;
DELETE FROM t1 WHERE id = 1;
INSERT INTO t1 VALUES (2, 1);
-- Transaction not committed
```

Session 2 (insert non-conflicting data):

```sql
INSERT INTO t1 VALUES (3, 2);
```

**Regular unique key**: Session 2 blocked by Gap lock, times out.

```sql
SELECT lock_data, lock_mode
FROM performance_schema.data_locks
WHERE index_name = 'uk1';

+-----------+---------------+
| lock_data | lock_mode     |
+-----------+---------------+
| 1, 1      | X,REC_NOT_GAP |
| 1, 1      | S,GAP         |
| 100, 100  | S,GAP         |
| 1, 2      | S,GAP         |
+-----------+---------------+
```

**Panda Index**: Session 2 succeeds immediately, no Gap locks.

```sql
SELECT lock_data, lock_mode
FROM performance_schema.data_locks
WHERE index_name = 'uk1';

+-----------+---------------+
| lock_data | lock_mode     |
+-----------+---------------+
| 1, 2      | X,REC_NOT_GAP |
+-----------+---------------+
```

## Notes

- **Isolation level limitation**: Panda Index optimizes Gap locks under RC isolation only. Under RR isolation, Next-Key locks (record lock + Gap lock) are still used for repeatable-read semantics.
- **Unsupported types**: Temporary tables, system tables, compressed tables, and multi-value indexes do not support Panda Index.
- **No version downgrade**: Instances using Panda Index cannot be downgraded to versions without Panda Index support.
- **Safe to toggle**: Enabling or disabling `opt_index_format_panda_enabled` requires no restart and has no impact on existing workloads.

## FAQ

**Q: After upgrading to Panda Index, why do I still see deadlocks involving unique key Gap locks?**

Check if the isolation level is RR. Panda Index eliminates Gap locks added under RC isolation for unique key constraint enforcement (see MySQL Bug #68021). Next-Key locks under RR isolation cannot be avoided.

**Q: Are all table types supported?**

Both Standard Edition and Enterprise Edition are supported. In Enterprise Edition, partitioned tables, single tables, broadcast tables, and both DRDS mode and AUTO mode databases are all supported.
