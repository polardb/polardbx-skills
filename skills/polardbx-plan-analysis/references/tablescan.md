# TableScan — Table Scan Operator EXPLAIN Output Reference

Data reading entry point. LogicalView encapsulates subplans pushed down to DN. LogicalIndexScan / OSSTableScan inherit from LogicalView and share the same output format, differing only in display name. LogicalTableScan uses the default implementation and is rare in final plans.

---

## Input / Output Attributes

| Attribute | Input | Output |
|-----------|-------|--------|
| Columns | None (leaf node) | SELECT list of pushed-down SQL |
| Order | — | Determined by pushed-down ORDER BY (usually no guarantee) |
| Distribution | — | No specific distribution; only OSSTableScan may preserve table partitioning via partitionWise |

---

## Output Format and Display Names

```
LogicalView(tables="<spec>", shardCount=<N>, sql="<pushed_sql>")
IndexScan(tables="<spec>", shardCount=<N>, sql="<pushed_sql>")
OSSTableScan(tables="<spec>", shardCount=<N>, sql="<pushed_sql>")
```

| Operator | Display Name |
|----------|-------------|
| LogicalView | `LogicalView` |
| LogicalIndexScan | `IndexScan` |
| OSSTableScan | `OSSTableScan` |

---

## Field Descriptions

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| tables | Conditional | Physical table and partition information (see below) |
| shardCount | shardCount > 1 or == 0 | Shard count. 0 means no scan needed after partition pruning |
| sql | Required | SQL pushed down to DN for execution |
| partition | Columnar mode | partitionWise attribute |
| XPlan | `EXPLAIN_X_PLAN=true` and template exists | XPlan physical plan (Protobuf JSON) |
| pruningInfo | Supports dynamic pruning and non-empty | Dynamic partition pruning information |
| physicalPlan | `EXPLAIN_SHOW_PHYSICAL_PLAN=true` and DETAIL/COST/ANALYZE mode (except OSSTableScan) | DN's MySQL EXPLAIN result |

### tables Format

**Partitioned tables**: `tables="<logical_name>[<partition_list>]"`
- Partitions <= 10 listed in full: `t[p1,p2,p3]`
- Partitions > 10 abbreviated: `t[p1,p2,p3,...p100]`
- Broadcast/replica tables do not display partition list
- Multi-table push-down separated by comma: `t1[p1,p2],t2[p3,p4]`

**Legacy sharded tables**: `tables="<logical_name>_<shard_suffix>"`

### sql Field

SQL pushed down to DN, parameterized with `?` placeholders, newlines replaced with spaces. This is the most direct information for understanding what a LogicalView does.

### physicalPlan Field

JSON array format, interpreted by MySQL EXPLAIN standard fields:

| Field | Meaning |
|-------|---------|
| table | Physical table name |
| selectType | Query type (SIMPLE, PRIMARY, SUBQUERY, etc.) |
| type | Access type (ALL, index, range, ref, eq_ref, const, etc.) |
| key | Actually used index name (null means not used) |
| rows | Estimated scan row count |
| filtered | Post-filter row percentage |
| extra | Additional information (Using where/index/filesort, etc.) |

**Known defect**: When used as BKAJoin lookup side, physicalPlan is generated without actual IN key values, so MySQL optimizer cannot correctly select indices or estimate rows — `key` and `rows` do not reflect reality and should be ignored.

---

## Special Display Mode: EXPLAIN_LOGICALVIEW

When `EXPLAIN_LOGICALVIEW=true`, LogicalView switches to expanded mode:

```
LogicalView(joinIndex="<index_name>")
  <Internal Mysql* operator subtree>
```

Does not output tables/shardCount/sql, instead expands the internal pushed-down plan tree (MysqlTableScan/MysqlHashJoin/MysqlAgg, etc.) as child nodes. `joinIndex` is output only when LogicalView is a BKA lookup table, showing the lookup index name.

---

## Examples

```
LogicalView(tables="t[p1]", sql="SELECT `id`, `name`, `age` FROM `t` WHERE (`id` = ?)")

LogicalView(tables="t[p1,p2,p3,p4]", shardCount=4, sql="SELECT ... WHERE (`name` = ?)")

LogicalView(tables="t[p1,p2,p3,...p64]", shardCount=64, sql="SELECT ...")

IndexScan(tables="g_idx_name[p1,p2]", shardCount=2, sql="SELECT `id`, `name` FROM `g_idx_name` WHERE (`name` = ?)")

LogicalView(tables="t1[p1],t2[p1]", sql="SELECT ... FROM `t1` INNER JOIN `t2` ON (`t1`.`id` = `t2`.`id`) WHERE ...")
```

- Single shard does not output shardCount
- Partitions > 10 abbreviated display
- Two co-located tables JOIN pushed down to same DN, sql contains complete JOIN
