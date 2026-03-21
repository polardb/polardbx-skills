---
title: PolarDB-X 建表（表类型、分区策略与管理）
---

# PolarDB-X 建表

PolarDB-X 分布式版支持三种表类型：单表、广播表和分区表。分区表是默认类型，数据按分区规则水平拆分到多个存储节点（DN）。

## 三种表类型

### 单表（SINGLE）

数据只存储在一个 DN 上，适用于无需分布式的小表。

```sql
CREATE TABLE config_tbl (
  id BIGINT PRIMARY KEY,
  key_name VARCHAR(64),
  value TEXT
) SINGLE;
```

### 广播表（BROADCAST）

数据全量复制到每个 DN，适用于数据量小但需要频繁 JOIN 的字典表。

```sql
CREATE TABLE region_dict (
  region_id INT PRIMARY KEY,
  region_name VARCHAR(64)
) BROADCAST;
```

### 分区表（默认）

数据按分区规则分布到多个 DN，适用于大数据量的业务表。

## 一级分区类型

### KEY 分区

类似 MySQL 的 KEY 分区，基于列值哈希路由，最常用的分区类型：

```sql
CREATE TABLE t_order (
  order_id BIGINT PRIMARY KEY,
  user_id BIGINT,
  amount DECIMAL(10,2)
) PARTITION BY KEY(order_id) PARTITIONS 16;
```

向量分区键（多列路由）：

```sql
CREATE TABLE t_item (
  order_id BIGINT,
  item_id BIGINT,
  product_name VARCHAR(128),
  PRIMARY KEY (order_id, item_id)
) PARTITION BY KEY(order_id, item_id) PARTITIONS 16;
```

### HASH 分区

支持分区函数，可对列值进行计算后再哈希：

```sql
-- 按年份哈希
CREATE TABLE t_event (
  id BIGINT PRIMARY KEY,
  event_date DATE
) PARTITION BY HASH(YEAR(event_date)) PARTITIONS 8;

-- 按天哈希（使用 TO_DAYS）
CREATE TABLE t_log (
  id BIGINT PRIMARY KEY,
  created_at DATETIME
) PARTITION BY HASH(TO_DAYS(created_at)) PARTITIONS 16;
```

注意：向量分区键不支持使用分区函数；时区敏感的时间列建议使用 `UNIX_TIMESTAMP()`。

### CO_HASH 分区

PolarDB-X 特有的联合哈希分区，多列参与路由，任意一列的等值条件都可以实现分区裁剪：

```sql
CREATE TABLE t_order (
  order_id BIGINT,
  buyer_id BIGINT,
  seller_id BIGINT,
  PRIMARY KEY (order_id)
) PARTITION BY CO_HASH(
  RIGHT(order_id, 4),
  RIGHT(buyer_id, 4)
) PARTITIONS 16;
```

### RANGE / RANGE COLUMNS 分区

按范围分区，适用于时间序列或连续数值数据：

```sql
CREATE TABLE t_sales (
  id BIGINT PRIMARY KEY,
  sale_date DATE,
  amount DECIMAL(10,2)
) PARTITION BY RANGE COLUMNS(sale_date) (
  PARTITION p2023 VALUES LESS THAN ('2024-01-01'),
  PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
  PARTITION p2025 VALUES LESS THAN ('2026-01-01'),
  PARTITION pmax  VALUES LESS THAN MAXVALUE
);
```

### LIST / LIST COLUMNS 分区

按离散值列表分区，适用于枚举类字段：

```sql
CREATE TABLE t_regional_order (
  id BIGINT PRIMARY KEY,
  region VARCHAR(20),
  amount DECIMAL(10,2)
) PARTITION BY LIST COLUMNS(region) (
  PARTITION p_east VALUES IN ('shanghai', 'hangzhou', 'nanjing'),
  PARTITION p_north VALUES IN ('beijing', 'tianjin'),
  PARTITION p_south VALUES IN ('guangzhou', 'shenzhen')
);
```

## 二级分区（SUBPARTITION）

PolarDB-X 支持二级分区，一级分区和二级分区可以自由组合（49 种组合）。

### 模板化二级分区

所有一级分区使用相同的二级分区定义：

```sql
CREATE TABLE t_order_detail (
  id BIGINT PRIMARY KEY,
  order_date DATE,
  user_id BIGINT,
  amount DECIMAL(10,2)
) PARTITION BY RANGE COLUMNS(order_date)
  SUBPARTITION BY KEY(user_id) SUBPARTITIONS 4
(
  PARTITION p2024 VALUES LESS THAN ('2025-01-01'),
  PARTITION p2025 VALUES LESS THAN ('2026-01-01'),
  PARTITION pmax  VALUES LESS THAN MAXVALUE
);
```

### 非模板化二级分区

各一级分区可以有不同数量的二级分区：

```sql
CREATE TABLE t_sales_detail (
  id BIGINT PRIMARY KEY,
  region VARCHAR(20),
  user_id BIGINT
) PARTITION BY LIST COLUMNS(region)
  SUBPARTITION BY KEY(user_id)
(
  PARTITION p_east VALUES IN ('shanghai', 'hangzhou') SUBPARTITIONS 8,
  PARTITION p_north VALUES IN ('beijing', 'tianjin') SUBPARTITIONS 4
);
```

## 分区管理操作

```sql
-- 添加分区（RANGE/LIST 适用）
ALTER TABLE t_sales ADD PARTITION (
  PARTITION p2026 VALUES LESS THAN ('2027-01-01')
);

-- 删除分区
ALTER TABLE t_sales DROP PARTITION p2023;

-- 分裂分区（将一个分区拆为多个）
ALTER TABLE t_order SPLIT PARTITION p0 INTO (
  PARTITION p0a,
  PARTITION p0b
);

-- 合并分区
ALTER TABLE t_order MERGE PARTITIONS p0a, p0b TO p0;

-- 迁移分区到指定 DN
ALTER TABLE t_order MOVE PARTITIONS p0 TO 'dn-1';
```

## 分区键选择原则

- 选择**最高频查询条件**中的列作为分区键，避免全分片扫描。
- 选择**数据分布均匀**的列，避免热点分区。
- 如果有多个高频查询维度，使用 `CO_HASH` 或创建全局二级索引（GSI）。
- 每张表**最多 8192 个分区**。

## 限制

- 分区键不支持 JSON 类型列。
- 分区键不支持 GEOMETRY 类型列。
- 单列 HASH 分区可使用分区函数，向量分区键不可使用。
- 二级分区表暂不支持 SPLIT/MERGE/ADD/DROP SUBPARTITION。
