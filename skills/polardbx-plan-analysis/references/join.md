# Join — Join Operator

The category with the most operator varieties. All Joins share a common `condition` + `type` base format, with some having additional fields.

---

## Common Attributes

| Attribute | Description |
|-----------|-------------|
| Output columns | Regular Join: left + right concatenated; Semi/Anti: left table columns only |
| Output order | No guarantee (SortMergeJoin requires input ordered by join keys) |

### Common Fields

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| condition | Required (except CorrelateApply/ColCorrelate) | Join condition, left table columns first, right table columns second |
| type | Required | `inner` / `left` / `right` / `full` / `semi` / `anti` |

---

## Logical Joins

Execution mode: Logical layer, intermediate state during RBO/CBO phase, generally does not appear in final plans (except CorrelateApply).

### LogicalJoin

Logical join with undetermined physical implementation, converted to physical Join during CBO phase.

### LogicalSemiJoin

Semi/anti join after decorrelation of EXISTS/IN/NOT EXISTS/NOT IN.

### CorrelateApply

Correlated subquery Apply operator, **does not output condition**. When appearing in the final plan, it means decorrelation was unsuccessful — executes as nested loops row by row, with high performance risk.

```
CorrelateApply(cor=$cor0, leftConditions=[], opKind=EXISTS, type=SEMI)
```

| Field | Description |
|-------|-------------|
| cor | Correlation variable ID |
| opKind | `IN` / `EXISTS` / `SOME` / `ALL` / `SCALAR_QUERY` |
| type | Join type |

**Cache Node**: Right-side child nodes that do not reference correlation variables are computed only once, shown at the end of EXPLAIN as `cache node: <digest>`. Can be disabled with `FORBID_APPLY_CACHE=true`.

### ColCorrelate

Column correlate operator (Columnar mode), only outputs `cor` and `type`.

---

## CN Physical Joins

Execution mode: CN layer.

### HashJoin

Build side constructs a hash table, probe side probes row by row.

```
HashJoin(condition="<cond>", type="<type>", build="<left|right>", partition=...)
```

| Extra Field | Output Condition | Description |
|-------------|-----------------|-------------|
| build | When `outerBuild=true` | Indicates outer table is the build side. When not output, build side defaults to right child |
| partition | Columnar mode | partitionWise flag: `[local]` / `[local, remote]` |

### NLJoin

Nested loop join. Left child drives, right child scans row by row. Suitable for no equi-condition or small table driving.

```
NlJoin(condition="<cond>", type="<type>")
```

### SortMergeJoin

Sort-merge join. Requires both sides ordered by join keys.

```
SortMergeJoin(condition="<cond>", type="<type>")
```

### BKAJoin

Batched Key Access: batches left-side keys, right side looks up via `IN (...)` index.

```
BKAJoin(condition="<cond>", type="<type>")
```

**Identification**: Lookup side LogicalView's sql field contains `in (...)`.

**Sharding behavior**:
- Lookup side displays the table's total shard count (e.g., `[p1,...p32]`), **actual execution does not access all shards**
- Join key is lookup side's partition key -> dynamically calculates target shards, accesses only relevant shards
- Join key is not the partition key -> IN query broadcasts to all shards
- EXPLAIN **cannot distinguish** between these two cases; check the table's partition definition

**Lookup side physicalPlan limitation**: Generated without specific IN key values, so DN's EXPLAIN key selection and rows estimation are unreliable.

**Scenario 1 — GSI table lookup** (most typical):

```
BKAJoin(condition="id = id", type="inner")
  IndexScan(tables="g_idx[p31]", sql="SELECT `id` FROM `g_idx` WHERE (`c_varchar` = ?)")
  Gather(concurrent=true)
    LogicalView(tables="main_table[p1,...p32]", shardCount=32, sql="... WHERE (`id` in (...))")
```

> If main_table is partitioned by id, only shards matching the primary key are accessed.

**Scenario 2 — Cross-shard two-table join**:

```
BKAJoin(condition="c_varchar = c_varchar", type="inner")
  LogicalView(tables="t1[p2]", sql="SELECT `c_varchar` FROM `t1` WHERE (`id` = ?)")
  Gather(concurrent=true)
    LogicalView(tables="t2[p1,...p32]", shardCount=32, sql="... WHERE (`c_varchar` in (...))")
```

> If t2 is partitioned by c_varchar, dynamically prunes to matching shards; otherwise broadcasts to all 32 shards.

**Scenario 3 — Two-level nested BKA (GSI + table lookup)**:

```
BKAJoin(condition="c_varchar = c_varchar", type="inner")
  LogicalView(tables="t1[p2]", ...)
  Project(...)
    BKAJoin(condition="id = id", type="inner")
      Gather -> IndexScan(sql="... WHERE (`c_varchar` in (...))")
      Gather -> LogicalView(sql="... WHERE (`id` in (...))")
```

### HashGroupJoin

Hash Join + Agg merged execution. When `parallel=true`, displayed as `ParallelHashJoin`.

```
HashGroupJoin(condition="<cond>", type="<type>", partition=...)
```

---

## CN Physical Semi Joins

Execution mode: CN layer. Outputs left table columns only.

| Operator | Format | Description |
|----------|--------|-------------|
| SemiHashJoin | `SemiHashJoin(condition, type, build, partition)` | build is required: `inner` (right child) / `outer` (reverse build) |
| SemiNLJoin | `SemiNLJoin(condition, type)` | Nested loop |
| SemiBKAJoin | `SemiBKAJoin(condition, type)` | BKA approach |
| SemiSortMergeJoin | `SemiSortMergeJoin(condition, type)` | Sort-merge |
| MaterializedSemiJoin | `MaterializedSemiJoin(condition, type)` | Right side materialized then probed |

SemiHashJoin `build="outer"` appears when the subquery result set is large and the main query side is small.

---

## DN Pushed-Down Joins

Execution mode: DN layer. MysqlXxx series (MysqlHashJoin, etc.), folded into LogicalView's pushed-down SQL, not directly visible in standard EXPLAIN. Only visible in `EXPLAIN_LOGICALVIEW=true` mode.

---

## Field Comparison Quick Reference

| Operator | condition | type | build | partition |
|----------|-----------|------|-------|-----------|
| LogicalJoin / LogicalSemiJoin | Required | Required | — | — |
| CorrelateApply / ColCorrelate | — | Required | — | — |
| HashJoin | Required | Required | Conditional | Conditional |
| NlJoin / SortMergeJoin / BKAJoin | Required | Required | — | — |
| HashGroupJoin | Required | Required | — | Conditional |
| SemiHashJoin | Required | Required | Required | Conditional |
| Other Semi Joins | Required | Required | — | — |
