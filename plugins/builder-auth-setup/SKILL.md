---
name: builder-auth-setup
description: >
  Sets up an authentication system for a web application. Triggered when the user says
  "setup auth", "add authentication", "jwt auth", "oauth setup", "login system", "add login",
  "add signup", "protect routes", or "add refresh tokens". Generates token-based auth
  (JWT access token + refresh token rotation) or an OAuth 2.0 provider integration, plus
  middleware for route protection, a logout endpoint, and test coverage. Detects the
  framework and existing user model from the project before generating any code.
version: 1.0.0
---

# Authentication Setup

## When to Use

Activate this skill when the user:

- Says "setup auth", "add authentication", "implement login", "add jwt auth", "oauth setup"
- Asks to "protect routes", "add middleware to require login", or "add auth guard"
- Wants to "add signup/login endpoints"
- Asks for "refresh token" support or "token rotation"
- Wants to integrate a specific OAuth provider: GitHub, Google, Discord, etc.

Do NOT activate for:
- Role-based access control beyond simple authenticated/unauthenticated (RBAC is a separate concern)
- Session-based auth with server-side session storage (clarify with user; this skill targets stateless JWT)
- Full-stack auth services like Auth0, Clerk, or Supabase Auth (those have their own SDKs;
  offer to scaffold the integration instead)

---

## Instructions

### Step 1 — Detect Project Context

Before writing any code, determine:

1. **Framework and language**: Express/Node.js, FastAPI/Python, Actix-web/Axum/Rust,
   or other. Read `package.json`, `pyproject.toml`, or `Cargo.toml`.

2. **User model**: Search for an existing `User` model, table, or schema. Read it to
   understand available fields (id type, email field, password field, timestamps).

3. **Database access layer**: Prisma, Drizzle, SQLAlchemy, sqlx, or raw SQL. Auth functions
   must use the same access pattern as the rest of the project.

4. **Existing auth**: Check for any existing auth middleware, JWT packages, or session
   libraries already installed. Do not duplicate or conflict with them.

5. **Auth variant requested**:
   - **JWT (default)**: Access token (short-lived) + refresh token (long-lived, stored in DB)
   - **OAuth**: Provider name + whether to add a local account link

6. **Environment variable conventions**: Check `.env.example` or existing `config.*` files
   to understand the naming convention for secrets (e.g., `JWT_SECRET` vs `AUTH_SECRET`).

If the User model does not exist, note that it needs to be created and ask whether to generate
it (trigger builder-api-scaffolder for the User model scaffolding, then continue).

### Step 2 — Plan the Auth System

Describe the structure to the user before generating:

**For JWT auth:**
- `POST /auth/signup` — create user, hash password, return access + refresh tokens
- `POST /auth/login` — verify credentials, return access + refresh tokens
- `POST /auth/refresh` — validate refresh token, rotate to new access + refresh tokens
- `POST /auth/logout` — invalidate the refresh token
- Auth middleware — validates access token on protected routes
- Refresh token storage — `refresh_tokens` table (not stored in JWT payload)

**For OAuth:**
- `GET /auth/[provider]` — redirect to provider authorization URL
- `GET /auth/[provider]/callback` — handle callback, create/link user, issue tokens
- Same token issuance and middleware as JWT flow

Confirm the plan with the user if any ambiguity exists.

### Step 3 — Generate Files

#### 3a. Password Hashing Utility

Use a strong, modern algorithm. Do not invent crypto:
- **Node.js**: `bcrypt` (cost factor 12) or `argon2` (preferred if available)
- **Python**: `passlib[argon2]` or `bcrypt`
- **Rust**: `argon2` crate

Provide `hashPassword(plain)` and `verifyPassword(plain, hash)` functions.

#### 3b. Token Utilities

**Access token (JWT):**
- Algorithm: `HS256` minimum; prefer `RS256` if the project has key pairs
- Payload: `sub` (user id), `iat`, `exp`
- Expiry: 15 minutes (configurable via env var `ACCESS_TOKEN_EXPIRY`)
- Never include sensitive data (email, password hash, roles beyond basic) in the payload

**Refresh token:**
- A cryptographically random opaque string (not a JWT)
- Stored hashed in the `refresh_tokens` table alongside `userId`, `expiresAt`, `revokedAt`
- Expiry: 7 days (configurable via `REFRESH_TOKEN_EXPIRY`)
- Each use rotates the token (old one is revoked, new one is issued)
- Implement refresh token family tracking: if a revoked token is reused, revoke the entire family

Generate:
- `generateAccessToken(userId)` → signed JWT string
- `verifyAccessToken(token)` → decoded payload or error
- `generateRefreshToken()` → random hex string
- `hashRefreshToken(token)` → bcrypt/argon2 hash for storage
- `issueTokenPair(userId, db)` → creates refresh token in DB, returns both tokens

#### 3c. Auth Middleware

Generates middleware/dependency that:
1. Reads the `Authorization: Bearer <token>` header
2. Verifies the JWT signature and expiry
3. Attaches the user id to the request context
4. Returns `401 Unauthorized` (not `403`) with a generic message if the token is missing or invalid

Never return information about why the token failed (expired vs invalid vs missing) — return
the same `401` for all cases to avoid oracle attacks.

**Framework specifics:**
- **Express**: `(req, res, next) => void` middleware function
- **FastAPI**: `Depends()` dependency that returns the current user id
- **Axum**: `FromRequestParts` extractor or middleware layer

#### 3d. Auth Route Handlers

Generate all route handlers with input validation (using the project's validation library),
error handling, and correct HTTP status codes:

- Signup: `422` on validation failure, `409` if email already exists, `201` on success
- Login: `422` on validation failure, `401` on wrong credentials (same message for wrong email
  or wrong password — do not distinguish), `200` on success
- Refresh: `401` if token not found, expired, or revoked, `200` with new token pair on success
- Logout: `200` always (do not reveal if the token was valid)

#### 3e. Database Schema

Generate the migration or schema update needed for:
- `password_hash` column on the `users` table (if not present)
- `refresh_tokens` table: `id`, `user_id` (FK), `token_hash`, `family_id`, `expires_at`,
  `revoked_at` (nullable), `created_at`

Emit a note to run the migration using the project's tool (trigger builder-db-migration if needed).

#### 3f. Environment Variables

Add to `.env.example`:
```
JWT_SECRET=change_me_to_a_random_64_char_string
ACCESS_TOKEN_EXPIRY=15m
REFRESH_TOKEN_EXPIRY=7d
```

Include a note that `JWT_SECRET` must be a cryptographically random secret, and show the
command to generate one: `openssl rand -hex 64`.

#### 3g. Tests

Generate:
- Unit tests for `hashPassword` / `verifyPassword` and token utilities
- Integration tests for signup, login, refresh, and logout endpoints covering:
  - Happy path
  - Invalid credentials
  - Expired/revoked refresh token
  - Reuse of a revoked refresh token (family revocation)
  - Access to a protected route with and without a valid token

### Step 4 — OAuth Flow (if requested)

For OAuth providers, additionally generate:

1. **State parameter generation and validation** (CSRF protection for the callback)
2. **PKCE** if the provider supports it (recommended for all new OAuth integrations)
3. **User lookup/creation logic**: find by provider ID, or create a new user record and
   link the provider account in an `oauth_accounts` table
4. **Token issuance**: same JWT + refresh token flow as above after successful OAuth

Show the exact callback URL to register with the provider:
`https://your-domain.com/auth/[provider]/callback`

---

## Examples

### Example 1 — Express + TypeScript: JWT Auth Files

**User input:**
> setup jwt auth for my Express app, I already have a User model with id (uuid), email, and
> passwordHash fields

**Generated file tree:**
```
src/
  auth/
    auth.utils.ts       — password hashing, token generation
    auth.middleware.ts  — JWT verification middleware
    auth.routes.ts      — signup, login, refresh, logout handlers
    auth.test.ts        — integration tests
  db/
    migrations/
      20260319_add_refresh_tokens.sql
```

`auth.utils.ts` (excerpt):
```typescript
import bcrypt from 'bcrypt';
import jwt from 'jsonwebtoken';
import crypto from 'crypto';

const COST_FACTOR = 12;
const ACCESS_EXPIRY = process.env.ACCESS_TOKEN_EXPIRY ?? '15m';

export async function hashPassword(plain: string): Promise<string> {
  return bcrypt.hash(plain, COST_FACTOR);
}

export async function verifyPassword(
  plain: string,
  hash: string,
): Promise<boolean> {
  return bcrypt.compare(plain, hash);
}

export function generateAccessToken(userId: string): string {
  const secret = process.env.JWT_SECRET;
  if (!secret) throw new Error('JWT_SECRET is not set');
  return jwt.sign({ sub: userId }, secret, { expiresIn: ACCESS_EXPIRY });
}

export function verifyAccessToken(token: string): { sub: string } {
  const secret = process.env.JWT_SECRET;
  if (!secret) throw new Error('JWT_SECRET is not set');
  return jwt.verify(token, secret) as { sub: string };
}

export function generateRefreshToken(): string {
  return crypto.randomBytes(40).toString('hex');
}
```

`auth.middleware.ts`:
```typescript
import { Request, Response, NextFunction } from 'express';
import { verifyAccessToken } from './auth.utils';

export function requireAuth(req: Request, res: Response, next: NextFunction) {
  const header = req.headers.authorization;
  if (!header?.startsWith('Bearer ')) {
    return res.status(401).json({ error: 'Unauthorized' });
  }
  try {
    const payload = verifyAccessToken(header.slice(7));
    res.locals.userId = payload.sub;
    next();
  } catch {
    return res.status(401).json({ error: 'Unauthorized' });
  }
}
```

**Mount line to add to `src/app.ts`:**
```typescript
import { authRouter } from './auth/auth.routes';
app.use('/auth', authRouter);
```

---

### Example 2 — FastAPI: Auth Dependency

**User input:**
> add jwt authentication to my FastAPI app

**`auth/dependencies.py`:**
```python
from fastapi import Depends, HTTPException, status
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
from .utils import verify_access_token

bearer_scheme = HTTPBearer()


async def get_current_user_id(
    credentials: HTTPAuthorizationCredentials = Depends(bearer_scheme),
) -> int:
    token = credentials.credentials
    payload = verify_access_token(token)
    if payload is None:
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Unauthorized",
        )
    return int(payload["sub"])
```

**Usage on a protected route:**
```python
from auth.dependencies import get_current_user_id

@router.get("/me")
async def get_me(user_id: int = Depends(get_current_user_id)):
    ...
```

---

### Example 3 — OAuth: GitHub Provider (Express)

**User input:**
> oauth setup with GitHub for my Express app

**Generated endpoints:**
- `GET /auth/github` — redirects to `https://github.com/login/oauth/authorize?...`
  with `state` parameter stored in a signed, short-lived cookie
- `GET /auth/github/callback` — validates `state`, exchanges code for access token,
  fetches GitHub user profile, upserts into `oauth_accounts` table, issues JWT pair

**Callback URL to register in GitHub OAuth App settings:**
```
https://your-domain.com/auth/github/callback
```

**Environment variables to add to `.env.example`:**
```
GITHUB_CLIENT_ID=
GITHUB_CLIENT_SECRET=
OAUTH_STATE_SECRET=change_me_to_a_random_32_char_string
```

---

## Anti-patterns

1. **Storing JWTs in `localStorage`** — JWTs in localStorage are accessible to any JavaScript
   on the page, making them vulnerable to XSS. Recommend `HttpOnly` cookies for refresh tokens.
   Access tokens can live in memory for SPAs. Never generate code that stores sensitive tokens
   in localStorage without an explicit warning.

2. **Using the same error message differentiation for wrong email vs wrong password** — Returning
   "email not found" vs "wrong password" separately lets attackers enumerate valid email addresses.
   Always return the same `401` with a generic message like "Invalid credentials" for both cases.

3. **Storing raw refresh tokens in the database** — If the `refresh_tokens` table is compromised,
   raw tokens can be replayed immediately. Always store the hash and compare on lookup. Generate
   the raw token only to return to the client once.

4. **Not implementing refresh token rotation** — Issuing a new access token from the same
   long-lived refresh token repeatedly means a stolen refresh token remains valid indefinitely.
   Each refresh call must revoke the old refresh token and issue a new one.

5. **Skipping the `JWT_SECRET` environment variable check at startup** — If `JWT_SECRET` is
   undefined and the code falls back to an empty string or a hardcoded fallback, every token
   becomes trivially forgeable. Always validate required secrets at application startup and
   crash-fast with a clear error message if they are missing.
