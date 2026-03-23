---
title: Panda Index — 避免死锁的唯一键索引
---

# Panda Index — 避免死锁的唯一键索引

PolarDB-X 存储引擎提供的新一代多版本唯一键索引。通过原生 MVCC 能力，避免查询路径上跨索引访问的额外开销，并将全局区间约束检测优化为行级锁粒度，有效规避 Gap 锁导致的死锁和锁等待问题。

## 前提条件

- **实例系列**：标准版或企业版均支持。
- **引擎版本**：MySQL 8.0。
- **存储节点版本**：xcluster8.4.20-20250527 及以上（2025-05-27 发布的版本及之后）。
- **没有实例？** 可通过 `polardbx-zero` skill 创建免费临时实例（2C4G 标准版）。

## 核心优势

- **消除不必要的 Gap 锁**：在 RC 隔离级别下，普通唯一键的"先删后插"等操作会产生 Gap 锁，阻塞不冲突的并发写入。Panda Index 将约束检测优化为行级锁粒度，避免此问题。
- **减少死锁**：Gap 锁范围扩散是 MySQL 唯一键死锁的主要原因，Panda Index 从根本上消除此类场景。
- **无额外性能开销**：benchmark 测试中性能均优于普通唯一键。唯一代价是每条唯一索引记录增加 28 字节（存储空间增长通常低于 5%）。

## 开启方式

`opt_index_format_panda_enabled` 支持 session 级别和 global 级别设置：

```sql
-- Session 级别：仅当前连接生效
SET opt_index_format_panda_enabled = ON;

-- Global 级别：对所有新连接生效
SET GLOBAL opt_index_format_panda_enabled = ON;
```

也可以在控制台 **配置与管理** > **参数设置** > **存储层** 中修改，无需重启实例，即时生效。

- 2025-06-04 之后新购实例默认开启。
- 开启后，新建表或新建唯一键索引时自动使用 Panda Index 格式。
- 关闭后，新建唯一键恢复为 MySQL 社区标准格式。

## 存量索引升级

已存在的唯一键不会自动转换为 Panda Index，需手动重建索引：

```sql
-- 1. 创建新的 Panda Index 唯一键
ALTER TABLE t1 ADD UNIQUE INDEX uk_new(c1);

-- 2. 更换索引名
ALTER TABLE t1 RENAME INDEX uk TO uk_old, RENAME INDEX uk_new TO uk;

-- 3. 删除原索引
ALTER TABLE t1 DROP INDEX uk_old;
```

## 使用示例

### 数据准备

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

### 并发场景对比

Session 1（先删后插）：

```sql
BEGIN;
DELETE FROM t1 WHERE id = 1;
INSERT INTO t1 VALUES (2, 1);
-- 事务未提交
```

Session 2（插入不冲突的数据）：

```sql
INSERT INTO t1 VALUES (3, 2);
```

**普通唯一键**：Session 2 被 Gap 锁阻塞，等待超时失败。

```sql
-- 查看锁信息，可以看到 uk1 上的 GAP 锁
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

**Panda Index**：Session 2 立即成功，无 Gap 锁。

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

## 注意事项

- **隔离级别限制**：Panda Index 主要优化 RC 隔离级别下的 Gap 锁问题。在 RR 隔离级别下，为保证可重复读语义，系统仍使用 Next-Key 锁（记录锁 + Gap 锁），Panda Index 无法避免此级别下的 Gap 锁。
- **不适用类型**：临时表、系统表、压缩表、多值索引不支持 Panda Index。
- **版本不可降级**：使用了 Panda Index 的实例无法降级至不支持 Panda Index 的老版本。
- **参数开关无风险**：开启或关闭 `opt_index_format_panda_enabled` 均无需重启实例，对现有业务无影响。

## 常见问题

**Q：升级为 Panda Index 后，为什么仍然出现涉及唯一键 Gap 锁的死锁？**

检查隔离级别是否为 RR。Panda Index 消除的是 RC 隔离级别下为保证唯一键约束而加的 Gap 锁（参见 MySQL Bug #68021）。RR 隔离级别下的 Next-Key 锁模式无法避免。

**Q：所有表类型都支持吗？**

标准版和企业版均支持。企业版中分区表、单表、广播表，DRDS 模式和 AUTO 模式的数据库均支持。


