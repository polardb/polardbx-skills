# Filter — Filter Operator EXPLAIN Output Reference

Pure row-filtering operator that only reduces row count without changing column, order, or distribution attributes. LogicalFilter and PhysicalFilter share the same output format.

---

## Input / Output Attributes

| Attribute | Input | Output |
|-----------|-------|--------|
| Columns | Any | Same as input |
| Order | Any | Same as input |
| Distribution | Any | Same as input |

---

## Output Format

```
Filter(condition="<expr>", cor=[$cor0, $cor1])
```

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| condition | Required | Filter condition, RexNode converted to SQL-style string (rules in `rex-explain-visitor.md`) |
| cor | When variablesSet is non-empty | Referenced outer correlation variables (appears before subquery decorrelation) |

---

## Examples

```
Filter(condition="id > 10")
Filter(condition="v1 > 10 AND v2 = 'abc' AND (v3 < 5 OR v3 > 100)")
Filter(condition="id = ?0 AND status = ?1")
Filter(condition="$cor0.id = id", cor=[$cor0])
Filter(condition="SUBSTR(name, 1, 3) = 'abc'")
```

- AND is flat-joined, nested OR gets parentheses
- `?N` are parameterized literals
- `$cor0.id` references an outer correlation variable column
