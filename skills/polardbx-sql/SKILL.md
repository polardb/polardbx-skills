---
name: polardbx-sql
description: 为 PolarDB-X 2.0 分布式版（企业版）AUTO 模式数据库编写、审查和适配 SQL，正确处理 PolarDB-X 与 MySQL 的差异（分区表、全局二级索引 GSI、列存索引 CCI、Sequence、分布式事务、表组、TTL 表等）。适用于生成需要在 PolarDB-X 上运行的 SQL、将 MySQL SQL 迁移到 PolarDB-X、或调试 PolarDB-X SQL 兼容性问题的场景。
---

# PolarDB-X SQL（MySQL 兼容性聚焦）

目标：生成能在 PolarDB-X 2.0 分布式版（企业版）AUTO 模式数据库上正确运行的 SQL，避免"MySQL 上能跑但 PolarDB-X 上报错"的问题。

## 适用范围

本 skill 仅适用于以下版本和模式：

- **PolarDB-X 2.0 企业版**（又称分布式版） + **AUTO 模式数据库**

不适用于：
- PolarDB-X 1.0（DRDS 1.0）
- PolarDB-X 2.0 标准版
- PolarDB-X 2.0 企业版的 DRDS 模式数据库

AUTO 模式和 DRDS 模式的主要区别：AUTO 模式使用 MySQL 兼容的 `PARTITION BY` 语法定义分区，DRDS 模式使用旧版 `dbpartition/tbpartition` 语法。可通过以下方式确认数据库模式：

```sql
SHOW CREATE DATABASE db_name;
-- 输出中 MODE = 'auto' 表示 AUTO 模式
```

## Workflow（每次使用时遵循）

1. 确认目标引擎和版本：
   - 执行 `SELECT VERSION();` 判断实例类型：
     - 结果含 `TDDL` 且版本号 > 5.4.12（如 `5.7.25-TDDL-5.4.19-20251031`）-> **2.0 企业版（分布式版）**，本 skill 适用。从中解析企业版版本号（如 5.4.19）。
     - 结果含 `TDDL` 且版本号 <= 5.4.12（如 `5.6.29-TDDL-5.4.12-16327949`）-> **DRDS 1.0**，本 skill 不适用。
     - 结果含 `X-Cluster`（如 `8.0.32-X-Cluster-8.4.20-20251017`）-> **2.0 标准版**，100% 兼容 MySQL，本 skill 不适用，直接按 MySQL 语法处理即可。
   - 确认是 2.0 企业版后，执行 `SHOW CREATE DATABASE db_name;` 确认是 AUTO 模式（MODE = 'auto'）。
   - 版本号影响特性可用性（如 NEW SEQUENCE 需要 5.4.14+，CCI 需要较新版本）。
2. 确认表类型需求：
   - 小表或字典表 -> 广播表 `BROADCAST`（全量复制到每个 DN）。
   - 无需分布式的表 -> 单表 `SINGLE`（仅存储在一个 DN）。
   - 其他情况 -> 分区表（默认），选择合适的分区键和分区策略。
3. 生成 SQL 时使用 PolarDB-X 安全默认值：
   - 避免不支持的 MySQL 特性（存储过程/触发器/EVENT/SPATIAL 等）。
   - 使用 `KEY` 或 `HASH` 分区替代 MySQL 的 AUTO_INCREMENT 主键写热点。
   - 需要非分区键查询时，考虑创建全局二级索引（GSI）。
   - 需要全局唯一 ID 时，使用 Sequence 而非依赖 AUTO_INCREMENT 的连续性。
4. 如果用户提供 MySQL SQL，进行兼容性检查：
   - 替换不支持的特性，给出 PolarDB-X 替代方案。
   - 明确标注行为差异和版本要求。
5. SQL 慢或报错时，使用 PolarDB-X 诊断工具：
   - `EXPLAIN` 查看逻辑执行计划。
   - `EXPLAIN EXECUTE` 查看下推到 DN 的物理执行计划。
   - `EXPLAIN SHARDING` 查看分片扫描情况，判断是否存在全分片扫描。
   - `EXPLAIN ANALYZE` 实际执行并收集运行统计。

## 核心差异速查

- **三种表类型**：单表(`SINGLE`)、广播表(`BROADCAST`)、分区表（默认）；根据数据量和访问模式选择。
- **分区表**：支持 KEY/HASH/RANGE/LIST/RANGE COLUMNS/LIST COLUMNS/CO_HASH + 二级分区（49 种组合）。
- **全局二级索引 GSI**：解决非分区键查询导致的全分片扫描问题，支持 GSI / UGSI / Clustered GSI 三种类型。
- **列存索引 CCI**：行列混合存储，通过 `CLUSTERED COLUMNAR INDEX` 加速 OLAP 分析查询。
- **Sequence**：全局唯一序列，默认类型为 `NEW SEQUENCE`（5.4.14+），替代 AUTO_INCREMENT 的分布式方案。
- **分布式事务**：基于 TSO 全局时钟 + MVCC + 2PC，默认强一致；单分片事务自动优化为本地事务。
- **表组**：相同分区规则的表绑定在同一表组，确保 JOIN 计算下推，避免跨分片数据搬运。
- **TTL 表**：基于时间列自动过期和清理冷数据，可配合 CCI 实现冷热数据分离。
- **不支持的 MySQL 特性**：存储过程/触发器/EVENT/SPATIAL/GEOMETRY/LOAD XML/HANDLER 等。
- **不支持 STRAIGHT_JOIN / NATURAL JOIN**：使用标准 JOIN 语法替代。
- **不支持 := 赋值运算符**：将逻辑移到应用层。
- **HAVING/JOIN ON 子句中不支持子查询**：将子查询改写为 JOIN 或 CTE。

## 参考文件（本 skill 内）

- `skills/polardbx-sql/references/create-table.md` - 建表语法、表类型（单表/广播表/分区表）、分区策略、二级分区、分区管理操作。
- `skills/polardbx-sql/references/gsi.md` - 全局二级索引 GSI/UGSI/Clustered GSI 的创建、查询和限制。
- `skills/polardbx-sql/references/cci.md` - 列存索引 CCI 的创建、使用和适用场景。
- `skills/polardbx-sql/references/sequence.md` - Sequence 类型（NEW/GROUP/SIMPLE/TIME）、创建和使用。
- `skills/polardbx-sql/references/transactions.md` - 分布式事务模型、隔离级别和注意事项。
- `skills/polardbx-sql/references/mysql-compatibility-notes.md` - MySQL 与 PolarDB-X 的兼容性差异和开发限制。
- `skills/polardbx-sql/references/explain.md` - EXPLAIN 系列命令的用法和执行计划诊断。
- `skills/polardbx-sql/references/ttl-table.md` - TTL 表定义、冷数据归档和清理调度。
- `skills/polardbx-sql/references/online-ddl.md` - Online DDL 判断、无锁执行策略、长事务检查、DMS 无锁变更等。
