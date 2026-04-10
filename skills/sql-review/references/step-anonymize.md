# Step 3: Name Anonymization

**Purpose**: Prevent business table/column names from leaking to the test instance. All SQL sent to the test instance uses randomized names only.

## 3.1 Build Mapping

Generate random aliases for all table and column names from the Step 2 inventory:

| Type | Rule | Example |
|------|------|---------|
| Table | `t_` + 4 random alphanumeric chars | `orders` -> `t_a3kf` |
| Column | `c_` + 4 random alphanumeric chars | `user_id` -> `c_m9x2` |
| Primary key | Always `c_pk` | `id` -> `c_pk` |
| Index | `idx_` + 4 random chars | `idx_status_created` -> `idx_r4k1` |

When MyBatis is detected, also anonymize statement IDs:

| Type | Rule | Example |
|------|------|---------|
| Statement ID | `m_` + 4 random alphanumeric chars | `OrderMapper.findOrderList` -> `m_k3f9` |

The same column name across different tables uses the same anonymous name. Ensure no collisions.

## 3.2 Persist Mapping File

Save the mapping to `$TMPDIR/sql_review_name_mapping.md`:

```markdown
# SQL Review Name Mapping

## Tables
| Real Name | Anon Name |
|-----------|-----------|
| orders    | t_a3kf    |
| users     | t_b7mq    |

## Columns
| Real Name   | Anon Name |
|-------------|-----------|
| id          | c_pk      |
| user_id     | c_m9x2    |
| created_at  | c_p4wn    |

## MyBatis Statements (if applicable)
| Real Name | Anon Name |
|-----------|-----------|
| OrderMapper.findOrderList | m_k3f9 |
```

> **Critical**: In subsequent steps (Step 4/5/6), **always re-read this file** when the mapping is needed. Do NOT rely on conversation context memory. This prevents mapping loss in long sessions where context may be truncated.

## 3.3 Transform SQL

Replace all table/column/index names in DDL and queries with anonymous names. Preserve data types and constraints (NOT NULL, UNIQUE, DEFAULT, etc.) unchanged.

The transformed SQL is used for all subsequent interactions with the test instance.
