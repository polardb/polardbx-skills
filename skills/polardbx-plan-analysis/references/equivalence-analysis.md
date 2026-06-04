# SQL-to-Execution-Plan Equivalence Analysis Reference

A systematic checking framework to determine whether a SQL statement and its execution plan are semantically equivalent.

---

## 14 Analysis Dimensions

### 1. Filter Conditions

- Whether every predicate in WHERE/HAVING appears in the plan (Filter condition or TableScan pushed-down SQL)
- Whether AND/OR logic is consistent, whether inequality direction is reversed
- Whether conditions are incorrectly pushed down (JOIN condition misplaced in Filter, or vice versa)

### 2. Column References & Projection

- Whether Project output list expressions match the SQL SELECT list
- Dual-input operator (Join) column indices: left table columns first, right table columns second — check if offset is correct
- Whether constants `?N` correspond to original literal values
- Whether expression operator precedence is preserved (e.g., `a * (1 - b)` vs `a * 1 - b`)
- Whether correlated variable `$cor0.col` references are correct

### 3. JOIN Conditions & Type

- **Conditions**: Completeness of equi/non-equi conditions, column reference correctness
- **Type**: `EXISTS` -> semi, `NOT EXISTS` -> anti, `IN` -> semi, `NOT IN` -> anti
- **Order**: Inner join order does not affect semantics; outer join left/right cannot be swapped
- **BKAJoin**: Lookup side `in (...)` maintains equi-join semantics
- **CorrelateApply**: `cor`/`opKind`/`type` reflects original subquery semantics

### 4. Aggregation

- GROUP BY column set is consistent (order may differ)
- Aggregate function type and parameter columns are correct, DISTINCT flag is preserved
- Two-phase aggregation merge functions are correct: partial COUNT -> final SUM, partial SUM -> final SUM
- Scalar aggregation (no GROUP BY) is handled correctly

### 5. Sort & LIMIT

- ORDER BY column set and direction (ASC/DESC) are consistent
- LIMIT/OFFSET values are correct (`?N` corresponds to original values)
- Two-phase TopN: local fetch >= global offset + fetch
- MergeSort sort keys match pushed-down ORDER BY

### 6. Exchange/Shuffle Strategy

- **broadcast**: The correct table is being broadcast
- **hash[cols]**: Column indices are JOIN keys or GROUP BY keys
- **single**: Used for global TopN/sort/scalar aggregation
- **partition=[local]**: Both tables share the same partition key
- When collation is non-empty, sort keys match upstream expectations

### 7. Subquery Decorrelation

- After decorrelation, JOIN condition correctly reflects the original correlation condition
- Column index shift (CorrelVariable right-shift) is correct
- Semi/Anti Join outputs only left table columns
- Scalar subquery result column is correctly correlated back to the outer layer

### 8. CTE

- Producer subplan is equivalent to the WITH definition query
- Consumer sn number correctly corresponds to each reference point
- projects column indices are correct, conditions reference the CTE original column space (filter before project)

### 9. Window Functions

- PARTITION BY, ORDER BY columns and direction are consistent
- Function type is correct, window frame (ROWS/RANGE) is consistent
- Multiple window definitions correspond to each window in the SQL

### 10. Set Operations

- UNION ALL <-> `UnionAll`, UNION <-> `UnionDistinct`
- INTERSECT/EXCEPT `all` attribute is correct
- Each branch has compatible column count and types

### 11. TableScan Pushed-Down SQL

- LogicalView sql field contains correct WHERE/JOIN/aggregation
- Multi-table push-down JOIN conditions are complete
- Partition pruning is reasonable (shardCount)

### 12. Parameterization Restoration

- `?N` corresponds to original literals in left-to-right order
- IN list merged parameterization is correct
- `NOW()` -> `cast(? as datetime)` semantics are preserved

### 13. Expand (GROUPING SETS)

- Each grouping set expansion projection is correct (participating columns retain original values, non-participating columns set to null)
- groupId numbering corresponds to grouping sets
- ROLLUP/CUBE expansion count is correct

### 14. TableLookup

- Lookup condition is a correct primary key equi-join
- Output columns include all columns required by the SQL

---

## Checking Procedure

### 1. Bottom-Up Traversal

Start from TableScan leaf nodes and verify layer by layer upward.

### 2. Focus on Column Space Change Points

| Operator | Column Space Change |
|----------|-------------------|
| Project | Redefined by projects (add/remove/reorder/new columns) |
| Agg | Group keys + aggregate result columns |
| Window | Input columns + window result columns (appended at end) |
| Semi/Anti Join | Left table columns only |
| Expand | Input columns + groupId |
| CTEAnchor | Right child (main query) columns only |
| CTEConsumer | Redefined by projects (original Producer columns when no projects) |

### 3. Focus on Data Distribution Change Points

- After hash shuffle, whether JOIN both sides' distribution keys are aligned
- After broadcast, whether the build side is the broadcast table
- With partition=[local], whether both tables share the same partitioning
- Whether Agg group keys before/after shuffle match the shuffle columns (ensuring same keys converge to the same partition before aggregation)

### 4. Output Conclusion

Per-dimension verdict of equivalent/not equivalent; when not equivalent, specify location and reason.

---

## Output Format

```
## Equivalence Analysis

### [Dimension Name]
- Result: Equivalent / Not equivalent
- Explanation: ...

## Conclusion
SQL and plan are [equivalent / not equivalent]. [When not equivalent, describe the issue]
```
