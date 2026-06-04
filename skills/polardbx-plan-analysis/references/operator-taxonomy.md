# PolarDB-X Execution Plan Operator Taxonomy

---

## 1. TableScan

| Operator | Display Name |
|----------|-------------|
| LogicalTableScan | LogicalTableScan |
| LogicalView | LogicalView |
| LogicalIndexScan | IndexScan |
| MysqlTableScan | MysqlTableScan |
| ParallelTableScan | ParallelTableScan |
| OSSTableScan | OSSTableScan |
| OrcTableScan | OrcTableScan |

---

## 2. Join

| Operator | Display Name |
|----------|-------------|
| LogicalJoin | LogicalJoin |
| LogicalSemiJoin | LogicalSemiJoin |
| LogicalCorrelate | CorrelateApply |
| LogicalColCorrelate | ColCorrelate |
| HashJoin | HashJoin |
| NLJoin | NLJoin |
| SortMergeJoin | SortMergeJoin |
| BKAJoin | BKAJoin |
| HashGroupJoin | HashGroupJoin |
| BushyJoin | BushyJoin |
| LookupJoin | LookupJoin |
| SemiHashJoin | SemiHashJoin |
| SemiNLJoin | SemiNLJoin |
| SemiBKAJoin | SemiBKAJoin |
| SemiSortMergeJoin | SemiSortMergeJoin |
| MaterializedSemiJoin | MaterializedSemiJoin |
| MysqlHashJoin | MysqlHashJoin |
| MysqlNLJoin | MysqlNLJoin |
| MysqlIndexNLJoin | MysqlIndexNLJoin |
| MysqlSemiHashJoin | MysqlSemiHashJoin |
| MysqlSemiNLJoin | MysqlSemiNLJoin |
| MysqlSemiIndexNLJoin | MysqlSemiIndexNLJoin |
| MysqlMaterializedSemiJoin | MysqlMaterializedSemiJoin |
| MysqlCorrelate | MysqlCorrelate |

---

## 3. Agg

| Operator | Display Name |
|----------|-------------|
| LogicalAggregate | LogicalAgg |
| HashAgg | HashAgg |
| SortAgg | SortAgg |
| MysqlAgg | MysqlAgg |

---

## 4. Sort

| Operator | Display Name |
|----------|-------------|
| LogicalSort | LogicalSort / Limit |
| MemSort | MemSort |
| TopN | TopN |
| GroupTopN | GroupTopN |
| Limit | Limit |
| MergeSort | MergeSort |
| MysqlSort | MysqlSort |
| MysqlTopN | MysqlTopN |
| MysqlLimit | MysqlLimit |

---

## 5. Window

| Operator | Display Name |
|----------|-------------|
| LogicalWindow | Window |
| HashWindow | HashWindow |
| SortWindow | SortWindow |

---

## 6. Filter

| Operator | Display Name |
|----------|-------------|
| LogicalFilter | Filter |
| PhysicalFilter | Filter |

---

## 7. Project

| Operator | Display Name |
|----------|-------------|
| LogicalProject | Project |
| LogicalCalc | Calc |
| PhysicalProject | Project |

---

## 8. Values

| Operator | Display Name |
|----------|-------------|
| LogicalValues | Values |
| LogicalDynamicValues | DynamicValues |

---

## 9. TableLookup

| Operator | Display Name |
|----------|-------------|
| LogicalTableLookup | TableLookup |

---

## 10. Exchange

| Operator | Display Name |
|----------|-------------|
| LogicalExchange | Exchange |
| Gather | Gather |
| MppExchange | Exchange |
| ColumnarExchange | Exchange |
| LocalBufferNode | LocalBuffer |

---

## 11. CTE — Common Table Expression

| Operator | Display Name |
|----------|-------------|
| LogicalCTEAnchor | LogicalCTEAnchor |
| LogicalCTEProducer | LogicalCTEProducer |
| LogicalCTEConsumer | LogicalCTEConsumer |
| PhysicalCTEAnchor | CTEAnchor |
| PhysicalCTEProducer | CTEProducer |
| PhysicalCTEConsumer | CTEConsumer |

---

## 12. SetOp

| Operator | Display Name |
|----------|-------------|
| LogicalUnion | UnionAll / UnionDistinct |
| LogicalHybridUnion | UnionAll / UnionDistinct |
| LogicalIntersect | Intersect |
| LogicalMinus | Minus |
| PhyViewUnion | PhyViewUnion |

---

## 13. Expand

| Operator | Display Name |
|----------|-------------|
| LogicalExpand | Expand |

---

## 14. DML — Data Modification

| Operator | Display Name |
|----------|-------------|
| LogicalTableModify | LogicalTableModify |
| LogicalInsert | LogicalInsert |
| LogicalInsertIgnore | LogicalInsertIgnore |
| LogicalUpsert | LogicalUpsert |
| LogicalReplace | LogicalReplace |
| LogicalModify | LogicalModify |
| LogicalRelocate | LogicalRelocate |
| LogicalModifyView | LogicalModifyView |
| SingleTableInsert | SingleTableInsert |
| BroadcastTableModify | BroadcastTableModify |

---

## 15. OutFile

| Operator | Display Name |
|----------|-------------|
| LogicalOutFile | LogicalOutFile |

---

## 16. BaseTableOperation

| Operator | Display Name |
|----------|-------------|
| BaseQueryOperation | — |
| BaseTableOperation | — |
| PhyTableOperation | PhyTableOperation |
| DirectTableOperation | LogicalView |
| DirectShardingKeyTableOperation | DirectShardingKeyOperation |
| DirectMultiDBTableOperation | DirectMultiDBTableOperation |
| SingleTableOperation | LogicalView |
| PhyQueryOperation | PhyQuery |
| PhyOSSTableOperation | PhyOSSTableOperation |
| PhyDdlTableOperation | PhyDdlTableOperation |

---

## 17. Other

| Operator | Display Name |
|----------|-------------|
| LogicalTableFunctionScan | TableFunctionScan |
| RuntimeFilterBuilder | RuntimeFilterBuilder |
