---
title: PolarDB-X Online DDL 与无锁 DDL 操作
---

# PolarDB-X Online DDL 与无锁 DDL 操作

PolarDB-X 分布式版针对 DDL 锁表问题进行了大量优化，包括 MDL 锁抢占、MDL 锁双版本、无锁列类型变更等特性。本文档介绍如何判断 DDL 是否锁表、如何以无锁方式执行 DDL，以及执行 DDL 前的长事务检查。

## EXPLAIN ONLINE_DDL — 判断 DDL 是否锁表

在执行 DDL 前，使用 `EXPLAIN ONLINE_DDL` 预判该 DDL 是否会锁表，不会实际执行 DDL。

**版本要求**：实例版本 >= 5.4.20-20241224。

### 语法

```sql
EXPLAIN ONLINE_DDL ALTER TABLE ...
```

### 返回字段

| 字段 | 含义 |
|------|------|
| DDL TYPE | `ONLINE_DDL`（不锁表）或 `LOCK_TABLE`（锁表） |
| ALGORITHM | DDL 将使用的执行算法 |

### DDL TYPE 与 ALGORITHM 对照

| DDL TYPE | ALGORITHM | 说明 | 对业务影响 |
|----------|-----------|------|-----------|
| ONLINE_DDL | INSTANT / META_ONLY / DEFAULT | 仅修改元数据，秒级完成 | 小 |
| ONLINE_DDL | INPLACE / OMC / OSC | 不锁表但耗时与数据量相关，占用部分磁盘/IO/CPU 资源 | 小 |
| LOCK_TABLE | COPY | 锁表，执行期间表不可写入 | 大 |

### 示例

```sql
-- 加字段：INSTANT，秒级完成，不锁表
EXPLAIN ONLINE_DDL ALTER TABLE t1 ADD COLUMN d int;
-- 结果: DDL TYPE = ONLINE_DDL, ALGORITHM = INSTANT

-- 修改列类型：COPY，锁表
EXPLAIN ONLINE_DDL ALTER TABLE t1 MODIFY COLUMN c bigint;
-- 结果: DDL TYPE = LOCK_TABLE, ALGORITHM = COPY

-- 加分区：META_ONLY，秒级完成，不锁表
EXPLAIN ONLINE_DDL ALTER TABLE t1 ADD PARTITION (PARTITION p2 VALUES LESS THAN (2000000));
-- 结果: DDL TYPE = ONLINE_DDL, ALGORITHM = META_ONLY
```

## 无锁执行 DDL 策略

根据 `EXPLAIN ONLINE_DDL` 的结果分两种情况处理：

1. **DDL TYPE = ONLINE_DDL**：直接执行原始 SQL 即可，不会锁表。
2. **DDL TYPE = LOCK_TABLE**：需要指定 `ALGORITHM=OMC` 启用无锁列类型变更功能。

### ALGORITHM=OMC 无锁列类型变更

对于会锁表的 ALTER TABLE 操作（如修改列类型），在 SQL 中指定 `ALGORITHM=OMC` 可避免锁表：

```sql
-- 原始 SQL 会锁表
EXPLAIN ONLINE_DDL ALTER TABLE t1 MODIFY COLUMN b text;
-- 结果: DDL TYPE = LOCK_TABLE, ALGORITHM = COPY

-- 指定 OMC 后不锁表
EXPLAIN ONLINE_DDL ALTER TABLE t1 MODIFY COLUMN b text, ALGORITHM=OMC;
-- 结果: DDL TYPE = ONLINE_DDL, ALGORITHM = OMC

-- 确认无锁后执行
ALTER TABLE t1 MODIFY COLUMN b text, ALGORITHM=OMC;
```

**注意**：OMC 执行速度较慢、资源消耗更高，仅在需要避免锁表时使用。优先使用原生 Online DDL。

## 执行 DDL 前检查长事务

即使 DDL 本身不锁表，如果目标表上存在**未提交的长事务或大查询**，DDL 仍可能引发问题。

### PolarDB-X 的 MDL 优化

PolarDB-X 对 MDL 锁进行了两项关键优化：

1. **抢占式 MDL 锁**：保证 DDL 在确定时间范围内（默认 15s）一定能获取到 MDL 锁，解决 DDL 长时间无法执行的问题。
2. **双版本 MDL 锁**：引入双版本元数据机制，新事务访问新的元数据，避免新事务被阻塞。

**副作用**：默认情况下，超过 15 秒的长事务或大查询所在连接会被 Kill。如果有重要的数据同步任务（DataWorks/DTS/mysqldump 等），应避免在任务执行期间进行 DDL 操作。

### 通过 POLARDBX_TRX 视图检查长事务

查询事务持续时间超过 15 秒的所有事务：

```sql
SELECT
    TRX_ID AS '事务ID',
    PROCESS_ID AS '连接ID',
    SCHEMA AS '库名',
    START_TIME AS '事务开始时间',
    ROUND(DURATION_TIME / 1000 / 1000, 3) AS '事务持续时间(秒)',
    ROUND(ACTIVE_TIME / 1000 / 1000, 3) AS '事务活跃时间(秒)',
    ROUND(IDLE_TIME / 1000 / 1000, 3) AS '事务空闲时间(秒)',
    SQL AS '当前执行的SQL'
FROM
    INFORMATION_SCHEMA.POLARDBX_TRX
WHERE
    DURATION_TIME > 15 * 1000 * 1000;
```

字段说明：
- **事务持续时间**：从事务开始到当前的总时长。
- **事务活跃时间**：数据库实际处理该事务所花费的总时长。
- **事务空闲时间**：事务中客户端未与数据库交互的时间总和（通常是客户端处理业务逻辑的时间）。

### 通过 METADATA_LOCK 视图检查事务持有的 MDL

定位到长事务后，查看该事务持有哪些表的 MDL 锁：

```sql
SELECT
    LOWER(HEX(TRX_ID)) AS '事务ID',
    CONN_ID AS '连接ID',
    SUBSTRING_INDEX(SUBSTRING_INDEX(`TABLE`, '#', 1), '.', 1) AS '库名',
    SUBSTRING_INDEX(SUBSTRING_INDEX(`TABLE`, '#', 1), '.', -1) AS '表名',
    SUBSTRING_INDEX(FRONTEND, '@', 1) AS '用户名',
    SUBSTRING_INDEX(FRONTEND, '@', -1) AS '客户端IP',
    TYPE AS 'MDL类型'
FROM
    INFORMATION_SCHEMA.METADATA_LOCK
WHERE
    `TABLE` NOT LIKE 'tablegroupid%'
    AND LOWER(HEX(TRX_ID)) = '<事务ID>';
```

### DDL 前长事务处理决策

| 长事务情况 | 建议操作 |
|-----------|---------|
| 长事务不符合业务预期 | 排查业务逻辑，解决后再执行 DDL |
| 长事务符合预期且优先级高 | 暂不执行 DDL，避免影响业务 |
| 长事务符合预期但优先级低 | 可执行 DDL，但 DDL 执行过程中会 Kill 长事务连接 |

## DDL 操作推荐流程

> **重要**：DDL 是高风险操作，在实际执行任何 DDL 语句前，**必须先将 DDL 语句、EXPLAIN ONLINE_DDL 结果、长事务检查结果等信息展示给用户，获得用户明确确认后再执行**。禁止在未经用户确认的情况下直接执行 DDL。

```
1. EXPLAIN ONLINE_DDL ALTER TABLE ...
   |
   |-- ONLINE_DDL -> 直接执行
   |-- LOCK_TABLE -> 改写为 ALGORITHM=OMC 后重新 EXPLAIN 确认
   |
2. 检查目标表上的长事务（POLARDBX_TRX + METADATA_LOCK）
   |
3. 将以上信息展示给用户，获得用户明确确认
   |
4. 确认安全后执行 DDL
```

## 查看 DDL 执行进度

对于耗时较长的 DDL 操作（如 OMC、INPLACE、OSC 等涉及数据回填的操作），可通过 `INFORMATION_SCHEMA.DDL_PROGRESS` 视图实时查看执行进度：

```sql
SELECT * FROM INFORMATION_SCHEMA.DDL_PROGRESS;
```

| 字段 | 说明 |
|------|------|
| JOB_ID | DDL 任务 ID |
| BACKFILL_ID | 数据回填任务 ID |
| TABLE_SCHEMA | 库名 |
| TABLE_NAME | 表名 |
| STATE | 当前执行状态 |
| PROGRESS | 执行进度百分比 |
| FINISHED_ROWS | 已完成的行数 |
| APPROXIMATE_TOTAL_ROWS | 预估总行数 |
| CURRENT_SPEED | 当前执行速度（行/秒） |
| AVERAGE_SPEED | 平均执行速度（行/秒） |
| CHECK_PROGRESS | 校验进度 |
| START_TIME | 开始时间 |
| UPDATE_TIME | 最近更新时间 |
| DDL_STMT | DDL 语句 |

当用户执行了耗时较长的 DDL 并询问进度时，使用此视图查看。

## DMS 无锁变更

PolarDB-X 的无锁列类型变更功能已集成到 DMS（数据管理服务）的无锁变更模块。使用 DMS 时，系统自动判断 DDL 是否存在锁表风险并智能选择最优执行策略。

- 入口：DMS 控制台 > 数据库开发 > 数据变更 > 无锁变更
- 开启后，普通数据变更工单和无锁变更工单均优先以无锁方式执行
- 版本要求：实例版本 >= 5.4.20-20241224

## 低版本兼容方案

对于实例版本低于 5.4.20-20241224 的场景：

- 参考官方文档 Online DDL 章节判断 DDL 是否锁表。
- 对于 ALTER TABLE 类型，可在语句末尾加 `LOCK=NONE` 测试：
  - 能正常执行 -> 该操作是 Online 的，不锁表。
  - 返回报错 -> 该操作不支持 Online 执行，会锁表。
- 建议在测试实例或测试表上验证。

## 常见问题

**Q：为什么不将 OMC 作为 ALTER TABLE 的默认执行策略？**

OMC 虽然能避免锁表，但执行速度较慢且资源消耗更高。大多数场景下优先使用 MySQL 原生 Online DDL，仅在需要避免锁表时选择 OMC。

**Q：如何加速 DDL 执行？**

PolarDB-X 支持并行 DDL 功能。在硬件资源空闲时，可通过调整 DDL 并发度来加速执行，缩短变更窗口期。
