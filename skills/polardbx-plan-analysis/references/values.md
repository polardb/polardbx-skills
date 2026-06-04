# Values — Values Operator EXPLAIN Output Reference

Leaf node with no input that produces row data by itself.

---

## Output Format

| Operator | Format | Description |
|----------|--------|-------------|
| LogicalValues | `Values(table="dual")` | Fixed output `dual`, does not display actual values |
| LogicalDynamicValues | `DynamicValues(tuples=[[?0, ?1], [?2, ?3]])` | Parameterized row list from INSERT VALUES |

---

## Examples

```
Project(1="?0", 'hello'="?1")
  Values(table="dual")

LogicalInsert(table="t", ...)
  DynamicValues(tuples=[[?0, ?1], [?2, ?3]])
```

- `Values(table="dual")`: Virtual table for SELECT with constants
- `DynamicValues(tuples=[...])`: VALUES part of INSERT, each inner list is one row
