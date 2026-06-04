# EXPLAIN COST / EXPLAIN ANALYZE — Cost and Runtime Statistics Reference

Appends cost estimation and runtime statistics after the standard EXPLAIN output.

---

## Output Format

**EXPLAIN COST**:
```
<operator>(<attributes>): rowcount = <N>, cumulative cost = value = <V>, cpu = <C>, memory = <M>, io = <I>, net = <NET>
```

**EXPLAIN ANALYZE** (appends to COST output):
```
..., actual time = <startup> + <duration>, actual rowcount = <R>, actual memory = <MEM>, instances = <INST>
```

Conditional fields (appear when > 0): `runtime filtered count`, `io bytes`, `spill count`

---

## Field Descriptions

### Cost Estimation (EXPLAIN COST)

| Field | Description |
|-------|-------------|
| `rowcount` | Estimated output row count (rounded up) |
| `cumulative cost` | Cumulative cost of operator and all subtrees (5-dimension weighted) |

Cumulative cost subfields:

| Subfield | Meaning |
|----------|---------|
| `value` | Composite cost = `cpu + ioWeight * io + netWeight * net + memoryWeight * memory` |
| `cpu` | Computation cost related to row count processing |
| `memory` | Memory estimation |
| `io` | Disk IO cost |
| `net` | Network transfer cost |

Outputs `huge` when cost is extremely large.

### Runtime Statistics (EXPLAIN ANALYZE)

| Field | Description |
|-------|-------------|
| `actual time` | `startup + duration` (seconds, 3 decimal places). **Not maintained — unreliable for performance analysis** |
| `actual rowcount` | Actual output row count |
| `actual memory` | Actual memory usage (bytes) |
| `instances` | Number of operator instances (parallelism) |
| `runtime filtered count` | Conditional. Rows filtered by Runtime Filter |
| `io bytes` | Conditional. IO bytes read |
| `spill count` | Conditional. Number of spills to disk |

---

## Analysis Guidelines

### rowcount Deviation

| Deviation | Meaning | Impact |
|-----------|---------|--------|
| Estimated ~ actual | Cardinality is accurate | Plan is reasonable |
| Estimated >> actual | Overestimation | Unnecessary shuffle/sort |
| Estimated << actual | Underestimation | NLJoin chosen over HashJoin, memory overflow |

### instances

- 0: Not scheduled for execution (data is empty or short-circuited)
- 1: Single-threaded
- \> 1: Multiple parallel instances (MPP/Columnar)

---

## Known Statistics Defects

| Operator | Issue | Recommendation |
|----------|-------|----------------|
| **Gather** | actual rowcount is always 0 (executor disables metric collection) | Check the child LogicalView's actual rowcount |
| **BKAJoin lookup side** | rowcount estimation ignores IN filter (uses static selectivity) | Check BKAJoin's own rowcount |
| **Exchange** | actual rowcount may be incomplete (depends on deserialization accumulation) | Use child/parent operator values |

### Reliability Summary

| Operator | actual rowcount | rowcount estimation |
|----------|----------------|-------------------|
| Join / SemiJoin | Reliable | Reliable |
| Agg / Filter / Project | Reliable | Reliable |
| Sort / TopN / Limit | Reliable | Reliable |
| LogicalView / Window / UnionAll | Reliable | Reliable (unreliable as BKAJoin lookup side) |
| **Gather** | Unreliable (always 0) | Reliable |
| **Exchange** | For reference only | Reliable |

---

## Examples

### EXPLAIN COST

```
Gather(concurrent=true): rowcount = 1.0, cumulative cost = value = 755002.0, cpu = 2.0, memory = 0.0, io = 1.0, net = 1.5
  LogicalView(tables="t2[p1,p2,p3]", shardCount=3, sql="SELECT `v1`, `v2`, `v3` FROM `t2` AS `t2`"): rowcount = 1.0, cumulative cost = value = 755001.0, cpu = 1.0, memory = 0.0, io = 1.0, net = 1.5
```

Gather's own cost = 755002 - 755001 = 1 (cpu +1 only).

### EXPLAIN ANALYZE

```
Gather(concurrent=true): rowcount = 1.0, cumulative cost = ..., actual time = 0.000 + 0.000, actual rowcount = 0, actual memory = 0, instances = 1
  LogicalView(tables="t2[p1,p2,p3]", shardCount=3, sql="..."): rowcount = 1.0, cumulative cost = ..., actual time = 0.000 + 0.000, actual rowcount = 0, actual memory = 0, instances = 0
```

instances = 0 indicates LogicalView was not actually scheduled. Gather actual rowcount = 0 is a known defect and does not reflect actual data volume.

### With Conditional Fields

```
HashJoin(...): rowcount = 100.0, ..., actual time = 0.015 + 0.230, actual rowcount = 95, runtime filtered count = 5000, actual memory = 1048576, spill count = 2, instances = 4
```

Estimated 100 rows, actual 95 rows, Runtime Filter filtered 5000 rows, 1MB memory, 2 spills, 4 parallel instances.
