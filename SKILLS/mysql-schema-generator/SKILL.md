---
name: mysql-schema-generator
description: Generates MySQL CREATE TABLE statements and fetch queries from entity or
  model definitions, using the minimum correct column length, proper indexes, and
  utf8mb4 / utf8mb4_unicode_ci defaults. Also handles complex fetch queries with a
  joins-first strategy and recursive CTE fallback. This skill should be used when the
  user asks for a table, DDL, schema, or "entity ka table bana do", "model ka table
  bana do", "create table query", "mysql schema", or a complex join query.
agent_created: true
allowed-tools: Read, Grep, Glob, Bash, Write
---

# MySQL schema and query generator

## When to use

- The user asks for a `CREATE TABLE` query for an entity, model, or class.
- The user asks for a schema, DDL, or migration SQL.
- The user asks for a complex fetch query that spans multiple tables.

## Assumptions

State these and proceed. If the project differs, follow the project.

- **MySQL 8.0 is the default target.** `WITH RECURSIVE`, window functions, CTEs, and
  `JSON_TABLE` are all available - use them when they are the right tool. Do not hold back
  for older-version compatibility unless the project explicitly pins MySQL 5.7 or older.
  If a project does pin an older version, say so in the output and offer a rewritten
  variant instead of silently downgrading the SQL.
- InnoDB storage engine.
- Table names: plural, snake_case (`user_profiles`).
- Primary key: `id BIGINT UNSIGNED NOT NULL AUTO_INCREMENT`.
- Foreign key columns: `<entity>_id BIGINT UNSIGNED`.

## Rule 1: charset and collation

Every `CREATE TABLE` ends with:

```sql
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;
```

- Never use `utf8` or `utf8mb3` - both are legacy and cannot store 4-byte characters.
- Never use `utf8mb4_general_ci` unless the user explicitly asks for it.
- Apply the same charset to any database-level statement.

## Rule 2: minimum correct column length

Never default to `VARCHAR(255)`. Pick the smallest type that safely fits the domain.

| Data | Type to use |
| --- | --- |
| Indian mobile number | `CHAR(10)` |
| Email | `VARCHAR(254)` |
| Indian pincode | `CHAR(6)` |
| ISO country code | `CHAR(2)` |
| ISO currency code | `CHAR(3)` |
| UUID | `CHAR(36)` |
| Bcrypt password hash | `CHAR(60)` |
| IPv4 / IPv6 address | `VARCHAR(45)` |
| Slug, username, code | `VARCHAR(50)` |
| Person or product name | `VARCHAR(100)` |
| Title | `VARCHAR(150)` |
| Short description | `VARCHAR(500)` |
| Long text | `TEXT` |
| URL or file path | `VARCHAR(2048)` |
| Money | `DECIMAL(12,2)` |
| Percentage or rate | `DECIMAL(5,2)` |
| Quantity, counts | `INT UNSIGNED` |
| Small closed set | `ENUM('PENDING','APPROVED','REJECTED')` |
| Boolean flag | `TINYINT(1)` |
| Created / updated time | `DATETIME` |
| Date only | `DATE` |
| Semi-structured payload | `JSON` |

Additional rules:

- Use `CHAR` for fixed-length values, `VARCHAR` for variable-length values.
- Use `UNSIGNED` for every numeric column that cannot be negative.
- `NOT NULL` is the default. Allow `NULL` only with a stated reason.
- Prefer `ENUM` over `VARCHAR` when the value set is closed and stable.

## Rule 3: add indexes by default

Every table gets:

1. `PRIMARY KEY (id)`.
2. `UNIQUE` on every natural or business key - email, mobile, code, slug, external id.
3. An index on **every** foreign key column.
4. A composite index for the most common filter order, respecting the leftmost prefix rule.
5. An index on any column used regularly in `WHERE`, `ORDER BY`, or `GROUP BY`.

Naming convention:

- `uq_<column>` for unique indexes.
- `idx_<column>` or `idx_<col1>_<col2>` for ordinary indexes.

Limits:

- Do not index a low-cardinality column on its own (boolean, status). Include it in a
  composite index instead, placed after the higher-cardinality column.
- Keep the table to roughly six indexes. More than that needs justification.

## Rule 4: timestamps

```sql
created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
```

Index `created_at` only when queries actually filter or sort by it.

## Rule 5: foreign keys

```sql
user_id BIGINT UNSIGNED NOT NULL,
KEY idx_orders_user_id (user_id),
CONSTRAINT fk_orders_user_id FOREIGN KEY (user_id) REFERENCES users (id) ON DELETE RESTRICT
```

- Default to `ON DELETE RESTRICT`. Use `CASCADE` only when child rows have no meaning
  without the parent.
- Always declare the index alongside the constraint.

## Rule 6: query strategy - joins first, recursive only when needed

Follow this order:

1. **Prefer plain joins.** Use explicit `INNER JOIN` / `LEFT JOIN` with an `ON` clause.
2. **Keep the join list small.** Select only the needed columns, filter early on indexed
   columns, and push conditions into the `ON` clause for outer joins.
3. **Switch to a recursive CTE when the data is hierarchical** - category trees, org
   charts, comment threads, menus, nested regions - where the depth is not known upfront.
4. **Switch when the join count explodes.** If the query needs more than roughly five or
   six tables, or the same table must be joined repeatedly just to walk depth levels, use
   `WITH RECURSIVE` instead.
5. **Always state the reason** for the switch in one line.

Recursive template:

```sql
WITH RECURSIVE category_tree AS (
    SELECT id, parent_id, name, 0 AS depth
    FROM categories
    WHERE parent_id IS NULL

    UNION ALL

    SELECT c.id, c.parent_id, c.name, ct.depth + 1
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
    WHERE ct.depth < 10
)
SELECT id, parent_id, name, depth
FROM category_tree
ORDER BY depth, id;
```

Always include a depth guard such as `WHERE ct.depth < 10` to prevent runaway recursion on
cyclic data.

## Anti-patterns to avoid

- `SELECT *` in application queries.
- Implicit joins written as comma-separated tables in `FROM`.
- Any join without an `ON` clause.
- Wrapping an indexed column in a function inside `WHERE` - it kills the index.
- Leading wildcard `LIKE '%value%'` on large tables.
- `VARCHAR(255)` used as a lazy default.

## Output format

For a `CREATE TABLE` request:

1. The complete `CREATE TABLE` statement with indexes declared inline.
2. A short **Notes** section listing: why each notable length was chosen, and why each
   index exists.
3. Any assumption that needs the user's confirmation.

Keep the notes brief - one line per decision, not paragraphs.

## Language

SQL identifiers, comments, and all written output stay in English.
