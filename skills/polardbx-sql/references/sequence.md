---
title: PolarDB-X Sequence 序列
---

# PolarDB-X Sequence 序列

PolarDB-X 中的 Sequence 用于生成全局唯一的数字序列，主要用于主键或唯一索引列。在分布式场景下，Sequence 替代 MySQL 的 AUTO_INCREMENT 提供全局唯一性保证。

## Sequence 类型

### NEW SEQUENCE（默认，推荐）

5.4.14 版本起引入，是当前**默认的 Sequence 类型**。核心特点：

- **连续自增**：在单个连接内生成的值严格连续递增，跨连接时也尽量保持递增趋势（与 GROUP SEQUENCE 的批量分段分配不同，不会出现大段跳跃）。
- **全局唯一**：保证在分布式环境下生成的值全局唯一。
- **基于持久化存储**：值分配基于底层存储节点，实例重启后能从上次位置继续分配，不会丢失或重复。
- **高性能**：通过内部缓存机制保证分配效率，接近 GROUP SEQUENCE 的性能水平。
- **可自定义**：支持 START WITH、INCREMENT BY、MAXVALUE、CYCLE 等参数（自定义特性需要 5.4.17+）。

```sql
-- 默认创建（等同于 CREATE NEW SEQUENCE）
CREATE SEQUENCE order_seq;

-- 指定参数
CREATE NEW SEQUENCE order_seq
  START WITH 1
  INCREMENT BY 1
  MAXVALUE 9999999999
  NOCYCLE;
```

### GROUP SEQUENCE（旧默认类型）

批量从数据库获取 ID 段缓存在内存中分配，性能高。每个节点独立缓存一段 ID，因此**跨节点分配的值不连续**（例如节点 A 分配 1-10000，节点 B 分配 10001-20000），起始值为 100001。

```sql
CREATE GROUP SEQUENCE user_id_seq;

-- 指定起始值
CREATE GROUP SEQUENCE user_id_seq START WITH 200001;
```

单元化部署时使用 UNIT COUNT / INDEX：

```sql
CREATE GROUP SEQUENCE user_id_seq
  START WITH 100001
  UNIT COUNT 3 INDEX 0;
```

### SIMPLE SEQUENCE

支持自定义步长、最大值和循环，功能已被 NEW SEQUENCE 取代，不推荐新场景使用：

```sql
CREATE SIMPLE SEQUENCE legacy_seq
  START WITH 1000
  INCREMENT BY 2
  MAXVALUE 99999999
  CYCLE;
```

### TIME SEQUENCE

基于时间戳生成唯一 ID，列类型必须为 BIGINT：

```sql
CREATE TIME SEQUENCE ts_seq;
```

## 显式使用 Sequence

### 获取值

```sql
-- 获取单个值
SELECT order_seq.NEXTVAL FROM DUAL;

-- 批量获取 10 个值
SELECT order_seq.NEXTVAL FROM DUAL WHERE COUNT = 10;
```

### 在 INSERT 中使用

```sql
INSERT INTO t_order (order_id, buyer_id, amount)
VALUES (order_seq.NEXTVAL, 12345, 99.90);
```

### 查看所有 Sequence

```sql
SHOW SEQUENCES;
```

## 隐式 Sequence

当表的列定义了 `AUTO_INCREMENT` 时，PolarDB-X 自动关联一个隐式 Sequence。用户无需手动创建和管理。

```sql
CREATE TABLE t_user (
  user_id BIGINT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(64)
) PARTITION BY KEY(user_id) PARTITIONS 16;
```

注意：AUTO_INCREMENT 在分布式场景下保证全局唯一，但**不保证连续递增**。

## 修改和删除 Sequence

```sql
-- 修改参数
ALTER SEQUENCE order_seq START WITH 500000 INCREMENT BY 5;

-- 修改类型（需指定 START WITH）
ALTER SEQUENCE order_seq CHANGE TO SIMPLE START WITH 1000000;

-- 删除
DROP SEQUENCE order_seq;
```

## 限制

- 每个数据库最多 **16384 个 Sequence**。
- NEW SEQUENCE 需要 5.4.14+ 版本；自定义特性需要 5.4.17+。
- GROUP SEQUENCE 起始值为 100001，不支持从 1 开始。
- TIME SEQUENCE 的列必须为 BIGINT 类型。
- 单元化 GROUP SEQUENCE 不支持转换类型。
