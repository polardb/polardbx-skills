# MyBatis Dynamic SQL Reference

This document applies when MyBatis mapper XML files are detected in the project (Step 1.0). It defines how to expand dynamic SQL tags into executable SQL for index analysis.

## 1. Dynamic Tag Expansion Rules

When expanding dynamic SQL, apply rules recursively from innermost tags outward. The primary variant assumes all `<if>` conditions are true (maximum expansion).

### `<if test="condition">`

Conditionally includes its body when the test expression evaluates to true.

**Expansion**: Include the body text for the primary variant (assume condition is true). For multi-path analysis (Section 3), selectively include/exclude based on path definition.

```xml
<!-- Input -->
WHERE is_deleted = 0
  <if test="userId != null">AND user_id = #{userId}</if>
  <if test="status != null">AND status = #{status}</if>

<!-- Primary variant (all true) -->
WHERE is_deleted = 0 AND user_id = ? AND status = ?

<!-- Minimum variant (all false) -->
WHERE is_deleted = 0
```

### `<where>`

Emits `WHERE` keyword only if at least one child produces content. Strips leading `AND` or `OR` from the first active child.

**Expansion**: Emit `WHERE`, then include all children. Strip the leading `AND`/`OR` from the first child that produces output.

```xml
<!-- Input -->
<where>
  <if test="name != null">AND name = #{name}</if>
  <if test="status != null">AND status = #{status}</if>
</where>

<!-- Primary variant -->
WHERE name = ? AND status = ?

<!-- If only status is active -->
WHERE status = ?
```

### `<set>`

Emits `SET` keyword for UPDATE statements. Strips trailing comma from the last active child.

**Expansion**: Emit `SET`, include all children, strip trailing comma. Used in `updateByPrimaryKeySelective` patterns — these are typically skipped for index analysis (UPDATE by PK).

```xml
<!-- Input -->
<set>
  <if test="name != null">name = #{name},</if>
  <if test="status != null">status = #{status},</if>
</set>

<!-- Expanded -->
SET name = ?, status = ?
```

### `<foreach>`

Iterates over a collection, producing repeated SQL fragments with configurable separators.

**Expansion**: Generate 3 iterations to represent a realistic list.

```xml
<!-- Input -->
WHERE id IN
<foreach collection="ids" item="id" open="(" separator="," close=")">
  #{id}
</foreach>

<!-- Expanded -->
WHERE id IN (?, ?, ?)
```

For batch INSERT:
```xml
<!-- Input -->
INSERT INTO t (a, b) VALUES
<foreach collection="list" item="item" separator=",">
  (#{item.a}, #{item.b})
</foreach>

<!-- Expanded -->
INSERT INTO t (a, b) VALUES (?, ?), (?, ?), (?, ?)
```

### `<choose>` / `<when>` / `<otherwise>`

Switch-case logic. Only the first matching `<when>` branch executes; if none match, `<otherwise>` executes.

**Expansion**: For the primary variant, use the first `<when>` body. Generate additional variants for other `<when>` branches and `<otherwise>` if they produce structurally different SQL (different ORDER BY, different WHERE columns).

```xml
<!-- Input -->
<choose>
  <when test="orderBy == 'price'">ORDER BY selling_price ASC</when>
  <when test="orderBy == 'stock'">ORDER BY stock_num DESC</when>
  <otherwise>ORDER BY goods_id DESC</otherwise>
</choose>

<!-- Primary variant -->
ORDER BY selling_price ASC

<!-- Note: if all branches are just ORDER BY variations on the same table,
   they don't affect index WHERE analysis. Only generate separate variants
   when branches change WHERE/JOIN structure. -->
```

**Important**: If `<choose>` branches only differ in ORDER BY columns, note all ordering options in Key Features but do NOT create separate query variants — they share the same WHERE clause and index requirements. Only create variants when branches change the WHERE/JOIN structure.

### `<trim>`

General-purpose whitespace and keyword trimmer. Attributes:
- `prefix`: prepended string
- `suffix`: appended string
- `prefixOverrides`: pipe-separated prefixes to strip from the first child's output
- `suffixOverrides`: pipe-separated suffixes to strip from the last child's output

**Expansion**: Apply prefix/suffix and override rules textually. Most common in `insertSelective` patterns:

```xml
<!-- Input -->
<trim prefix="(" suffix=")" suffixOverrides=",">
  <if test="name != null">name,</if>
  <if test="status != null">status,</if>
</trim>

<!-- Expanded (all conditions true) -->
(name, status)
```

### `<include refid="...">`

References a reusable `<sql>` fragment by ID.

**Expansion**: Replace with the content of the referenced `<sql id="...">` block. See Section 2.

### `<sql id="...">`

Defines a reusable SQL fragment. Not a standalone statement.

**Handling**: Collect into fragment map during Step 1.2.1. Used for `<include>` resolution in Step 1.2.2.

### `<bind>`

Creates a local variable for use in subsequent expressions.

**Expansion**: Ignore for SQL structure analysis. Note that bound variables may appear in LIKE clauses or other expressions.

```xml
<bind name="pattern" value="'%' + name + '%'"/>
WHERE name LIKE #{pattern}

<!-- Expanded -->
WHERE name LIKE ?
```

### `<selectKey>`

Auxiliary query for key generation (e.g., fetching sequence values before INSERT).

**Handling**: Skip for index analysis — it's a separate auxiliary query, not part of the main statement.

## 2. Fragment Resolution (`<sql>` / `<include>`)

**Algorithm** (must execute before dynamic tag expansion):

1. Read ALL mapper XML files in the project
2. Build a global fragment map: `{ "namespace.fragmentId" => content, "fragmentId" => content }`
3. For each SQL statement, replace `<include refid="X"/>` with the fragment content:
   - First try `currentNamespace.X`
   - Then try `X` as a global lookup
   - If the refid contains a dot (e.g., `com.example.CommonMapper.baseColumns`), use it as a fully qualified key
4. Apply resolution recursively (fragments can include other fragments), with a depth limit of 5

**Common pattern**: `Base_Column_List` fragment used in SELECT statements:
```xml
<sql id="Base_Column_List">
  order_id, order_no, user_id, total_price, order_status, create_time
</sql>

<select id="findOrders">
  SELECT <include refid="Base_Column_List"/> FROM orders WHERE ...
</select>

<!-- After resolution -->
SELECT order_id, order_no, user_id, total_price, order_status, create_time FROM orders WHERE ...
```

## 3. Multi-Path Analysis for Dynamic WHERE

### When to Apply

Only apply multi-path analysis when ALL of these conditions are met:
- The statement is a SELECT (or COUNT) query
- It has `<if>` or `<choose>` tags inside the WHERE clause
- The conditional tags affect **which columns appear in WHERE** (not just values)
- There are at least 2 optional conditions

Do NOT apply for:
- `<if>` in SELECT column list (doesn't affect indexing)
- `<if>` in SET clause of UPDATE
- `<foreach>` (always same WHERE shape, different list length)
- `insertSelective` / `updateByPrimaryKeySelective` patterns

### Algorithm

#### Step A: Classify Parameters

Extract parameter names from `<if test="...">` expressions and classify:

| Category | Pattern | Examples | Significance |
|----------|---------|----------|-------------|
| **Identity** | userId, tenantId, orgId, shopId, merchantId | `<if test="userId != null">` | Present = user-scoped query; absent = admin/system query |
| **Filter** | status, type, category, level, isDeleted | `<if test="orderStatus != null">` | Optional refinement, affects selectivity |
| **Search** | keyword, name, orderNo, email, phone | `<if test="goodsName != null">` | Highly selective when present |
| **Range** | startTime/endTime, startDate/endDate, minPrice/maxPrice | `<if test="startTime != null">` | Usually paired, creates range scan |

#### Step B: Generate Paths (max 4)

| Priority | Path | Rule | Business Meaning |
|----------|------|------|-----------------|
| 1 (always) | **Full** | All `<if>` conditions true | Best case — most specific query |
| 2 (always) | **Minimum** | All `<if>` conditions false, only fixed WHERE clauses | Worst case — broadest scan |
| 3 (if applicable) | **No-Identity** | Remove Identity params, keep others | Admin/system view without user scoping |
| 4 (if applicable) | **Key-Branch** | For `<choose>`: most structurally different branch | Alternative execution path |

#### Step C: Deduplicate

If two paths produce identical WHERE column sets, merge them (keep the more descriptive label).

Example: If the only `<if>` conditions are Identity params, "Minimum" and "No-Identity" paths are identical — merge into one.

#### Step D: Cap at 4

If more than 4 paths remain, keep in priority order: Full > Minimum > No-Identity > Key-Branch.

### Naming Convention

Each variant gets a sub-letter: `Q6a`, `Q6b`, `Q6c`, `Q6d`, with a descriptive label:

- `Q6a (all filters)` — full path
- `Q6b (admin: no userId)` — no-identity path
- `Q6c (minimum: isDeleted only)` — minimum path

### How Variants Feed Into Downstream Steps

| Step | Behavior |
|------|----------|
| Step 1.5 (Anonymization) | All variants share the same table/column mapping |
| Step 2 (Data Characteristics) | No change — characteristics are per-table |
| Step 4 (Mock Data) | No change — same mock data serves all variants |
| Step 5 (EXPLAIN) | Each variant EXPLAINed independently; candidate indexes tested against ALL variants of the same query |
| Step 6 (Report) | Variants grouped together in a comparison table |

## 4. Parameter Placeholder Rules

| Syntax | Meaning | Expansion | Security |
|--------|---------|-----------|----------|
| `#{param}` | Prepared statement bind variable | Replace with `?` | Safe |
| `#{param,jdbcType=VARCHAR}` | Bind variable with type hint | Replace with `?` (strip jdbcType) | Safe |
| `${param}` | String interpolation (direct text substitution) | Replace with a representative literal (e.g., `${tableName}` -> `t_example`, `${orderByClause}` -> `id ASC`) | **SQL injection risk** — flag in security warnings |

Special case: `${orderByClause}` or `${sortColumn}` means ORDER BY is fully dynamic. Note in caveats that index-based sort optimization may not be reliable for this query.

## 5. Worked Example: newbee-mall `findNewBeeMallOrderList`

### Original XML

```xml
<select id="findNewBeeMallOrderList" parameterType="Map" resultMap="BaseResultMap">
  select
  <include refid="Base_Column_List"/>
  from tb_newbee_mall_order
  where is_deleted = 0
  <if test="orderNo != null and orderNo != ''">and order_no = #{orderNo}</if>
  <if test="userId != null">and user_id = #{userId}</if>
  <if test="payType != null">and pay_type = #{payType}</if>
  <if test="orderStatus != null">and order_status = #{orderStatus}</if>
  <if test="isDeleted != null">and is_deleted = #{isDeleted}</if>
  <if test="startTime != null and startTime.trim() != ''">and create_time &gt; #{startTime}</if>
  <if test="endTime != null and endTime.trim() != ''">and create_time &lt; #{endTime}</if>
  order by create_time desc
  <if test="start!=null and limit!=null">limit #{start},#{limit}</if>
</select>
```

### Step 1: Fragment Resolution

`<include refid="Base_Column_List"/>` -> `order_id, order_no, user_id, total_price, pay_status, pay_type, pay_time, order_status, extra_info, user_name, user_phone, user_address, is_deleted, create_time, update_time`

### Step 2: Parameter Classification

| Parameter | Category | Reasoning |
|-----------|----------|-----------|
| orderNo | Search | Order number lookup, highly selective |
| userId | Identity | User scoping, admin won't pass this |
| payType | Filter | Optional payment type filter |
| orderStatus | Filter | Optional status filter |
| isDeleted | Filter | Usually fixed (overrides the default `is_deleted=0`) |
| startTime/endTime | Range | Date range filter |
| start/limit | Pagination | Not part of WHERE |

### Step 3: Path Generation

| Path | Label | Active Conditions | WHERE Columns |
|------|-------|-------------------|---------------|
| A | Full (user view) | All | order_no, user_id, pay_type, order_status, is_deleted, create_time range |
| B | Minimum (admin list) | None | is_deleted (fixed) |
| C | No-Identity (admin + filters) | payType, orderStatus, startTime/endTime | pay_type, order_status, is_deleted, create_time range |

Path D is skipped (no `<choose>` branches, and we've already captured the key scenarios).

### Step 4: Expanded SQL for Each Path

**Q6a (full):**
```sql
SELECT ... FROM tb_newbee_mall_order
WHERE is_deleted = 0 AND order_no = ? AND user_id = ?
  AND pay_type = ? AND order_status = ? AND create_time > ? AND create_time < ?
ORDER BY create_time DESC LIMIT ?, ?
```

**Q6b (minimum — admin list):**
```sql
SELECT ... FROM tb_newbee_mall_order
WHERE is_deleted = 0
ORDER BY create_time DESC LIMIT ?, ?
```

**Q6c (admin + filters):**
```sql
SELECT ... FROM tb_newbee_mall_order
WHERE is_deleted = 0 AND pay_type = ? AND order_status = ?
  AND create_time > ? AND create_time < ?
ORDER BY create_time DESC LIMIT ?, ?
```

### Step 5: Index Implications

- **Q6a**: `(user_id, order_status, is_deleted, create_time)` — ref, rows=1
- **Q6b**: No suitable index from Q6a. Needs `(is_deleted, create_time)` or just `(create_time)` for ORDER BY optimization
- **Q6c**: `(order_status, is_deleted, create_time)` could help; or the `(is_deleted, create_time)` index from Q6b also partially helps

Recommendation: Two indexes to cover all paths:
1. `(user_id, order_status, is_deleted, create_time)` — for Q6a
2. `(is_deleted, create_time)` — for Q6b and Q6c
