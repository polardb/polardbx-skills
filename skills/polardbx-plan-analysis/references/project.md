# Project — Projection Operator EXPLAIN Output Reference

Changes the column space: output column count and values are entirely determined by the projects expression list. LogicalProject and PhysicalProject share the same output format.

---

## Input / Output Attributes

| Attribute | Input | Output |
|-----------|-------|--------|
| Columns | Any | Redefined by projects (can add/remove/reorder/compute new columns) |
| Order | Any | May change (lost when sort columns are removed or rewritten) |
| Distribution | Any | May change (invalidated when distribution keys are removed or rewritten) |

---

## Output Format

```
Project(<col_1>="<expr_1>", <col_2>="<expr_2>", ..., cor=[$cor0])
```

Each output column corresponds to a key-value pair (dynamic count).

| Field | Output Condition | Description |
|-------|-----------------|-------------|
| Output column mapping | Required (N columns) | Key = output column name, value = computation expression (RexNode -> SQL-style string) |
| cor | When variablesSet is non-empty | Referenced outer correlation variables |

---

## Expression Forms

| Scenario | Key | Value | Description |
|----------|-----|-------|-------------|
| Column passthrough | `id` | `id` | Key and value are the same |
| Alias | `user_id` | `id` | Key is the alias |
| Expression | `total` | `price * quantity` | Computation |
| Function | `name_upper` | `UPPER(name)` | Function call |
| Constant | `flag` | `?0` | Parameterized literal |
| Correlation variable | `outer_id` | `$cor0.id` | Outer correlated field |

---

## Examples

```
Project(id="id", name="name", age="age")
Project(user_id="id", full_name="CONCAT(first_name, ' ', last_name)", is_adult="age >= 18")
Project(id="id", flag="?0", status="?1")
Project(id="id", outer_val="$cor0.value", cor=[$cor0])
Project(v1="v1", v2="v2", v3="v3", v4="v4")
```

- Key = value means column passthrough (identity projection, usually eliminated by ProjectRemoveRule)
- `?N` = parameterized constant
- `$cor0.col` = outer correlation variable reference, accompanied by `cor=[...]`
