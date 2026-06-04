# SQL Parameterization — Literal Replacement Rules

PolarDB-X replaces SQL literals with `?` placeholders via `DrdsParameterizeSqlVisitor` before entering the optimizer, extracting values into a parameter list. The parameterized SQL is used for Plan Cache matching, so `?0`, `?1`, etc. seen in EXPLAIN output typically originate from this stage.

---

## Parameterization Scope

### Elements That Are Parameterized

| SQL Element | After Parameterization | Value in Parameter List |
|-------------|----------------------|------------------------|
| Integer literal (`42`) | `?` | `Long` or `BigInteger` |
| Decimal literal (`3.14`) | `?` | `BigDecimal` |
| String literal (`'hello'`) | `?` | `String` |
| Hex literal (`X'FF'`) | `?` | `byte[]` |
| Charset string (`_utf8'abc'`) | `?` | `NlsString` (carries charset and collation info) |
| `NOW()` | `cast(? as datetime)` | `ConstantVariable("NOW", args)`, args carries precision |
| `LAST_INSERT_ID()` | `?` | `SysDefVariable("LAST_INSERT_ID")` |
| System variable (`@@version`) | `?` | `SysDefVariable(variable_name, isGlobal)` |
| User variable (`@my_var`) | `?` | `UserDefVariable(variable_name)` |
| PreparedStatement `?` | `?` (unchanged) | `PreparedParamRef(index)` |

### Elements That Are NOT Parameterized

| SQL Element | Reason |
|-------------|--------|
| Column names, table names, aliases | Identifiers are not parameterized |
| Negative integers (`-42`) | `-` and integer are preserved as a whole, not split for parameterization |
| `INTERVAL` expression values and units | Both `1` and `DAY` in `INTERVAL 1 DAY` are preserved as-is |
| Window function `OVER` clause | Content within `OVER(PARTITION BY ... ORDER BY ...)` is not parameterized |
| Certain time function arguments | Arguments of `CURTIME`, `CURRENT_TIME`, `CURRENT_TIMESTAMP`, `LOCALTIME`, `LOCALTIMESTAMP`, `SYSDATE`, `UTC_DATE`, `UTC_TIME`, `UTC_TIMESTAMP` are not parameterized |
| Non-DML/DQL statements | `SET`, DDL, and similar statements do not go through parameterization |

---

## IN List Special Handling

In default mode, multiple values in an IN list are **merged** into a single `?`, with a `List` in the parameter list:

```sql
-- Original SQL
SELECT * FROM t WHERE id IN (1, 2, 3)
-- After parameterization
SELECT * FROM t WHERE id IN (?)
-- Parameter list: [List(1, 2, 3)]
```

In Prepare mode (`isPrepare=true`), IN list values are not merged — each value is parameterized independently:

```sql
-- After parameterization
SELECT * FROM t WHERE id IN (?, ?, ?)
-- Parameter list: [1, 2, 3]
```

---

## Parameter Numbering

`?0`, `?1`, `?2` shown in EXPLAIN output are **zero-based indices** into the parameter list. Numbers are assigned in left-to-right order of literal appearance in the SQL.

```sql
-- Original SQL
SELECT * FROM t WHERE a = 10 AND b = 'hello' AND c > 3.14
-- In parameterized EXPLAIN output
Filter(condition="a = ?0 AND b = ?1 AND c > ?2")
-- Parameter list: [10, 'hello', 3.14]
```

---

## Impact on EXPLAIN Output

Parameterization occurs at the optimizer entry point, so throughout the entire plan tree in EXPLAIN output, all original SQL literals have been replaced with `?N` form. This means:

- Filter `condition` shows `?0` instead of original literals
- Project expression columns show `?0` instead of constant values
- LogicalView `sql` field contains the parameterized form of pushed-down SQL
- JOIN `condition` typically contains column references (not involving parameterization), but literals in ON conditions will also be parameterized
