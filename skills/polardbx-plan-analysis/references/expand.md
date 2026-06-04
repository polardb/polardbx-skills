# Expand — Expand Operator EXPLAIN Output Reference

Used for GROUPING SETS / ROLLUP / CUBE, expands one row into multiple rows, each corresponding to a grouping set.

---

## Input / Output Attributes

| Attribute | Input | Output |
|-----------|-------|--------|
| Columns | Any | Input columns + groupId column |
| Order | Any | No guarantee |
| Distribution | Any | Same as input |

---

## Output Format

```
Expand(projects="{col1=col1, col2=null, groupId=0}, {col1=null, col2=col2, groupId=1}")
```

---

## Field Descriptions

### projects (Required)

Each `{...}` corresponds to the expansion rule for one grouping set, multiple separated by commas.

Format: `{<output_column_name>=<expression>, ...}`

Expression patterns:
- Participating columns: Retain original value (`col1=col1`)
- Non-participating columns: Set to null (`col2=null`)
- groupId column: Group number (`groupId=0`)

---

## Examples

### GROUPING SETS

```
HashAgg(group="dept,region,groupId", SUM(salary)="SUM(salary)")
  Expand(projects="{dept=dept, region=null, salary=salary, groupId=0}, {dept=null, region=region, salary=salary, groupId=1}")
    LogicalView(...)
```

groupId=0 groups by dept (region=null), groupId=1 groups by region (dept=null).

### ROLLUP

```
HashAgg(group="dept,region,groupId", SUM(salary)="SUM(salary)")
  Expand(projects="{dept=dept, region=region, salary=salary, groupId=0}, {dept=dept, region=null, salary=salary, groupId=1}, {dept=null, region=null, salary=salary, groupId=3}")
    LogicalView(...)
```

Three grouping sets: (dept, region), (dept), () — three-level aggregation.
