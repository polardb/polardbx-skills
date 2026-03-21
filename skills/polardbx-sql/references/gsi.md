---
title: PolarDB-X 全局二级索引（GSI）
---

# PolarDB-X 全局二级索引（GSI）

全局二级索引（Global Secondary Index, GSI）是 PolarDB-X 中的特殊分区表，冗余了主表部分列的数据，按照指定的分区方式分布在各个存储节点上。GSI 的核心作用是解决**非分区键查询导致的全分片扫描**问题。

## 三种 GSI 类型

### 全局二级索引（GSI）

提供与主表不同的分区方式，索引表只包含索引列、主键列和主表分区键：

```sql
CREATE TABLE t_order (
  order_id BIGINT PRIMARY KEY,
  buyer_id BIGINT,
  seller_id BIGINT,
  order_snapshot TEXT,
  GLOBAL INDEX g_i_buyer(buyer_id) PARTITION BY KEY(buyer_id) PARTITIONS 16
) PARTITION BY KEY(order_id) PARTITIONS 16;
```

### 全局唯一索引（UGSI）

在 GSI 基础上增加全局唯一约束，确保索引键在全表范围内唯一：

```sql
CREATE TABLE t_user (
  user_id BIGINT PRIMARY KEY,
  phone VARCHAR(20),
  name VARCHAR(64),
  UNIQUE GLOBAL INDEX g_i_phone(phone) PARTITION BY KEY(phone) PARTITIONS 16
) PARTITION BY KEY(user_id) PARTITIONS 16;
```

### 全局聚簇索引（Clustered GSI）

默认冗余主表的全部列，避免回表查询，代价是占用与主表相同的存储空间：

```sql
CREATE TABLE t_order (
  order_id BIGINT PRIMARY KEY,
  buyer_id BIGINT,
  seller_id BIGINT,
  order_info TEXT,
  create_time DATETIME,
  CLUSTERED INDEX cg_i_buyer(buyer_id) PARTITION BY KEY(buyer_id) PARTITIONS 16
) PARTITION BY KEY(order_id) PARTITIONS 16;
```

## 创建方式

### 建表时内联创建（推荐）

见上方各类型示例。

### 在已有表上添加

```sql
-- 添加普通 GSI
ALTER TABLE t_order ADD GLOBAL INDEX g_i_seller(seller_id)
  PARTITION BY KEY(seller_id) PARTITIONS 16;

-- 添加全局唯一索引
ALTER TABLE t_order ADD UNIQUE GLOBAL INDEX g_i_order_no(order_no)
  PARTITION BY KEY(order_no) PARTITIONS 16;

-- 添加全局聚簇索引
ALTER TABLE t_order ADD CLUSTERED INDEX cg_i_seller(seller_id)
  PARTITION BY KEY(seller_id) PARTITIONS 16;
```

### 使用 CREATE INDEX 语法

```sql
CREATE GLOBAL INDEX g_i_seller ON t_order(seller_id)
  PARTITION BY KEY(seller_id) PARTITIONS 16;
```

## 覆盖列（COVERING）

通过 `COVERING` 指定额外冗余的列，减少回表开销：

```sql
CREATE TABLE t_order (
  order_id BIGINT PRIMARY KEY,
  buyer_id BIGINT,
  seller_id BIGINT,
  amount DECIMAL(10,2),
  status INT,
  GLOBAL INDEX g_i_buyer(buyer_id) COVERING(amount, status)
    PARTITION BY KEY(buyer_id) PARTITIONS 16
) PARTITION BY KEY(order_id) PARTITIONS 16;
```

索引表自动包含主键列和主表分区键列，无需在 COVERING 中重复指定。

## 查询使用

PolarDB-X 优化器可以自动选择 GSI，也可以手动指定：

```sql
-- 使用 FORCE INDEX
SELECT * FROM t_order FORCE INDEX(g_i_buyer)
WHERE buyer_id = 12345;

-- 使用 HINT
SELECT /*+TDDL:INDEX(t_order, g_i_buyer)*/ *
FROM t_order WHERE buyer_id = 12345;
```

## 限制条件

- 每张表最多 **32 个全局索引**。
- GSI 需要 **XA/TSO 分布式事务**支持。
- 禁止直接对 GSI 索引表执行 DML（INSERT/UPDATE/DELETE）或 DDL。
- 禁止 `TRUNCATE` 含 GSI 的表，用 `DELETE` 清空数据替代。
- 删除包含在 GSI 中的列之前，需先删除对应的 GSI。
- GSI 的写入性能有额外开销（需要同步维护索引表数据一致性）。
- 创建和删除 GSI 为在线操作，不阻塞 DML。

## 设计建议

- 优先将**最高频的非分区键查询条件**设计为 GSI。
- 如果查询需要返回较多列，使用 **Clustered GSI** 避免回表。
- 如果只需要少量额外列，使用 **COVERING** 比 Clustered GSI 更节省空间。
- 如果需要全局唯一约束（如手机号、订单号），使用 **UGSI**。
