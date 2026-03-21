---
title: PolarDB-X EXPLAIN 执行计划诊断
---

# PolarDB-X EXPLAIN 执行计划诊断

PolarDB-X 提供丰富的 EXPLAIN 命令变体，用于查看和分析 SQL 的执行计划。与 MySQL 不同，PolarDB-X 的执行计划分为**CN（计算节点）逻辑计划**和**DN（存储节点）物理计划**两层。

## 完整语法

```sql
EXPLAIN {选项} <SQL语句>
```

支持的选项：`LOGICALVIEW` | `LOGIC` | `SIMPLE` | `DETAIL` | `EXECUTE` | `PHYSICAL` | `OPTIMIZER` | `SHARDING` | `COST` | `ANALYZE` | `BASELINE` | `JSON_PLAN`

## 常用选项

### EXPLAIN（默认）

查看 CN 层的逻辑执行计划：

```sql
EXPLAIN SELECT * FROM t_order WHERE buyer_id = 12345;
```

输出中的关键参数：
- `HitCache`：是否命中 PlanCache（true/false）。
- `TemplateId`：查询计划的全局唯一标识符。
- `Source`：计划来源（如 PLAN_CACHE）。
- `WorkloadType`：工作负载类型（如 TP、AP）。

### EXPLAIN EXECUTE

查看下推到 DN 的物理执行计划（类似 MySQL 的 EXPLAIN），快速诊断索引使用情况：

```sql
EXPLAIN EXECUTE SELECT * FROM t_order WHERE buyer_id = 12345;
```

默认聚合所有 DN 的执行计划信息，通过 Extra 列标注差异：
- `Same plan` / `Different plan` 表示各 DN 执行计划是否一致。
- `Scan rows` 显示扫描行数统计。

### EXPLAIN ANALYZE

实际执行 SQL 并收集运行统计信息（注意：会真正执行查询）：

```sql
EXPLAIN ANALYZE SELECT * FROM t_order WHERE buyer_id = 12345;
```

额外输出 `rowCount`、执行时间等运行时信息，用于对比估算行数和实际行数。

### EXPLAIN SHARDING

查看查询在 DN 上的分片扫描情况，判断是否存在全分片扫描：

```sql
EXPLAIN SHARDING SELECT * FROM t_order WHERE buyer_id = 12345;
```

如果发现扫描了所有分片，说明查询条件未命中分区键，考虑添加 GSI。

### EXPLAIN COST

查看各算子的代价估算和 WORKLOAD 类型识别：

```sql
EXPLAIN COST SELECT * FROM t_order WHERE buyer_id = 12345;
```

### EXPLAIN PHYSICAL

查看执行模式、Fragment 依赖关系和并行度：

```sql
EXPLAIN PHYSICAL SELECT * FROM t_order WHERE buyer_id = 12345;
```

## DN 级 EXPLAIN 变体

需要较新版本（polardb-2.5.0_5.4.20+）。

### EXPLAIN DIFF_EXECUTE

仅显示存在差异的 DN 执行计划，快速定位问题 DN：

```sql
EXPLAIN DIFF_EXECUTE SELECT * FROM t_order WHERE buyer_id = 12345;
```

### EXPLAIN ALL_EXECUTE

显示所有 DN 的详细执行计划：

```sql
EXPLAIN ALL_EXECUTE SELECT * FROM t_order WHERE buyer_id = 12345;
```

### EXPLAIN TREE_EXECUTE

以树状结构展示 DN 执行计划：

```sql
EXPLAIN TREE_EXECUTE SELECT * FROM t_order WHERE buyer_id = 12345;
```

### EXPLAIN JSON_EXECUTE

以 JSON 格式输出 DN 优化器信息：

```sql
EXPLAIN JSON_EXECUTE SELECT * FROM t_order WHERE buyer_id = 12345;
```

### EXPLAIN ANALYZE_EXECUTE

实际执行 SQL 并展示 DN 级别的执行统计信息：

```sql
EXPLAIN ANALYZE_EXECUTE SELECT * FROM t_order WHERE buyer_id = 12345;
```

## HINT 辅助

```sql
-- 查看所有物理分表级别的 DN 执行计划
/*+TDDL:EXPLAIN_EXECUTE_PHYTB_LEVEL=2*/
EXPLAIN EXECUTE SELECT * FROM t_order WHERE buyer_id = 12345;
```

## 诊断建议

- 如果 `EXPLAIN` 显示全分片扫描，检查查询条件是否包含分区键或 GSI 键。
- 如果 `EXPLAIN EXECUTE` 显示 `Different plan`，可能是数据倾斜导致部分 DN 选择了不同执行计划。
- 如果估算行数与实际行数差距大，考虑执行 `ANALYZE TABLE` 刷新统计信息。
- 如果 `EXPLAIN SHARDING` 显示扫描过多分片，考虑优化分区策略或添加 GSI。
