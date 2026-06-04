# OutFile — Output File Operator EXPLAIN Output Reference

Terminal operator for `SELECT ... INTO OUTFILE`, consumes input and writes to a file with no output rows.

---

## Output Format

```
LogicalOutFile(outfile name="<file_path>")
```

| Field | Description |
|-------|-------------|
| outfile name | Required. Export file path (specified by `INTO OUTFILE '<path>'` in SQL) |

EXPLAIN only displays the file path, does not display charset, delimiter, or other format parameters.

---

## Examples

```
LogicalOutFile(outfile name="/tmp/data.csv")
  LogicalView(...)
```
