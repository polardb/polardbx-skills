# Other — Miscellaneous Operator EXPLAIN Output Reference

Special operators that do not belong to the main categories.

---

## LogicalTableFunctionScan

Table function scan. Does not override `explainTermsForDisplay`, uses default implementation. Output columns are defined by the function return type.

---

## RuntimeFilterBuilder

```
RuntimeFilterBuilder(condition="<expr>")
```

Builds a Bloom Filter on the Join build side; the probe side uses this filter to pre-filter non-matching rows. Passes through all attributes (columns, order, distribution).

| Field | Description |
|-------|-------------|
| condition | Required. Key condition for building the runtime filter |

Example:

```
RuntimeFilterBuilder(condition="id = id")
  Exchange(...)
```
