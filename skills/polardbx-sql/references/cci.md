---
title: PolarDB-X 列存索引（CCI）
---

# PolarDB-X 列存索引（CCI）

列存索引（Clustered Columnar Index, CCI）是 PolarDB-X 企业版提供的行列混合存储能力。CCI 本质上是一个基于对象存储的列式聚簇索引，默认冗余主表全部列的数据，以列存格式存储，用于加速 OLAP 分析查询。

## 适用场景

- 宽表多列聚合查询（SUM/COUNT/AVG 等）。
- 复杂报表和数据分析。
- 需要同时支持 OLTP 和 OLAP 的混合负载（HTAP）场景。
- 配合 TTL 表实现冷热数据分离。

## 创建语法

### 建表时创建

```sql
CREATE TABLE t_order (
  order_id BIGINT PRIMARY KEY,
  buyer_id BIGINT,
  seller_id BIGINT,
  amount DECIMAL(10,2),
  create_time DATETIME,
  CLUSTERED COLUMNAR INDEX cci_seller(seller_id)
    PARTITION BY KEY(seller_id) PARTITIONS 16
) PARTITION BY KEY(order_id) PARTITIONS 16;
```

### 在已有表上添加

```sql
-- 使用 CREATE INDEX
CREATE CLUSTERED COLUMNAR INDEX cci_buyer
  ON t_order(buyer_id)
  PARTITION BY KEY(buyer_id) PARTITIONS 16;

-- 使用 ALTER TABLE
ALTER TABLE t_order ADD CLUSTERED COLUMNAR INDEX cci_buyer(buyer_id)
  PARTITION BY KEY(buyer_id) PARTITIONS 16;
```

## 查询使用

PolarDB-X 优化器可以自动选择使用 CCI 执行分析查询，也可以手动指定：

```sql
-- 自动选择（优化器根据代价模型决定）
SELECT seller_id, SUM(amount) FROM t_order
GROUP BY seller_id ORDER BY SUM(amount) DESC LIMIT 10;

-- 手动指定 CCI
SELECT /*+TDDL:FORCE_INDEX(t_order, cci_seller)*/ seller_id, SUM(amount)
FROM t_order GROUP BY seller_id;

-- 使用 FORCE INDEX
SELECT seller_id, SUM(amount) FROM t_order FORCE INDEX(cci_seller)
GROUP BY seller_id;
```

## 查看 CCI 信息

```sql
SHOW COLUMNAR INDEX;
```

## 与 GSI 的关系

CCI 本质上是列存版本的 Clustered GSI：
- **Clustered GSI**：行存格式，适合点查和小范围扫描。
- **CCI**：列存格式，适合大范围扫描和聚合分析。

两者都默认冗余主表全部列，但存储格式和适用查询类型不同。

## 与 TTL 表配合

CCI 可以与 TTL 表结合，实现冷热数据分离：热数据在行存分区表中，冷数据归档到列存 CCI（基于对象存储），降低存储成本的同时保留分析查询能力。

## 限制条件

- 仅 PolarDB-X **企业版（分布式版）** 支持。
- CCI 数据存储在对象存储上，写入存在一定延迟（最终一致）。
- 创建 CCI 为在线操作，不阻塞 DML。
