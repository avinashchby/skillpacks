---
name: builder-db-migration
description: >
  Generates database migration files from a schema change description. Triggered when the user
  says "create migration", "db migration", "add column", "alter table", "schema change",
  "rename column", "drop table", or describes any structural database modification. Detects the
  ORM or migration tool in use (Prisma, Drizzle, SQLAlchemy/Alembic, Flyway, raw SQL), then
  generates both an up migration and a down (rollback) migration. Warns explicitly when the
  requested change is destructive — data loss, nullability change on existing data, or unique
  constraint on a populated column.
version: 1.0.0
---

# Database Migration Generator

## When to Use

Activate this skill when the user:

- Says "create migration", "generate migration", "db migration for [change]"
- Says "add column [name] to [table]", "add [field] to [model]"
- Says "alter table", "rename column", "drop column", "drop table"
- Describes a schema change and asks how to apply it
- Asks to "make [model] have [new field]" in the context of an existing database
- Wants to add an index, unique constraint, foreign key, or check constraint

Do NOT activate for:
- Initial schema creation on a brand-new project (use the api-scaffolder instead)
- Seeding data (a different concern; note it after migration if relevant)
- Query optimization without a schema change

---

## Instructions

### Step 1 — Detect Migration Tool and Database

Scan the project for:

1. **ORM / migration tool**:
   - `schema.prisma` → Prisma Migrate
   - `drizzle.config.ts` or `drizzle/` directory → Drizzle Kit
   - `alembic.ini` or `migrations/versions/` → Alembic (SQLAlchemy)
   - `db/migrate/` and `Gemfile` → Active Record (note: outside scope, inform user)
   - `flyway.conf` or `V__*.sql` files → Flyway
   - Plain `migrations/` directory with `.sql` files → Raw SQL migrations

2. **Database engine**: PostgreSQL, MySQL, SQLite, or other. Extract from the `DATABASE_URL`
   env var, the Prisma `datasource` block, or `SQLALCHEMY_DATABASE_URI`. This matters for
   syntax (e.g., `SERIAL` vs `AUTOINCREMENT`, `BOOLEAN` vs `TINYINT(1)`).

3. **Naming convention**: Check existing migration file names to match the pattern
   (timestamp prefix, sequential number, or descriptive slug).

4. **Current schema state**: Read the current schema file (if one exists) to understand the
   before-state. This is required to generate a correct down migration.

If the tool cannot be determined, ask before generating.

### Step 2 — Parse the Requested Change

Classify the change type and flag risk level:

| Change Type                          | Risk Level  | Down Migration Possible? |
|--------------------------------------|-------------|--------------------------|
| Add nullable column                  | Safe        | Yes — DROP COLUMN        |
| Add non-null column with default     | Safe        | Yes — DROP COLUMN        |
| Add non-null column without default  | DESTRUCTIVE | No — existing rows fail  |
| Drop column                          | DESTRUCTIVE | Only if data was saved   |
| Drop table                           | DESTRUCTIVE | Only if data was saved   |
| Rename column                        | Moderate    | Yes — rename back        |
| Add index                            | Safe        | Yes — DROP INDEX         |
| Add unique constraint (populated)    | Moderate    | Yes — DROP CONSTRAINT    |
| Add foreign key                      | Moderate    | Yes — DROP CONSTRAINT    |
| Change column type                   | Moderate    | Depends on types         |
| Change nullable to NOT NULL          | DESTRUCTIVE | Yes — DROP CONSTRAINT    |

For any DESTRUCTIVE change, emit a clearly visible warning block before the migration:

```
> WARNING: This migration is destructive.
> [Exact reason — e.g., "Adding a NOT NULL column without a default will fail if the table
> has existing rows."]
> Recommended: [Safe alternative, e.g., add as nullable first, backfill, then add constraint]
```

### Step 3 — Generate the Migration

#### Prisma

Update `schema.prisma` with the new field/model/relation. Then generate the migration SQL
by providing the command the user should run:

```bash
npx prisma migrate dev --name <slug>
```

Also show the preview of what Prisma would generate (the SQL statements), so the user can
review before running.

#### Drizzle

Update the schema file in `src/db/schema.ts` (or wherever the schema is). Generate the
migration file that `drizzle-kit generate` would produce, and show the command:

```bash
npx drizzle-kit generate --name <slug>
```

Include both the `sql` up statements and the snapshot diff.

#### Alembic (SQLAlchemy)

Generate a migration file at `migrations/versions/<timestamp>_<slug>.py` with:
- `upgrade()` function containing `op.*` calls
- `downgrade()` function that reverses them exactly
- Correct `revision`, `down_revision`, `branch_labels`, `depends_on` headers

#### Raw SQL

Generate two files:
- `migrations/<timestamp>_up_<slug>.sql` — the forward migration
- `migrations/<timestamp>_down_<slug>.sql` — the rollback

Include `BEGIN;` / `COMMIT;` transaction wrappers unless the change is not transactional
(e.g., `CREATE INDEX CONCURRENTLY` in PostgreSQL).

### Step 4 — Verify the Down Migration

For every up operation, confirm the inverse is included in the down migration:

- `ADD COLUMN x` → `DROP COLUMN x`
- `CREATE INDEX idx` → `DROP INDEX idx`
- `ADD CONSTRAINT fk` → `ALTER TABLE ... DROP CONSTRAINT fk`
- `RENAME COLUMN a TO b` → `RENAME COLUMN b TO a`

If a true down migration is impossible (column was dropped, data is gone), generate a
comment stub and warn the user explicitly.

### Step 5 — Output Format

1. If schema file needs updating (Prisma, Drizzle, SQLAlchemy model), show the diff.
2. Show the migration file(s) in full, in fenced code blocks.
3. Show the exact CLI command(s) to run to apply the migration.
4. List any manual steps (e.g., backfill nulls before adding NOT NULL constraint).
5. Confirm what the down/rollback command is.

---

## Examples

### Example 1 — Prisma: Add nullable column

**User input:**
> create migration to add `bio` (text, optional) to the User model

**Schema diff (`schema.prisma`):**
```diff
 model User {
   id        String   @id @default(cuid())
   email     String   @unique
   name      String
+  bio       String?
   createdAt DateTime @default(now())
 }
```

**Generated SQL (preview of what Prisma will create):**
```sql
-- migrations/20260319120000_add_bio_to_user/migration.sql
ALTER TABLE "User" ADD COLUMN "bio" TEXT;
```

**Command to run:**
```bash
npx prisma migrate dev --name add_bio_to_user
```

**Rollback** (handled by Prisma; manual equivalent):
```sql
ALTER TABLE "User" DROP COLUMN "bio";
```

---

### Example 2 — Alembic: Add index and foreign key

**User input:**
> db migration: add `category_id` integer foreign key to `products` table, with an index

**Generated `migrations/versions/20260319_130000_add_category_id_to_products.py`:**
```python
"""add category_id to products

Revision ID: a1b2c3d4e5f6
Revises: 9z8y7x6w5v4u
Create Date: 2026-03-19 13:00:00.000000
"""
from alembic import op
import sqlalchemy as sa

revision = 'a1b2c3d4e5f6'
down_revision = '9z8y7x6w5v4u'
branch_labels = None
depends_on = None


def upgrade() -> None:
    op.add_column(
        'products',
        sa.Column('category_id', sa.Integer(), nullable=True),
    )
    op.create_index(
        'ix_products_category_id',
        'products',
        ['category_id'],
    )
    op.create_foreign_key(
        'fk_products_category_id',
        'products', 'categories',
        ['category_id'], ['id'],
        ondelete='SET NULL',
    )


def downgrade() -> None:
    op.drop_constraint('fk_products_category_id', 'products', type_='foreignkey')
    op.drop_index('ix_products_category_id', table_name='products')
    op.drop_column('products', 'category_id')
```

**Command to run:**
```bash
alembic upgrade head
```

**Rollback:**
```bash
alembic downgrade -1
```

---

### Example 3 — Raw SQL: Destructive change warning

**User input:**
> alter table orders add column status VARCHAR(20) NOT NULL

**Warning:**
> WARNING: This migration is destructive.
> Adding a NOT NULL column without a default will fail at runtime if the `orders` table
> contains any existing rows.
> Recommended safe sequence:
> 1. Add the column as nullable.
> 2. Backfill existing rows with a default value.
> 3. Add the NOT NULL constraint in a follow-up migration.

**Generated `migrations/20260319_140000_up_add_status_to_orders.sql`:**
```sql
BEGIN;

-- Step 1: Add as nullable first (safe for existing rows)
ALTER TABLE orders ADD COLUMN status VARCHAR(20);

-- Step 2: Backfill — adjust the default value to match your business logic
UPDATE orders SET status = 'pending' WHERE status IS NULL;

-- Step 3: Apply NOT NULL constraint after backfill
ALTER TABLE orders ALTER COLUMN status SET NOT NULL;

COMMIT;
```

**Generated `migrations/20260319_140000_down_add_status_to_orders.sql`:**
```sql
BEGIN;
ALTER TABLE orders DROP COLUMN status;
COMMIT;
```

---

## Anti-patterns

1. **Generating destructive operations without a warning** — Dropping a column, making a
   nullable column NOT NULL, or adding a unique constraint on a populated column can destroy
   data or cause downtime. Always surface this with a visible warning and a safe alternative.

2. **Missing the down migration** — Every up migration must have a corresponding down migration
   unless it is explicitly impossible. A missing rollback turns a minor deploy issue into an
   incident. If a true rollback is impossible, say so and generate a comment stub.

3. **Using raw `ALTER TABLE` inside Prisma or Drizzle projects** — These tools track schema
   state in a manifest or shadow database. Bypassing them with raw SQL puts the migration
   history out of sync. Always go through the tool's designated migration mechanism.

4. **Generating a migration without reading the current schema** — Down migrations can only
   be generated correctly if you know the before-state. Guessing column types or nullability
   leads to broken rollbacks.

5. **Omitting transaction wrappers for raw SQL** — Without `BEGIN; ... COMMIT;`, a partially
   applied migration leaves the database in a broken intermediate state. Wrap all DDL in a
   transaction unless the operation explicitly cannot run inside one (e.g., `CREATE INDEX
   CONCURRENTLY`), and call that out as a comment.
