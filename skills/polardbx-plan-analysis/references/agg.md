# Agg — Aggregation Operator EXPLAIN Output Reference

Four Agg operators share the same core output logic: group keys + aggregate functions. Differences are limited to display names and a few extra fields.

---

## Input / Output Attributes

| Attribute | Input | Output |
|-----------|-------|--------|
| Columns | Any | Group key columns + aggregate function result columns (completely redefined) |
| Order | SortAgg requires input ordered by group keys; others have no requirement | HashAgg unordered; SortAgg preserves group key order |
| Distribution | Any | May change (data volume significantly reduced) |
| partitionWise | Any | HashAgg may output in Columnar mode |

---

## Output Format and Display Names

| Operator | Display Name | Description |
|----------|-------------|-------------|
| LogicalAggregate | `LogicalAgg` | — |
| HashAgg | `HashAgg` | Displayed as `PartialHashAgg` when `partial=true` (first stage of two-phase aggregation) |
| SortAgg | `SortAgg` | — |
| MysqlAgg | `MysqlAgg` | DN pushed-down aggregation |

```
HashAgg(group="<col1>,<col2>", <agg_out>="AGG_FUNC(...)", partition=...)
PartialHashAgg(group="<col1>", <agg_out>="AGG_FUNC(...)")
SortAgg(group="<col1>,<col2>", <agg_out>="AGG_FUNC(...)")
MysqlAgg(group="<col1>,<col2>", <agg_out>="AGG_FUNC(...)", index="<idx>")
```

---

## Field Descriptions

### group (Conditional Output)

Group key list, output when `groupSet` is non-empty. Column indices are resolved to column names via `RexExplainVisitor.getField(index)`, comma-separated. Not output when there is no GROUP BY (scalar aggregation).

### Aggregate Function Output Columns (Conditional Output, N columns)

Each column in `rowType` after the group keys corresponds to an aggregate function. Key is the output column name, value is the aggregate expression.

Rendering format: `AGG_FUNC([DISTINCT ]col1, col2, ...)[ FILTER $N]`
- Function name: `COUNT`, `SUM`, `MIN`, `MAX`, `AVG`, `GROUP_CONCAT`, etc.
- `DISTINCT`: Output when the aggregate call is marked distinct
- No parameters (e.g., `COUNT(*)`): Displayed as `COUNT()`
- `FILTER $N`: Appended when the aggregate has a filter

`SELECT DISTINCT` scenario has only group keys and no aggregate functions, so this part is empty.

### partition (Conditional Output, HashAgg only)

Output in Columnar mode when `partitionWise` is not Top, indicating partition-level aggregation.

### index (Conditional Output, MysqlAgg only)

Physical index name available at the DN layer.

---

## Examples

### Simple Aggregation

```sql
SELECT dept_id, COUNT(*), SUM(salary) FROM emp GROUP BY dept_id;
```

```
HashAgg(group="dept_id", COUNT()="COUNT()", SUM(salary)="SUM(salary)")
  LogicalView(...)
```

### Scalar Aggregation (No GROUP BY)

```sql
SELECT COUNT(*), MAX(age) FROM t;
```

```
HashAgg(COUNT()="COUNT()", MAX(age)="MAX(age)")
  LogicalView(...)
```

### DISTINCT Aggregation

```sql
SELECT COUNT(DISTINCT name) FROM t;
```

```
HashAgg(COUNT(DISTINCT name)="COUNT(DISTINCT name)")
  LogicalView(...)
```

### Two-Phase Aggregation

```
PartialHashAgg(group="dept_id", SUM(salary)="SUM(salary)")
  LogicalView(...)
```

First stage partial aggregation, upstream HashAgg (final) completes the merge.

### MysqlAgg with Index

```
MysqlAgg(group="category", COUNT()="COUNT()", index="idx_category")
  MysqlTableScan(...)
```
