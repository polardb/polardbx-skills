---
title: Verify Panda Index Eliminates Gap Locks
---

# Verify Panda Index Eliminates Gap Locks

End-to-end verification: controlled experiment proving Panda Index eliminates Gap locks on unique keys under RC isolation level.

See `skills/polardbx-standard/references/panda-index.md` for feature details.

## Step 1: Obtain Instance

If the user already provided connection info, use it directly. Otherwise, create a temporary instance via PolarDB-X Zero:

```bash
curl -s -X POST https://zero.polardbx.com/api/v1/instances \
  -H 'Content-Type: application/json' \
  -d '{"tag": "panda-index-test", "ttlMinutes": 60}'
```

Extract `host`, `port`, `username`, `password` from the response.

## Step 2: Connect and Verify Version

```bash
mysql -h <host> -P <port> -u <username> -p<password> -e "SELECT VERSION();"
```

Confirm output contains `X-Cluster` and version >= `xcluster8.4.20-20250527`. If not met, abort and inform the user.

## Step 3: Disable Panda Index, Create Baseline Table with Regular Unique Key

Create a table with regular unique key as the control group:

```sql
SET opt_index_format_panda_enabled = OFF;

CREATE DATABASE IF NOT EXISTS panda_test;
USE panda_test;

DROP TABLE IF EXISTS t_normal;
CREATE TABLE t_normal(
  id int,
  c1 int,
  PRIMARY KEY(id),
  UNIQUE KEY uk1(c1)
);

INSERT INTO t_normal VALUES (1, 1);
INSERT INTO t_normal VALUES (100, 100);
```

## Step 4: Control Test — Regular Unique Key Has Gap Locks

```sql
BEGIN;
DELETE FROM t_normal WHERE id = 1;
INSERT INTO t_normal VALUES (2, 1);

-- Check locks on uk1, expect S,GAP
SELECT lock_data, lock_mode
FROM performance_schema.data_locks
WHERE object_name = 't_normal' AND index_name = 'uk1';

ROLLBACK;
```

**Expected**: `lock_mode` column contains `S,GAP`, proving regular unique keys produce Gap locks on delete-then-insert under RC isolation.

## Step 5: Enable Panda Index, Create Panda Unique Key Table

```sql
SET opt_index_format_panda_enabled = ON;

DROP TABLE IF EXISTS t_panda;
CREATE TABLE t_panda(
  id int,
  c1 int,
  PRIMARY KEY(id),
  UNIQUE KEY uk1(c1)
);

INSERT INTO t_panda VALUES (1, 1);
INSERT INTO t_panda VALUES (100, 100);
```

## Step 6: Experiment Test — Panda Index Has No Gap Locks

```sql
BEGIN;
DELETE FROM t_panda WHERE id = 1;
INSERT INTO t_panda VALUES (2, 1);

-- Check locks on uk1, expect only REC_NOT_GAP, no GAP locks
SELECT lock_data, lock_mode
FROM performance_schema.data_locks
WHERE object_name = 't_panda' AND index_name = 'uk1';

ROLLBACK;
```

**Expected**: `lock_mode` column contains only `X,REC_NOT_GAP`, no `S,GAP`. If `S,GAP` still appears, Panda Index is not active — check parameter and version.

## Step 7: Cleanup

```sql
DROP DATABASE IF EXISTS panda_test;
```

If the instance was temporarily created and is no longer needed:

```bash
curl -s -X DELETE https://zero.polardbx.com/api/v1/instances/<instance_id>
```

## Pass/Fail Criteria

- **Pass**: Step 4 shows `S,GAP` (baseline confirmed) AND Step 6 shows no `GAP` locks (Panda Index effective).
- **Fail**: Step 4 missing `S,GAP` (baseline invalid), OR Step 6 shows `S,GAP` (Panda Index ineffective), OR version prerequisite not met.

## Step 8: Output Test Report

After all steps are completed, output the following Markdown report directly to the user (do not write to file). Fill `<>` placeholders with actual execution results:

~~~markdown
# Panda Index 测试报告

- 测试时间：<执行时的时间戳>
- 实例版本：<SELECT VERSION() 的实际输出>
- 隔离级别：RC

## Panda Index 简介

Panda Index 是 PolarDB-X 存储引擎提供的多版本唯一键索引。

MySQL 的普通唯一键在执行"先删后插"等操作时，需要通过 Gap 锁（区间锁）来保证唯一约束不被并发事务破坏。这种 Gap 锁会锁住一个范围而非单行，导致不冲突的并发写入也被阻塞，甚至引发死锁（参见 MySQL Bug #68021）。

Panda Index 在索引记录中内置了 MVCC 版本信息，使引擎可以通过行级版本判断来完成唯一约束检测，无需依赖 Gap 锁锁定区间。这样将全局区间约束检测优化为行级锁粒度，从根本上消除了 RC 隔离级别下唯一键相关的 Gap 锁和由此产生的死锁问题。

## 表结构

### t_normal（普通唯一键）

```sql
<Step 3 中 CREATE TABLE t_normal 的完整语句>
```

`opt_index_format_panda_enabled = OFF`

### t_panda（Panda Index 唯一键）

```sql
<Step 5 中 CREATE TABLE t_panda 的完整语句>
```

`opt_index_format_panda_enabled = ON`

## 测试语句与时序

以下操作在同一个 Session 内按顺序执行（分别对 t_normal 和 t_panda 各执行一轮）：

| 序号 | SQL 语句 | 说明 |
|------|----------|------|
| 1 | `BEGIN;` | 开启事务 |
| 2 | `DELETE FROM <table> WHERE id = 1;` | 删除 id=1 的记录 |
| 3 | `INSERT INTO <table> VALUES (2, 1);` | 插入新记录，复用已删除的唯一键值 c1=1 |
| 4 | `SELECT lock_data, lock_mode FROM performance_schema.data_locks WHERE object_name = '<table>' AND index_name = 'uk1';` | 查看 uk1 上的锁信息 |
| 5 | `ROLLBACK;` | 回滚事务 |

## 测试结果

### 反向验证：t_normal（普通唯一键）

<Step 4 中 SELECT 查询返回的完整结果表格>

结论：<是否出现 S,GAP>

### 正向验证：t_panda（Panda Index）

<Step 6 中 SELECT 查询返回的完整结果表格>

结论：<是否消除 S,GAP>

## 最终结论

<通过 / 失败，以及一句话总结>
~~~
