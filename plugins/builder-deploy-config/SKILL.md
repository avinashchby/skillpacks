---
name: builder-deploy-config
description: >
  Generates production-ready deployment configuration files. Triggered when the user says
  "deploy", "dockerfile", "ci/cd", "github actions", "docker compose", "deploy config",
  "containerize", "set up pipeline", "add health check", "railway config", or "vercel config".
  Detects the project type (Node.js, Python, Rust, static site), runtime requirements, and
  target platform, then produces a multi-stage Dockerfile, docker-compose for local dev,
  a GitHub Actions workflow, and a platform-specific config (Vercel, Railway, Fly.io) as
  appropriate — all optimized for production: minimal image size, health checks, and secrets
  handled via environment variables.
version: 1.0.0
---

# Deployment Config Generator

## When to Use

Activate this skill when the user:

- Says "deploy", "set up deployment", "deploy config", "deployment setup"
- Says "dockerfile", "containerize", "add docker", "write a Dockerfile"
- Says "docker compose", "add docker-compose", "local dev setup with Docker"
- Says "ci/cd", "github actions", "set up pipeline", "add a workflow"
- Says "vercel config", "railway config", "fly.io", "add platform config"
- Asks to "add health checks" to an existing service
- Asks to "optimize the Dockerfile" or "reduce image size"

Do NOT activate for:
- Kubernetes manifests (much larger scope — decompose and confirm with user first)
- Terraform or infrastructure-as-code (different skill domain)
- Secrets management systems (Vault, AWS Secrets Manager) — mention them, don't configure them

---

## Instructions

### Step 1 — Detect Project Type and Requirements

Scan the project before generating anything:

1. **Runtime**:
   - `package.json` with `start`/`build` scripts → Node.js
   - `pyproject.toml` or `requirements.txt` → Python
   - `Cargo.toml` → Rust
   - `go.mod` → Go
   - Static site builder (`next build`, `vite build`, `astro build`) → static HTML output

2. **Node.js specifics**:
   - Package manager: `package-lock.json` (npm), `yarn.lock` (yarn), `pnpm-lock.yaml` (pnpm)
   - Framework: Next.js (requires Node runtime), plain Express (any Node), Astro/Vite (may
     be fully static)
   - Node version: `.nvmrc`, `engines` field in `package.json`, or `.node-version`

3. **Python specifics**:
   - Package manager: `uv` (`uv.lock`, `pyproject.toml`), `poetry` (`poetry.lock`), or `pip`
     (`requirements.txt`)
   - ASGI vs WSGI: FastAPI/Starlette → `uvicorn`; Flask/Django → `gunicorn`
   - Python version: `.python-version` or `pyproject.toml`

4. **Rust specifics**:
   - Binary name from `Cargo.toml` `[[bin]]` or `[package.name]`
   - Target: `x86_64-unknown-linux-musl` for smallest static binary

5. **Port**: Check for `PORT` env var usage, `listen(3000)`, or `uvicorn --port`. If not
   found, use `8080` as a safe default and comment it.

6. **Health check endpoint**: Look for `/health`, `/healthz`, `/ping`, or `/ready` endpoints.
   If none exist, note that one should be added and generate a minimal stub.

7. **Environment variables**: Check `.env.example` for required vars to document in the
   compose file and CI workflow.

8. **Database / services**: Check for Postgres, Redis, or other services in the app code.
   Include them in `docker-compose.yml` if found.

### Step 2 — Choose What to Generate

Based on the user's request and project context, generate the applicable subset:

| Artifact                     | Generate when                                     |
|------------------------------|---------------------------------------------------|
| `Dockerfile`                 | Any containerizable project                       |
| `docker-compose.yml`         | Project has external services OR user asked       |
| `.github/workflows/ci.yml`   | Always, if `.github/` exists or user asked        |
| `.github/workflows/deploy.yml`| User asked for deploy pipeline                  |
| `vercel.json`                | Next.js / static site + user asked               |
| `railway.toml`               | User mentioned Railway                            |
| `fly.toml`                   | User mentioned Fly.io                             |
| `.dockerignore`              | Always, alongside Dockerfile                      |

Do not generate all of these by default. Ask or infer from the user's request.

### Step 3 — Generate Dockerfile

Use multi-stage builds for all compiled or bundled runtimes. Rules:

**General rules:**
- Stage 1 (`builder`): full SDK image, installs deps, runs build
- Stage 2 (`runner`): minimal base image, copies only production artifacts
- Base images by runtime:
  - Node.js production: `node:22-alpine`
  - Python production: `python:3.13-slim` or `distroless/python3`
  - Rust production: `scratch` or `alpine:3` (with musl build)
  - Static site: `nginx:alpine`
- Run as a non-root user in the final stage. Create a dedicated user if the base image
  does not provide one.
- Use `COPY --chown=user:user` to set ownership in the same layer as the copy.
- Pin the base image to a specific minor version (e.g., `node:22.14-alpine`), not `latest`.
- Set `NODE_ENV=production` (or equivalent) in the production stage.
- Expose only the application port.
- Use `CMD` (not `ENTRYPOINT`) unless the container is always used as an executable.

**Node.js multi-stage example structure:**
```
FROM node:22-alpine AS deps      — install only production deps
FROM node:22-alpine AS builder   — copy deps, copy src, run build
FROM node:22-alpine AS runner    — copy built output, run app
```

**Python example structure:**
```
FROM python:3.13-slim AS builder — install deps into a venv
FROM python:3.13-slim AS runner  — copy venv, set PATH, run app
```

**Rust example structure:**
```
FROM rust:1.83-alpine AS builder — cargo build --release --target musl
FROM alpine:3.21 AS runner       — copy binary, run
```

### Step 4 — Generate .dockerignore

Always include:
```
.git
.github
node_modules
__pycache__
*.pyc
target/
dist/
.env
.env.*
!.env.example
*.log
coverage/
.DS_Store
```

Adjust for the detected runtime (e.g., add `venv/` for Python, `*.lock` exceptions for
package managers that need lock files).

### Step 5 — Generate docker-compose.yml

Purpose: local development parity with production services.

Rules:
- Use `docker-compose.yml` version `"3.9"` (widely supported)
- Define the app service with a `build: .` context
- Mount source code as a volume for hot reload (dev only — note this in a comment)
- Define named volumes for database data directories
- Use `depends_on` with `condition: service_healthy` for databases
- Set healthchecks on all service definitions
- Use `env_file: .env` for application secrets; define service passwords as
  `environment:` vars with `${VAR:-default}` syntax
- Include a `networks:` section so services communicate by name

Services to include based on detection:
- PostgreSQL → `postgres:17-alpine` with health check `pg_isready`
- Redis → `redis:7-alpine` with health check `redis-cli ping`
- MongoDB → `mongo:7` with health check

### Step 6 — Generate GitHub Actions Workflow

#### CI Workflow (`.github/workflows/ci.yml`)

Triggers: `push` to any branch, `pull_request` to `main`/`master`.

Jobs to include:
1. **lint** — run linter (ESLint, ruff, clippy)
2. **test** — run test suite with services (postgres, redis) as job services if needed
3. **build** — build Docker image (do not push in CI, only verify it builds)

Rules:
- Use `actions/checkout@v4`
- Cache dependencies: `actions/cache` with the correct key for the runtime
  (npm cache, uv cache, Cargo registry + target)
- Use matrix strategy if multiple Node/Python versions are tested
- Pin all action versions to a full commit SHA or a specific major version (e.g., `@v4`)
- Set `permissions: contents: read` at the workflow level; escalate only where needed

#### Deploy Workflow (`.github/workflows/deploy.yml`)

Triggers: push to `main` (or tag `v*.*.*`).

Steps:
1. Checkout
2. Log in to container registry (GHCR or Docker Hub via secrets)
3. Set up Docker Buildx with layer caching
4. Build and push image with tags: `latest` and the commit SHA
5. Platform-specific deploy step (comment placeholder if platform unknown)

Secrets to document:
- `REGISTRY_USERNAME`, `REGISTRY_PASSWORD` (or `GITHUB_TOKEN` for GHCR)
- Any platform deploy tokens

### Step 7 — Add Health Check Endpoint (if missing)

If no health endpoint exists, generate a minimal one:

**Express:**
```typescript
app.get('/health', (_req, res) => {
  res.status(200).json({ status: 'ok' });
});
```

**FastAPI:**
```python
@app.get("/health")
async def health():
    return {"status": "ok"}
```

**Axum:**
```rust
async fn health() -> impl IntoResponse {
    Json(serde_json::json!({ "status": "ok" }))
}
```

---

## Examples

### Example 1 — Node.js (Express + Prisma + Postgres)

**User input:**
> add Dockerfile, docker-compose, and GitHub Actions for my Express app

**`Dockerfile`:**
```dockerfile
FROM node:22-alpine AS deps
WORKDIR /app
COPY package.json package-lock.json ./
RUN npm ci --omit=dev

FROM node:22-alpine AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN npm run build

FROM node:22-alpine AS runner
WORKDIR /app
ENV NODE_ENV=production
RUN addgroup --system appgroup && adduser --system --ingroup appgroup appuser
COPY --from=builder --chown=appuser:appgroup /app/dist ./dist
COPY --from=deps --chown=appuser:appgroup /app/node_modules ./node_modules
COPY --chown=appuser:appgroup package.json ./
USER appuser
EXPOSE 8080
CMD ["node", "dist/server.js"]
```

**`docker-compose.yml`** (excerpt):
```yaml
version: "3.9"
services:
  app:
    build: .
    ports:
      - "8080:8080"
    env_file: .env
    depends_on:
      postgres:
        condition: service_healthy
    networks:
      - app-net

  postgres:
    image: postgres:17-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER:-app}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-secret}
      POSTGRES_DB: ${POSTGRES_DB:-appdb}
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER:-app}"]
      interval: 5s
      timeout: 5s
      retries: 5
    networks:
      - app-net

volumes:
  pgdata:

networks:
  app-net:
```

---

### Example 2 — Python (FastAPI + uv)

**User input:**
> dockerfile for my FastAPI app, using uv for packages

**`Dockerfile`:**
```dockerfile
FROM python:3.13-slim AS builder
WORKDIR /app
RUN pip install uv
COPY pyproject.toml uv.lock ./
RUN uv sync --frozen --no-dev

FROM python:3.13-slim AS runner
WORKDIR /app
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PATH="/app/.venv/bin:$PATH"
RUN useradd --no-create-home --shell /bin/false appuser
COPY --from=builder --chown=appuser /app/.venv ./.venv
COPY --chown=appuser . .
USER appuser
EXPOSE 8080
CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8080"]
```

---

### Example 3 — GitHub Actions CI with Caching

**`.github/workflows/ci.yml`:**
```yaml
name: CI

on:
  push:
  pull_request:
    branches: [main]

permissions:
  contents: read

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:17-alpine
        env:
          POSTGRES_PASSWORD: testpass
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
        ports:
          - 5432:5432

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version-file: .nvmrc
          cache: npm

      - run: npm ci
      - run: npm run lint
      - run: npm test
        env:
          DATABASE_URL: postgres://postgres:testpass@localhost:5432/test

  docker-build:
    runs-on: ubuntu-latest
    needs: test
    steps:
      - uses: actions/checkout@v4
      - uses: docker/setup-buildx-action@v3
      - name: Build image (no push)
        uses: docker/build-push-action@v6
        with:
          context: .
          push: false
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

---

## Anti-patterns

1. **Using `FROM node:latest` or unpinned base images** — `latest` tags change without warning
   and will break builds unpredictably. Always pin to a specific minor version tag. For
   long-lived images, also pin to a digest for full reproducibility.

2. **Installing dev dependencies in the production image** — Dev dependencies (test runners,
   linters, type checkers) can add hundreds of megabytes and introduce vulnerabilities that
   are never present at runtime. Always use multi-stage builds and install only production
   deps in the final stage.

3. **Running the container process as root** — The default user in most base images is root.
   A process running as root inside a container has elevated capabilities if the container
   escapes. Always create and switch to a non-root user in the final stage.

4. **Storing secrets in the Dockerfile or docker-compose.yml** — Hardcoded passwords, API keys,
   or connection strings in image layers are extractable from any machine that can pull the image.
   All secrets must come from environment variables, `.env` files (excluded from Docker builds
   via `.dockerignore`), or a secrets manager at runtime.

5. **Skipping health checks in both the Dockerfile and docker-compose** — Without a health check,
   Docker and orchestrators have no way to know if the app is actually ready to serve traffic
   after startup. The container will be marked healthy the moment the process starts, even if
   it's still connecting to the database. Always define a `HEALTHCHECK` in the Dockerfile and
   a `healthcheck` in compose.
