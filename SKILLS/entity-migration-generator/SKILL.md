---
title: "Entity Migration Generator"
summary: "Automatically generate MySQL migration SQL from JPA entity changes and save to the migrations folder."
agent_created: true
---

# Entity Migration Generator

## When to use

- User creates a new `@Entity` class.
- User modifies an existing entity: adds/removes/renames fields, changes column definitions, adds indexes, etc.
- User explicitly asks to generate a migration for an entity.
- Any time a JPA entity in `src/main/java/**/jpa/model/` changes.

## Goal

Produce a Flyway-compatible MySQL migration file in `migrations/VYYYY_MM_DD__description.sql` that exactly matches the entity change.

## Step-by-step workflow

### 1. Read the entity source

Use `Read` to fetch the `.java` file(s) involved. Parse:

| Annotation | What to capture |
|---|---|
| `@Entity` | Confirms this is an entity. |
| `@Table(name = "...")` | Table name. If missing, derive: plural + snake_case of class name. |
| `@Id` | Primary key column. |
| `@GeneratedValue(strategy = IDENTITY)` | Auto-increment PK. |
| `@Column(name = "...", length = N, nullable = false, unique = true)` | Column name, length, nullability, uniqueness. |
| `@ManyToOne` / `@OneToMany` / `@JoinColumn` | Foreign key columns and relationships. |
| `@Enumerated(EnumType.STRING)` | Store enum as `VARCHAR` with length = longest enum name. |
| `@CreationTimestamp` / `@UpdateTimestamp` / `createdAt` field | Add `created_at` / `updated_at` columns if present. |

### 2. Determine operation type

| Scenario | SQL to generate |
|---|---|
| Brand new entity | `CREATE TABLE` |
| Existing entity + new fields only | `ALTER TABLE ... ADD COLUMN` |
| Existing entity + modified columns | `ALTER TABLE ... MODIFY COLUMN` |
| Existing entity + removed columns | `ALTER TABLE ... DROP COLUMN` |
| Mixed changes | Combine `ADD`, `MODIFY`, `DROP` in one `ALTER TABLE` |

### 3. Java-to-MySQL type mapping

Follow these exact mappings:

| Java type | MySQL type | Notes |
|---|---|---|
| `String` + `length = N` | `VARCHAR(N)` | |
| `String` (no length) | `VARCHAR(100)` for names/titles, `TEXT` for long content | Infer from field name |
| `String` (UUID / ID field) | `CHAR(36)` or `VARCHAR(50)` | Use `CHAR(36)` only if storing standard UUID |
| `Boolean` | `TINYINT(1)` | |
| `Integer` | `INT` | Use `INT UNSIGNED` if value cannot be negative |
| `Long` + `@GeneratedValue(IDENTITY)` | `BIGINT UNSIGNED NOT NULL AUTO_INCREMENT` | PK |
| `Long` (non-PK) | `BIGINT` / `BIGINT UNSIGNED` | |
| `Double` | `DOUBLE` | |
| `Float` | `FLOAT` | |
| `LocalDateTime` | `DATETIME` | Add `DEFAULT CURRENT_TIMESTAMP` / `ON UPDATE CURRENT_TIMESTAMP` for audit fields |
| `LocalDate` | `DATE` | |
| `BigDecimal` | `DECIMAL(12,2)` | Use for money |
| `byte[]` / `Blob` | `BLOB` / `LONGBLOB` | |

### 4. Nullability & defaults

- If `@Column(nullable = false)` → `NOT NULL`
- If `nullable` is absent or `true` → allow `NULL`
- For `Boolean` fields without explicit default: add `DEFAULT 0` or `DEFAULT 1` based on domain (ask if unsure)
- For audit fields:
  ```sql
  created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
  updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
  ```

### 5. Indexes and constraints

- Always declare `PRIMARY KEY (id)` for identity PKs.
- Add `UNIQUE` for `@Column(unique = true)`.
- Add `KEY idx_<column> (<column>)` for foreign key columns.
- Add composite indexes for common query patterns if known.

### 6. File naming and location

- **Folder**: `migrations/` (at project root, sibling to `src/`)
- **Format**: `VYYYY_MM_DD__description.sql`
  - `YYYY_MM_DD` = today's date
  - `description` = short snake_case summary, e.g., `add_user_table`, `add_order_status_field`
- **Example**: `V2026_09_09__add_payment_gateway_config.sql`

### 7. SQL output format

Every migration file must include:
1. A header comment with description
2. The forward migration SQL
3. A commented-out rollback section

```sql
-- ==============================================================================
-- Migration: V2026_09_09__add_example_table.sql
-- Description: Add example table for demo feature
-- ==============================================================================

CREATE TABLE `examples` (
    `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT,
    `name` VARCHAR(100) NOT NULL,
    `status` TINYINT(1) NOT NULL DEFAULT 0,
    `created_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
    `updated_at` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
    PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- ==============================================================================
-- ROLLBACK
-- ==============================================================================
-- DROP TABLE IF EXISTS `examples`;
```

For ALTER TABLE:
```sql
-- ==============================================================================
-- Migration: V2026_09_09__add_user_profile_fields.sql
-- ==============================================================================

ALTER TABLE `users`
    ADD COLUMN `profile_image` VARCHAR(2048) NULL AFTER `email`,
    ADD COLUMN `date_of_birth` DATE NULL AFTER `profile_image`;

-- ROLLBACK:
-- ALTER TABLE `users`
--     DROP COLUMN `profile_image`,
--     DROP COLUMN `date_of_birth`;
```

### 8. Execution order

After generating the SQL:
1. Call `Write` to save the file to `migrations/VYYYY_MM_DD__description.sql`
2. Present the file path and a preview of the SQL to the user
3. Ask user to review before applying

## Important notes

- Always use backticks for table/column names.
- Always end `CREATE TABLE` with `ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;`
- Never use `utf8` or `utf8mb3` — only `utf8mb4`.
- For enum-like String fields, prefer `ENUM('VALUE1','VALUE2')` only if the set is truly closed and stable. Otherwise use `VARCHAR` with a CHECK constraint (MySQL 8.0.16+).
- If multiple entities change in one session, generate **one migration file per logical change** or **one per entity** — keep them focused and reviewable.
