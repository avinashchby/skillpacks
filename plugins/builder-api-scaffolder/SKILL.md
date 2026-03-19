---
name: builder-api-scaffolder
description: >
  Generates a complete CRUD API from a data model description. Triggered when
  the user says "scaffold api", "create crud", "build api for", "generate endpoints",
  or describes a resource and asks for routes/handlers. Detects the target framework
  (Express/Node.js, FastAPI/Python, Actix-web/Rust) from the project context, then
  produces route definitions, request/response types, input validation, error handling,
  and unit tests as separate files — ready to drop into the existing project structure.
version: 1.0.0
---

# API Scaffolder

## When to Use

Activate this skill when the user:

- Says "scaffold api for [resource]", "create crud for [model]", "build api for [entity]",
  or "generate endpoints for [resource]"
- Describes a data model and asks for routes, handlers, or controllers
- Wants to add a new resource to an existing REST or HTTP API
- Asks to "wire up" a database model to HTTP endpoints
- Requests CRUD operations without specifying exactly how to implement them

Do NOT activate for:
- GraphQL schema generation (different skill)
- Database migration alone (use builder-db-migration)
- Authentication middleware (use builder-auth-setup)

---

## Instructions

### Step 1 — Detect Framework and Project Layout

Before generating any code, scan the project to determine:

1. **Framework**: Check for `package.json` (Express/Fastify/Hono), `pyproject.toml`/`requirements.txt`
   (FastAPI/Flask/Django), or `Cargo.toml` (Actix-web/Axum).
2. **Language version**: Node version from `.nvmrc` or `engines` field; Python version from
   `.python-version` or `pyproject.toml`; Rust edition from `Cargo.toml`.
3. **Existing route structure**: Look for a `routes/`, `api/`, `src/routes/`, or `app/routers/`
   directory to match the naming convention already in use.
4. **Validation library in use**: `zod`, `joi`, `class-validator`, `pydantic`, `validator` crate.
5. **Test runner**: `jest`, `vitest`, `pytest`, `cargo test`.
6. **ORM/query layer**: Prisma, Drizzle, SQLAlchemy, sqlx, raw SQL helpers.

If the framework cannot be determined, ask the user before generating code.

### Step 2 — Parse the Data Model

Extract from the user's description:

- **Resource name** (singular noun, e.g., `product`, `invoice`, `user`)
- **Fields**: name, type, whether required, whether unique, any constraints (min/max, enum values)
- **Relations**: belongs-to, has-many (generate foreign key stubs only; full join queries are opt-in)
- **Soft delete**: if the model has a `deletedAt` field, use soft-delete semantics for DELETE routes

If any field type is ambiguous, default to `string` and leave a `// TODO: confirm type` comment.

### Step 3 — Generate Files

Produce the following files. Adjust paths to match the detected project layout.

#### 3a. Types / Schema file

Define request and response types.

- **TypeScript (Express)**: Use `zod` schemas, derive types with `z.infer<>`. Export both the
  schema and the type.
- **Python (FastAPI)**: Use Pydantic `BaseModel` classes — one for Create, one for Update
  (all fields optional), one for Response (includes `id` and timestamps).
- **Rust (Actix-web/Axum)**: Use `serde::{Deserialize, Serialize}` structs. Derive `Validate`
  from the `validator` crate for constraints.

#### 3b. Repository / Data access layer

Thin functions that execute queries. Accept a database client/pool as a parameter (dependency
injection, not a global). Return `Result` types — never panic or throw unhandled errors.

Functions to generate:
- `findById(id)` — returns `Option`/`None`/`404` if not found
- `findAll(filters, pagination)` — accepts `limit`, `offset`, optional filter fields
- `create(data)` — returns the created record
- `update(id, data)` — returns the updated record; errors if not found
- `remove(id)` — hard or soft delete per model definition; errors if not found

#### 3c. Route handlers / Controller

Map HTTP verbs to repository calls:

| Method   | Path          | Handler      |
|----------|---------------|--------------|
| GET      | /resources    | findAll      |
| GET      | /resources/:id| findById     |
| POST     | /resources    | create       |
| PATCH    | /resources/:id| update       |
| DELETE   | /resources/:id| remove       |

Each handler must:
1. Parse and validate the request body/params using the schema from 3a.
2. Call the repository function.
3. Return appropriate HTTP status codes: `200` (GET), `201` (POST), `204` (DELETE), `404`
   (not found), `422` (validation error), `500` (unexpected error).
4. Never leak internal error messages to the response; log them server-side.

#### 3d. Router registration

Export a router/blueprint/scope that can be mounted at a prefix. Add a comment showing the
mount point, e.g., `// Mount at: app.use('/api/v1/products', productRouter)`.

#### 3e. Tests

Generate one test file per layer:

- **Unit tests** for repository functions using a mock/in-memory database.
- **Integration tests** for route handlers using a test HTTP client (`supertest`, `httpx`,
  `actix-web::test`).

Test cases to include for each endpoint:
- Happy path (valid input, record exists)
- Not found (invalid ID)
- Validation failure (missing required field, wrong type)
- At least one edge case specific to the resource (e.g., duplicate unique field on create)

### Step 4 — Output Format

Present output as a file tree first, then each file in a fenced code block labeled with the
file path. Example:

```
src/
  api/
    products/
      products.schema.ts
      products.repository.ts
      products.routes.ts
      products.test.ts
```

After the files, add a **"Next steps"** section listing:
1. The exact import/mount line to add to the main app file
2. Any environment variables or dependencies to install
3. Any migrations needed (link to builder-db-migration skill)

---

## Examples

### Example 1 — Express + TypeScript + Prisma

**User input:**
> scaffold api for Product with fields: name (string, required), price (number, required, min 0),
> description (string, optional), categoryId (foreign key to Category)

**Generated structure:**

```
src/api/products/
  products.schema.ts
  products.repository.ts
  products.routes.ts
  products.test.ts
```

`products.schema.ts`:
```typescript
import { z } from 'zod';

export const createProductSchema = z.object({
  name: z.string().min(1),
  price: z.number().min(0),
  description: z.string().optional(),
  categoryId: z.string().uuid(),
});

export const updateProductSchema = createProductSchema.partial();

export type CreateProductInput = z.infer<typeof createProductSchema>;
export type UpdateProductInput = z.infer<typeof updateProductSchema>;
```

`products.repository.ts`:
```typescript
import { PrismaClient, Product } from '@prisma/client';
import type { CreateProductInput, UpdateProductInput } from './products.schema';

export async function findProductById(
  db: PrismaClient,
  id: string,
): Promise<Product | null> {
  return db.product.findUnique({ where: { id } });
}

export async function findAllProducts(
  db: PrismaClient,
  limit = 20,
  offset = 0,
): Promise<Product[]> {
  return db.product.findMany({ skip: offset, take: limit });
}

export async function createProduct(
  db: PrismaClient,
  data: CreateProductInput,
): Promise<Product> {
  return db.product.create({ data });
}

export async function updateProduct(
  db: PrismaClient,
  id: string,
  data: UpdateProductInput,
): Promise<Product> {
  return db.product.update({ where: { id }, data });
}

export async function removeProduct(
  db: PrismaClient,
  id: string,
): Promise<void> {
  await db.product.delete({ where: { id } });
}
```

**Next steps:**
- Run `npx prisma migrate dev --name add_products_table`
- Add `app.use('/api/v1/products', productRouter)` to `src/app.ts`

---

### Example 2 — FastAPI + SQLAlchemy

**User input:**
> create crud for Invoice with: amount (float, required), status (enum: draft/sent/paid),
> customerId (int, foreign key), dueDate (date, optional)

**Generated `invoices/schemas.py`:**
```python
from datetime import date
from enum import Enum
from typing import Optional
from pydantic import BaseModel, Field


class InvoiceStatus(str, Enum):
    draft = "draft"
    sent = "sent"
    paid = "paid"


class InvoiceCreate(BaseModel):
    amount: float = Field(gt=0)
    status: InvoiceStatus = InvoiceStatus.draft
    customer_id: int
    due_date: Optional[date] = None


class InvoiceUpdate(BaseModel):
    amount: Optional[float] = Field(default=None, gt=0)
    status: Optional[InvoiceStatus] = None
    due_date: Optional[date] = None


class InvoiceResponse(InvoiceCreate):
    id: int
    created_at: date

    class Config:
        from_attributes = True
```

---

### Example 3 — Axum + sqlx (Rust)

**User input:**
> build api for Tag: name (string, required, unique), color (hex string, optional)

**Generated `src/api/tags/mod.rs`** (excerpt):
```rust
use axum::{extract::{Path, State}, http::StatusCode, Json, Router};
use axum::routing::{get, post, patch, delete};
use sqlx::PgPool;
use validator::Validate;
use crate::errors::AppError;
use super::schema::{CreateTag, UpdateTag, TagResponse};
use super::repository;

pub fn router() -> Router<PgPool> {
    Router::new()
        .route("/tags", get(list_tags).post(create_tag))
        .route("/tags/:id", get(get_tag).patch(update_tag).delete(delete_tag))
}

async fn create_tag(
    State(pool): State<PgPool>,
    Json(body): Json<CreateTag>,
) -> Result<(StatusCode, Json<TagResponse>), AppError> {
    body.validate()?;
    let tag = repository::create(&pool, body).await?;
    Ok((StatusCode::CREATED, Json(tag)))
}
```

---

## Anti-patterns

1. **Generating a global database client singleton** — Always inject the database client/pool
   as a function parameter or router state. Global singletons make unit testing impossible and
   hide dependencies.

2. **Putting business logic in route handlers** — Handlers should only parse input, call a
   repository or service function, and format the response. Any computation, validation beyond
   schema, or cross-resource lookups belong in a service layer.

3. **Returning raw database errors to the client** — Error messages from Prisma, SQLAlchemy,
   or sqlx often include table names, column names, and query fragments. Always map them to
   generic user-facing messages and log the original server-side.

4. **Skipping the Update partial-type** — Using the same schema for Create and Update means
   PATCH requests require all fields, breaking partial updates. Always generate a separate
   schema with all fields optional for Update.

5. **Omitting pagination on list endpoints** — Returning all rows from a table without
   `limit`/`offset` or cursor pagination will cause performance problems in production.
   Always default to a reasonable page size (20–100) and document the query parameters.
