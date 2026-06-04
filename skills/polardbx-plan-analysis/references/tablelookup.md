# TableLookup — Table Lookup Operator EXPLAIN Output Reference

GSI table lookup: first queries primary keys via index, then looks up the primary table for complete rows. During CBO phase, this is rewritten to BKAJoin / LookupJoin.

---

## Input / Output Attributes

| Attribute | Input | Output |
|-----------|-------|--------|
| Columns | Index table scan columns (child node is IndexScan) | Defined by internal Project (columns needed from primary table) |
| Order | Any | No guarantee |
| Distribution | Any | Same as input |

---

## Output Format

```
LogicalTableLookup(<col1>="<expr1>", ..., condition="<join_cond>", type="<type>", removable="false")
```

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| Output column mapping | Required (N columns) | Key = output column name, value = expression. Key equals value for passthrough |
| condition | Required | Join condition between index table and primary table, typically primary key equi-join (`id = id`) |
| type | Required | `inner` (most common) / `left` |
| removable | When operators are pushed down to primary table | `removable="false"` means table lookup cannot be removed |

---

## Examples

```
LogicalTableLookup(id="id", name="name", age="age", condition="id = id", type="inner")
  IndexScan(tables="g_idx", ...)

LogicalTableLookup(id="id", age="age", condition="id = id", type="inner")
  IndexScan(tables="g_idx", ...)

LogicalTableLookup(id="id", name="name", condition="id = id", type="inner", removable="false")
  IndexScan(tables="g_idx", ...)
```

- Output columns include only query-required columns (column pruning)
- `removable="false"`: Filter or other operators pushed down to primary table side, table lookup cannot be eliminated
