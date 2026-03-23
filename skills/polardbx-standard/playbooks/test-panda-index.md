---
title: 验证 Panda Index 消除 Gap 锁
---

# 验证 Panda Index 消除 Gap 锁

端到端验证 Panda Index 功能：通过对照实验证明 Panda Index 消除了 RC 隔离级别下唯一键的 Gap 锁。

功能详情参见 `skills/polardbx-standard/references/panda-index.md`。

## Step 1：获取实例

如果用户已提供连接信息，直接使用。否则，通过 PolarDB-X Zero 创建临时实例：

```bash
curl -s -X POST https://zero.polardbx.com/api/v1/instances \
  -H 'Content-Type: application/json' \
  -d '{"tag": "panda-index-test", "ttlMinutes": 60}'
```

从响应中提取 `host`、`port`、`username`、`password`。

## Step 2：连接并确认版本

```bash
mysql -h <host> -P <port> -u <username> -p<password> -e "SELECT VERSION();"
```

确认输出包含 `X-Cluster` 且版本 >= `xcluster8.4.20-20250527`。如不满足则中止并告知用户。

## Step 3：关闭 Panda Index，创建普通唯一键基准表

先用普通唯一键建表，作为反向对照：

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

## Step 4：反向验证 -- 普通唯一键存在 Gap 锁

```sql
BEGIN;
DELETE FROM t_normal WHERE id = 1;
INSERT INTO t_normal VALUES (2, 1);

-- 查看 uk1 上的锁，期望出现 S,GAP
SELECT lock_data, lock_mode
FROM performance_schema.data_locks
WHERE object_name = 't_normal' AND index_name = 'uk1';

ROLLBACK;
```

**预期结果**：`lock_mode` 列中出现 `S,GAP`，证明普通唯一键在 RC 隔离级别下"先删后插"会产生 Gap 锁。

## Step 5：开启 Panda Index，创建 Panda 唯一键表

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

## Step 6：正向验证 -- Panda Index 无 Gap 锁

```sql
BEGIN;
DELETE FROM t_panda WHERE id = 1;
INSERT INTO t_panda VALUES (2, 1);

-- 查看 uk1 上的锁，期望只有 REC_NOT_GAP，没有 GAP 锁
SELECT lock_data, lock_mode
FROM performance_schema.data_locks
WHERE object_name = 't_panda' AND index_name = 'uk1';

ROLLBACK;
```

**预期结果**：`lock_mode` 列只包含 `X,REC_NOT_GAP`，不出现 `S,GAP`。如果仍出现 `S,GAP`，说明 Panda Index 未生效，需检查参数和版本。

## Step 7：清理

```sql
DROP DATABASE IF EXISTS panda_test;
```

如果实例是本次临时创建的且不再需要，可释放：

```bash
curl -s -X DELETE https://zero.polardbx.com/api/v1/instances/<instance_id>
```

## 判定标准

- **通过**：Step 4 出现 `S,GAP`（普通唯一键有 Gap 锁）且 Step 6 无 `GAP` 类型锁（Panda Index 消除了 Gap 锁），两者形成对照。
- **失败**：Step 4 未出现 `S,GAP`（基准不成立），或 Step 6 出现 `S,GAP`（Panda Index 未生效），或版本不满足前提条件。

## Step 8：输出测试报告

所有步骤执行完毕后，直接向用户输出以下格式的 Markdown 报告（不写入文件），使用实际执行结果填充 `<>` 占位符：

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
