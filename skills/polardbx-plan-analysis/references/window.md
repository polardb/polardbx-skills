# Window — Window Function Operator EXPLAIN Output Reference

Appends window function result columns to input columns (does not remove existing columns). Three operators (Window/HashWindow/SortWindow) share the same output format, differing only in display name.

---

## Input / Output Attributes

| Attribute | Input | Output |
|-----------|-------|--------|
| Columns | Any | Input columns + window result columns (appended at end) |
| Order | SortWindow requires input ordered by window ORDER BY; others have no requirement | No guarantee |
| Distribution | Any | Same as input |

---

## Output Format

```
Window(<col1>="<col1>", ..., <win_col>="window#0ROW_NUMBER()", Reference Windows="window#0=...", constants=[...], partition=...)
```

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| Input column passthrough | Required (N columns) | Key-value pairs, generally key equals value (column name passthrough) |
| Window result columns | Required (M columns) | Value is `window#<N><aggCall>`, e.g., `window#0ROW_NUMBER()`, `window#1SUM($2)` |
| Reference Windows | Required | Window definition summary: `window#N=window(partition {cols} order by [col DIR] rows between ... and ...)` |
| constants | When non-empty | Constants referenced in window frame boundaries |
| partition | Columnar mode | partitionWise attribute |

### Reference Windows Contains

- `partition {cols}`: PARTITION BY column set
- `order by [col DIR]`: ORDER BY definition
- `rows between ... and ...`: Window frame boundary (ROWS/RANGE)

---

## Display Names

| Operator | Display Name |
|----------|-------------|
| LogicalWindow | `Window` |
| HashWindow | `HashWindow` |
| SortWindow | `SortWindow` |

---

## Examples

```
Window(id="id", name="name", dept_id="dept_id", salary="salary", rn="window#0ROW_NUMBER()", Reference Windows="window#0=window(partition {2} order by [3 DESC] rows between UNBOUNDED PRECEDING and CURRENT ROW)")
  LogicalView(...)

HashWindow(id="id", v="v", grp="grp", sum_v="window#0SUM($1)", rnk="window#1RANK()", Reference Windows="window#0=window(partition {} order by [0 ASC] ...),window#1=window(partition {2} order by [1 ASC] ...)")
  LogicalView(...)
```

- Input column passthrough comes first, window result columns follow
- `window#N` links to window definitions in Reference Windows
- Multiple window functions belong to different window numbers
