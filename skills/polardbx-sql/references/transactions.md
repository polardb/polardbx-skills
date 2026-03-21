---
title: PolarDB-X 分布式事务
---

# PolarDB-X 分布式事务

PolarDB-X 分布式版基于 TSO（Timestamp Oracle）全局时钟 + MVCC（多版本并发控制）+ 2PC（两阶段提交）实现分布式事务，保证跨分片事务的 ACID 特性。

## 事务模型

- **TSO 全局时钟**：中心授时节点提供全局单调递增的时间戳，用于保证分布式一致性读。
- **MVCC**：多版本并发控制，读取操作基于快照时间戳，不会读到中间状态。
- **2PC**：跨多个 DN 的写入操作通过两阶段提交保障原子性。

## 隔离级别

PolarDB-X 支持以下隔离级别：

- **READ COMMITTED（RC）**：默认隔离级别。每条语句读取最新已提交的数据。
- **REPEATABLE READ（RR）**：事务开始时获取快照，整个事务期间读取一致。

```sql
-- 查看当前隔离级别
SELECT @@transaction_isolation;

-- 设置隔离级别
SET SESSION TRANSACTION ISOLATION LEVEL REPEATABLE READ;
```

## 事务语法

```sql
-- 开启事务
BEGIN;
-- 或
START TRANSACTION;

-- 提交
COMMIT;

-- 回滚
ROLLBACK;

-- SAVEPOINT
SAVEPOINT sp1;
ROLLBACK TO SAVEPOINT sp1;
RELEASE SAVEPOINT sp1;
```

## 单分片事务优化

当事务涉及的所有操作只落在**同一个分片**时，PolarDB-X 自动将其优化为本地事务（1PC），避免 2PC 的额外开销。设计分区表时尽量让同一事务的操作落在同一分片上，可以显著提升性能。

## 与 GSI 的关系

全局二级索引（GSI）的写入操作需要同时更新主表和索引表（可能在不同分片），因此 GSI 的正确运作依赖分布式事务支持（XA/TSO）。

## 注意事项

- **避免长事务**：长事务持有锁的时间长，影响并发性能，同时会阻碍 MVCC 版本回收。
- **跨分片事务开销**：跨分片事务需要 2PC 协调，性能低于单分片事务。设计时尽量让高频事务操作局限在同一分片。
- **大事务限制**：单个事务中修改的数据量过大时，可能触发内存限制。批量数据操作建议分批提交。
