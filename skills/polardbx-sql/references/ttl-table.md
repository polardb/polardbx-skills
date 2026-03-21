---
title: PolarDB-X TTL 表与冷数据归档
---

# PolarDB-X TTL 表与冷数据归档

TTL（Time-to-Live）表是 PolarDB-X 提供的自动数据生命周期管理能力。基于时间列定义数据过期规则，系统定期自动清理过期数据，可选将冷数据归档到低成本存储（OSS）。

## TTL 定义

TTL 定义由三部分组成：

### TTL_EXPR（必需）

定义数据的存活时间条件：

```
TTL_EXPR = `时间列` EXPIRE AFTER 过期间隔 TIMEZONE '时区'
```

过期间隔支持：`N YEAR` / `N MONTH` / `N DAY`。

### TTL_JOB（可选）

定义清理任务的调度计划：

```
TTL_JOB = CRON 'cron表达式' TIMEZONE '时区'
```

默认为每天凌晨一点执行。

### 归档表（可选）

冷数据归档到 OSS 的归档表，详见冷数据归档文档。

## 创建 TTL 表

```sql
CREATE TABLE t_order (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT,
  amount DECIMAL(10,2),
  gmt_modified DATETIME NOT NULL
) PARTITION BY KEY(id) PARTITIONS 16
TTL = TTL_DEFINITION (
  TTL_EXPR = `gmt_modified` EXPIRE AFTER 3 MONTH TIMEZONE '+08:00'
);
```

指定自定义清理调度：

```sql
CREATE TABLE t_log (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  log_content TEXT,
  created_at DATETIME NOT NULL
) PARTITION BY KEY(id) PARTITIONS 8
TTL = TTL_DEFINITION (
  TTL_EXPR = `created_at` EXPIRE AFTER 30 DAY TIMEZONE '+08:00'
  TTL_JOB = CRON '0 0 2 * * ?' TIMEZONE '+08:00'
);
```

## TTL 与 CCI 配合

CCI（列存索引）可以与 TTL 表结合，实现冷热数据分离：
- **热数据**：在行存分区表中，支持高性能 OLTP 操作。
- **冷数据**：通过 CCI 归档到列存（对象存储），降低存储成本，保留分析查询能力。

```sql
CREATE TABLE t_order (
  id BIGINT AUTO_INCREMENT PRIMARY KEY,
  user_id BIGINT,
  amount DECIMAL(10,2),
  gmt_modified DATETIME NOT NULL,
  CLUSTERED COLUMNAR INDEX cci_user(user_id) PARTITION BY KEY(user_id) PARTITIONS 16
) PARTITION BY KEY(id) PARTITIONS 16
TTL = TTL_DEFINITION (
  TTL_EXPR = `gmt_modified` EXPIRE AFTER 6 MONTH TIMEZONE '+08:00'
);
```

## 管理 TTL 定义

```sql
-- 查看表的 TTL 定义
SHOW CREATE TABLE t_order;

-- 修改 TTL 过期时间
ALTER TABLE t_order
  MODIFY TTL SET TTL_EXPR = `gmt_modified` EXPIRE AFTER 6 MONTH TIMEZONE '+08:00';

-- 移除 TTL 定义（如果已有归档表，需先删除归档表）
ALTER TABLE t_order REMOVE TTL;
```

## 限制条件

- TTL 时间列必须为 `DATE` / `DATETIME` / `TIMESTAMP` 类型。
- 删除 TTL 定义前，如果存在关联的归档表，需先删除归档表。
- TTL 清理任务是后台异步执行的，数据不会在过期瞬间立刻删除。
