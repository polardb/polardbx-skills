# Sort — Sort Operator EXPLAIN Output Reference

Only changes row order and count, does not change column space or data distribution.

---

## Input / Output Attributes

| Attribute | Input | Output |
|-----------|-------|--------|
| Columns | Any | Same as input |
| Order | MergeSort requires input already ordered; others have no requirement | Ordered by sort keys (Limit without sort is indeterminate) |
| Distribution | Any | Same as input |

---

## Output Format

```
LogicalSort(sort="<col> ASC,<col> DESC", offset=<N>, fetch=<M>)
Limit(offset=<N>, fetch=<M>)
MemSort(sort="<col> ASC,<col> DESC")
TopN(sort="<col> ASC,<col> DESC", offset=<N>, fetch=<M>)
MergeSort(sort="<col> ASC,<col> DESC", offset=<N>, fetch=<M>)
GroupTopN(sort="<col> ASC", offset=<N>, fetch=<M>, group={0, 1}, partial=true)
```

LogicalSort automatically switches its display name to `Limit` when there are no sort keys.

---

## Field Descriptions

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| sort | When sort keys exist | Sort key list, `<column_name> <ASC\|DESC>` comma-separated |
| offset | When non-null | Rows to skip, corresponds to OFFSET |
| fetch | When non-null (required for TopN/GroupTopN) | Row count limit, corresponds to LIMIT |
| group | GroupTopN only, required | Group column set like `{0, 1}`, TopN computed independently per group |
| partial | GroupTopN only, conditional | Output when `partial=true` |

---

## Field Comparison

| Operator | sort | offset | fetch | Extra |
|----------|------|--------|-------|-------|
| LogicalSort | Conditional | Conditional | Conditional | — |
| MemSort | Conditional | — | — | — |
| TopN | Conditional | Conditional | Required | — |
| GroupTopN | Conditional | Conditional | Required | group, partial |
| Limit | — | Conditional | Conditional | — |
| MergeSort | Required | Conditional | Conditional | — |

---

## Examples

```
MemSort(sort="name ASC,age DESC")
TopN(sort="id ASC", fetch=?1)
Limit(offset=?0, fetch=?1)
MergeSort(sort="id ASC", fetch=?1)
GroupTopN(sort="salary DESC", fetch=3, group={0})
```

- `fetch=?N`: Parameterized LIMIT value
- `offset=?N`: Parameterized OFFSET value
- MergeSort: Merges ordered streams from multiple shards
- GroupTopN: TopN computed independently per group (`ROW_NUMBER() OVER (PARTITION BY ...) <= N` optimization)
