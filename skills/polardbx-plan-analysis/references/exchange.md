# Exchange — Data Exchange Operator EXPLAIN Output Reference

Four Exchange operators handle data transfer and redistribution between nodes/shards. **Does not change column definitions**, only changes physical distribution and sort attributes.

---

## OptimizerType to Exchange Operator Mapping

| OptimizerType | Exchange Operator | Identification |
|---------------|-------------------|----------------|
| SMP | Gather | LogicalView + Gather, CN collects then computes locally |
| MPP | MppExchange | Exchange(distribution=..., collation=[...]) |
| COLUMNAR | ColumnarExchange | Same display as MPP, with partition=[local] characteristics |

**Identification criteria**:

| | SMP | MPP | COLUMNAR |
|--|-----|-----|----------|
| Scan operator | LogicalView | LogicalView | OSSTableScan |
| Exchange | Gather(concurrent=true) | Exchange(distribution=...) | Exchange(distribution=...) |
| Join/Agg partition | None | None | `partition=[local]` |
| Scan partition | None | None | `partition=[local, remote]` / `partition=[remote]` |

MPP is a plan-level capability declaration (supports cross-CN coordination); whether it actually executes across multiple nodes is determined at runtime.

---

## Output Format and Fields

```
Gather(concurrent=true)
Exchange(distribution=<dist>, collation=[<sort>])
Exchange(distribution=<dist>, collation=[<sort>], partition=...)
LocalBuffer()
```

| Field | Operator | Description |
|-------|----------|-------------|
| concurrent | Gather | Fixed true |
| distribution | Exchange | Redistribution strategy (see below) |
| collation | Exchange | Sort constraint, `[]` for unordered, `[0 ASC, 1 DESC]` for ordered (merge sort semantics) |
| partition | ColumnarExchange | Output in Columnar mode when partitionWise is not Top |

### distribution Format

**MppExchange**:

| Value | Meaning |
|-------|---------|
| `broadcast` | Broadcast to all nodes (small table build side) |
| `single` | Collect to single node (global TopN/scalar aggregation) |
| `hash[cols]` | Hash redistribute by columns, e.g., `hash[0]`, `hash[0, 1]` |
| `rr` | Round-robin |
| `any` | No constraint |

**ColumnarExchange** (additionally supports shardCnt):

| Value | Meaning |
|-------|---------|
| `hash[cols]N` | Hash to N partitions by columns, e.g., `hash[0]96`. N typically equals OSSTableScan's shardCount |

`hash[col]N` with numeric suffix -> Columnar mode can be determined.

---

## Operator Semantics

### Gather

SMP mode, collects data from multiple DN shards to a single CN node. Simple concatenation across shards, **does not preserve order**.

For sort push-down scenarios, MergeSort operator (not Gather) performs k-way merge sort to guarantee global ordering. EXPLAIN shows Gather -> no sort guarantee; MergeSort -> order-preserving merge.

Appears when LogicalView involves multiple Groups (multi-database scan). No Gather is inserted for single-database scans.

### MppExchange

MPP mode, redistributes data according to distribution, appears at Fragment boundaries.

| distribution | Typical downstream |
|-------------|-------------------|
| broadcast | HashJoin build side |
| hash[cols] | HashAgg / HashJoin (align distribution) |
| single | TopN / scalar computation |

When collation is non-empty, the receiving end performs merge sort.

### ColumnarExchange

Columnar mode, functionally identical to MppExchange, additionally supports partitionWise trait.

Characteristics distinguishing from MppExchange:
- OSSTableScan has `partition=[local, remote]` / `partition=[remote]`
- HashJoin/HashAgg has `partition=[local]`
- `hash[col]N` in distribution where N = shardCount

### LocalBuffer

CN-local pipeline buffer that materializes intermediate results to allow multiple reads.

Typical position: Inside of NLJoin (right child).

---

## Field Comparison

| Operator | Display Name | concurrent | distribution | collation | partition |
|----------|-------------|-----------|-------------|-----------|-----------|
| Gather | `Gather` | Required | — | — | — |
| MppExchange | `Exchange` | — | Required | Required | — |
| ColumnarExchange | `Exchange` | — | Required | Required | Conditional |
| LocalBufferNode | `LocalBuffer` | — | — | — | — |

---

## Examples

### SMP Gather

```
Gather(concurrent=true)
  LogicalView(tables="orders[p1,p2,p3,p4]", shardCount=4, sql="SELECT ...")
```

### MPP Hash Shuffle + Two-Phase Aggregation (Q4)

```
Exchange(distribution=single, collation=[0 asc-nulls-first])
  MemSort(sort="o_orderpriority asc")
    HashAgg(group="o_orderpriority", order_count="sum(order_count)")
      Exchange(distribution=hash[0], collation=[])
        LogicalView(tables="orders[p1,...p4],lineitem[p1,...p4]", shardCount=4, sql="... GROUP BY o_orderpriority")
```

- `hash[0]`: Hash by o_orderpriority, same group keys converge to same partition
- `single, collation=[0 asc]`: Order-preserving collection to single node (merge sort)

### MPP Broadcast + HashJoin (Q3)

```
HashJoin(condition="o_custkey = c_custkey", type="inner")
  LogicalView(tables="orders[p1,...p4],lineitem[p1,...p4]", shardCount=4, sql="... JOIN ...")
  Exchange(distribution=broadcast, collation=[])
    LogicalView(tables="customer[p1,...p4]", shardCount=4, sql="... WHERE c_mktsegment = ?")
```

Customer filtered then broadcast, left side orders-lineitem stays in place.

### Columnar PartitionWise Join (Q3)

```
HashJoin(condition="o_orderkey = l_orderkey", type="inner", partition=[local])
  OSSTableScan(tables="lineitem_col_index[p1,...p96]", shardCount=96, partition=[local, remote])
  HashJoin(condition="o_custkey = c_custkey", type="inner")
    OSSTableScan(tables="orders_col_index[p1,...p96]", shardCount=96, partition=[local, remote])
    Exchange(distribution=broadcast, collation=[])
      OSSTableScan(tables="customer_col_index[p1,...p96]", shardCount=96)
```

- `partition=[local]`: lineitem and orders share the same partition key, local join
- customer broadcast

### Columnar Hash Shuffle + Broadcast (Q5)

```
HashJoin(condition="l_orderkey = o_orderkey", type="inner")
  OSSTableScan(tables="lineitem_col_index[p1,...p96]", shardCount=96, partition=[remote])
  Exchange(distribution=hash[6]96, collation=[])
    HashJoin(condition="o_custkey = c_custkey", type="inner")
      Exchange(distribution=hash[1]96, collation=[])
        OSSTableScan(tables="orders_col_index[p1,...p96]", shardCount=96)
      HashJoin(condition="n_nationkey = c_nationkey", type="inner")
        OSSTableScan(tables="customer_col_index[p1,...p96]", partition=[remote])
        Exchange(distribution=broadcast, collation=[])
          HashJoin(condition="n_regionkey = r_regionkey", type="inner")
            OSSTableScan(tables="nation_col_index[p1]", partition=[remote])
            Exchange(distribution=broadcast, collation=[])
              OSSTableScan(tables="region_col_index[p1]")
```

- broadcast: nation, region small tables broadcast
- `hash[1]96`: orders hashed by o_custkey to 96 partitions to align with customer
- `hash[6]96`: join result hashed by o_orderkey to 96 partitions to align with lineitem
- lineitem large table stays in place (`partition=[remote]`)

### LocalBuffer (NLJoin Inner Side)

```
NlJoin(condition="true", type="inner")
  Exchange(distribution=single, collation=[])
    ...
  LocalBuffer()
    Exchange(distribution=broadcast, collation=[])
      ...
```

NLJoin inner side materialized, allows repeated reads for each outer row.
