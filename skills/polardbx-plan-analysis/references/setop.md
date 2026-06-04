# SetOp — Set Operation Operator EXPLAIN Output Reference

UNION, INTERSECT, EXCEPT, and physical-layer PhyViewUnion. Output columns are determined by the first input; all branches must have compatible column count and types.

---

## Output Format

| Operator | Format | Description |
|----------|--------|-------------|
| UnionAll | `UnionAll(concurrent=true)` | `all=true`, concurrent execution |
| UnionDistinct | `UnionDistinct(concurrent=true)` | `all=false`, deduplication |
| Intersect | `Intersect(all=<true\|false>)` | Default implementation |
| Minus | `Minus()` | No extra fields |
| PhyViewUnion | `PhyViewUnion(concurrent=true)` | Multi-shard physical union |

---

## Examples

```
UnionAll(concurrent=true)
  LogicalView(tables="t1", ...)
  LogicalView(tables="t2", ...)

UnionDistinct(concurrent=true)
  LogicalView(tables="t1", ...)
  LogicalView(tables="t2", ...)

PhyViewUnion(concurrent=true)
  LogicalView(tables="t[p0]", ...)
  LogicalView(tables="t[p1]", ...)
```
