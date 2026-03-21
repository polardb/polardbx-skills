---
title: PolarDB-X 与 MySQL 兼容性说明
---

# PolarDB-X 与 MySQL 兼容性说明

PolarDB-X 分布式版（企业版）高度兼容 MySQL 协议和语法，但由于分布式架构的差异，存在部分不支持或行为不同的地方。迁移 MySQL SQL 到 PolarDB-X 时，以此作为检查清单。

## 检测 PolarDB-X 版本

```sql
SELECT VERSION();
```

通过返回值区分实例类型：

| 返回值示例 | 实例类型 | MySQL 兼容性 |
|-----------|---------|-------------|
| `5.7.25-TDDL-5.4.19-20251031` | **2.0 企业版（分布式版）** | 高度兼容，存在本文所列差异 |
| `5.6.29-TDDL-5.4.12-16327949` | **DRDS 1.0**（版本号 <= 5.4.12） | 旧版本，本文不适用 |
| `8.0.32-X-Cluster-8.4.20-20251017` | **2.0 标准版** | 100% 兼容 MySQL，无需关注本文 |

- 含 `TDDL` 且版本号 > 5.4.12 -> 2.0 企业版，版本号取 `TDDL-` 后的部分（如 `5.4.19`）。
- 含 `TDDL` 且版本号 <= 5.4.12 -> DRDS 1.0，本 skill 不适用。
- 含 `X-Cluster` -> 2.0 标准版，直接按 MySQL 语法处理即可。

## 不支持的 MySQL 特性（默认不要生成）

- 存储过程（Stored Procedures）和存储函数（Stored Functions）
- 触发器（Triggers）
- 事件调度器（Events）
- 用户自定义函数（UDF）
- `SPATIAL` / `GEOMETRY` 数据类型、空间函数和空间索引
- `LOAD XML`
- `HANDLER` 语句
- `IMPORT TABLE`
- `INSERT DELAYED`
- `STRAIGHT_JOIN`（使用标准 JOIN 替代）
- `NATURAL JOIN`（使用显式 JOIN ON 替代）
- `:=` 赋值运算符（将逻辑移到应用层）
- XML 函数
- GTID 函数
- 全文检索函数（MySQL 的 FULLTEXT 不可用）
- `ALTER EVENT` / `ALTER INSTANCE` / `ALTER SERVER`
- `CREATE EVENT` / `DROP EVENT`
- `CREATE SERVER` / `DROP SERVER`
- `CREATE SPATIAL REFERENCE SYSTEM`
- `LOCK INSTANCE FOR BACKUP` / `UNLOCK INSTANCE`
- 复制相关语句（`CHANGE MASTER TO`、`START/STOP SLAVE` 等）
- 组复制语句（`START/STOP GROUP_REPLICATION`）
- `INSTALL/UNINSTALL COMPONENT/PLUGIN`

## 部分支持或行为差异

### 子查询限制

- `HAVING` 子句中**不支持子查询**，改写为 JOIN 或 CTE。
- `JOIN ON` 子句中**不支持子查询**，将子查询提取为独立 JOIN。
- 等号操作符的标量子查询正常支持。

### DML 差异

- `ON UPDATE CURRENT_TIMESTAMP` 的行为与 MySQL 不完全一致，建议在应用层显式设置更新时间。
- 变量引用操作（`@c=1, @d=@c+1`）不支持。

### SHOW 命令

- `SHOW WARNINGS` 和 `SHOW ERRORS` 不支持 `LIMIT` 和 `COUNT` 组合。
- `HELP` 命令不支持。

### 关键字限制

- `MILLISECOND` 和 `MICROSECOND` 关键字不支持。

### 数据类型限制

- `JSON` 类型不能做分区键。
- `GEOMETRY` / `LINESTRING` 等空间类型不支持。

## DDL 限制

- 二级分区表暂不支持 Merge/Split/Add/Drop Subpartition。
- 索引分区表暂不支持 Merge/Split/Add/Drop。
- 支持外键。
- 支持 Generated Column。
- 支持 `RENAME TABLE`。

## 标识符限制

| 类型 | 最大字符长度 |
|------|------------|
| Database | 32 |
| Table | 64 |
| Column | 64 |
| Partition | 16 |
| Sequence | 128 |
| View | 64 |
| Constraint | 64 |

## 资源限制

| 资源 | 限制 |
|------|------|
| 每个数据库表数量 | 8192 |
| 每张表列数量 | 1017 |
| 每张表分区数量 | 8192 |
| 每张表全局索引数量 | 32 |
| 每个数据库 Sequence 数量 | 16384 |
| 每个数据库视图数量 | 8192 |
| 每个数据库用户数量 | 2048 |
| 数据库数量 | 32 |

## 字符集和排序规则

- 默认字符集：`utf8mb4`。
- 建议显式指定排序规则，避免依赖默认行为（PolarDB-X 的默认排序规则可能与 MySQL 不同）。
- 如果依赖大小写不敏感比较，显式设置 `utf8mb4_general_ci` 排序规则。

## 兼容的 MySQL 特性（可以放心使用）

- 标准 DML：SELECT / INSERT / UPDATE / DELETE / REPLACE
- 事务：BEGIN / COMMIT / ROLLBACK / SAVEPOINT
- DDL：CREATE/ALTER/DROP TABLE / CREATE/DROP INDEX / CREATE/ALTER/DROP VIEW
- 账户管理：CREATE/ALTER/DROP USER / GRANT / REVOKE
- 预处理语句：PREPARE / EXECUTE / DEALLOCATE PREPARE
- LOAD DATA（默认关闭，需手动开启）
- LOCK TABLES
- SET TRANSACTION
- 大部分 SHOW 命令
