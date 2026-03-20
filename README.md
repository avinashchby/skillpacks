# skillpacks

20 production-ready Claude Code skills in 4 themed packs — install once, invoke from any project.

## Quick Start

```bash
claude install https://github.com/avinashchby/skillpacks
```

## What It Does

skillpacks is a curated collection of Claude Code plugins that accelerate real development workflows. Each skill is a focused, trigger-driven command you invoke directly inside a Claude Code session using a slash command — no configuration required. The 4 packs cover solo developer git workflows, code quality automation, full-stack scaffolding, and AI/LLM tooling. Skills are plain Markdown files with structured metadata, so they are readable, forkable, and fully customizable.

## The Packs

### Solo Developer Essentials

Slash commands: `/commit` `/pr` `/deps` `/release` `/env`

- `solo-git-workflow` — Analyzes staged changes and writes a conventional commit message with correct type, scope, and breaking-change detection
- `solo-pr-creator` — Creates a GitHub PR with AI-generated title, structured description, and reviewer suggestions from CODEOWNERS
- `solo-dependency-updater` — Audits outdated packages, fetches changelogs, identifies breaking changes, and performs the update
- `solo-release-notes` — Generates release notes grouped by conventional commit type between two semver tags
- `solo-env-setup` — Detects project type from manifest files and verifies all required tools and runtime versions are installed

### Code Quality

Slash commands: `/review` `/tests` `/docs` `/refactor` `/perf`

- `quality-review-checklist` — Systematic code review covering OWASP Top 10 security, performance bottlenecks, WCAG 2.1 accessibility, and error handling
- `quality-test-generator` — Generates a complete test suite with happy paths, edge cases, error cases, and all reachable branches
- `quality-doc-generator` — Generates JSDoc, TSDoc, Python docstrings, Go doc comments, or Rust doc comments for all exported public items
- `quality-refactor-advisor` — Identifies code smells (long functions, deep nesting, god objects, duplicate code) and suggests targeted refactorings
- `quality-perf-profiler` — Identifies N+1 queries, missing indexes, synchronous blocking, and memory leaks through static analysis

### Full-Stack Builder

Slash commands: `/api` `/migrate` `/component` `/auth` `/deploy`

- `builder-api-scaffolder` — Generates a complete CRUD API (routes, validation, repository, tests) from a data model description; supports Express, FastAPI, and Actix-web/Axum
- `builder-db-migration` — Generates database migration files from a before/after schema description
- `builder-component-gen` — Generates typed UI components with TypeScript types, Storybook stories, and tests from a plain-language description
- `builder-auth-setup` — Sets up JWT or OAuth authentication with middleware and route guards
- `builder-deploy-config` — Generates production-ready Dockerfile, GitHub Actions workflow, and deployment manifests

### AI/MCP Developer

Slash commands: `/mcp` `/prompt` `/agent` `/rag` `/eval`

- `ai-mcp-server-creator` — Scaffolds a complete MCP server with typed tool definitions, JSON Schema validation, and resource handlers from a spec
- `ai-prompt-engineer` — Analyzes and restructures LLM prompts for token efficiency, output consistency, and model performance
- `ai-agent-debugger` — Diagnoses misbehaving agent workflows by analyzing tool call traces, identifying loops, and classifying root causes (works with LangChain, LangGraph, CrewAI, AutoGen)
- `ai-rag-pipeline` — Designs and implements end-to-end RAG pipelines covering document loading, chunking, embedding, and vector store indexing
- `ai-eval-harness` — Creates evaluation harnesses for LLM outputs with test case suites, accuracy metrics, latency tracking, and prompt variant comparisons

## Usage

Install all 20 skills with a single command, then invoke them inside any Claude Code session:

```
/commit
```
Reads your staged diff, infers the conventional commit type and scope, and proposes a commit message for approval before running `git commit`.

```
/api
User model with id, email, hashed_password, created_at. Needs CRUD endpoints.
```
Detects your framework (Express, FastAPI, Axum) from project files, then generates route handlers, request/response types, input validation, a repository layer, and unit tests.

```
/tests src/utils/parser.ts
```
Reads the file, identifies all exported functions, and writes a test suite covering happy paths, edge cases, and error conditions using the test runner already in the project.

```
/agent
```
Paste an agent trace or describe the failure; the skill normalizes the tool call sequence into a structured table, classifies the root cause (loop, hallucinated args, context overflow, etc.), and produces a concrete fix.

```
/review
```
Runs a multi-dimensional checklist across security (OWASP Top 10), performance, accessibility (WCAG 2.1), and error handling — producing a prioritized findings list with remediation guidance.

## Example Output

**`/commit` on a staged diff that fixes a JWT validation bug:**

```
fix(auth): enforce token expiry check for admin roles

Admin tokens bypassed the expiry validation introduced in v1.4.0 because
the role check short-circuited before reaching the expiry guard. All
tokens now go through the same validation path regardless of role.

Closes #88
```

**`/agent` on a stuck LangGraph workflow:**

```
## Agent Debug Report

### Trace Summary
Step | Tool Called  | Key Inputs                         | Issue?
-----|--------------|------------------------------------|-------
1    | search_web   | query="OpenAI GPT-5 release date"  | —
2    | search_web   | query="OpenAI GPT-5 release date"  | LOOP
3    | search_web   | query="OpenAI GPT-5 release date"  | LOOP

### Root Cause
RC-1: Missing termination condition + RC-6: Infinite retry on recoverable error.

### Fix
Add a one-reformulation fallback to the system prompt and cap retries at 2
in the tool definition.
```

## Installation

```bash
claude install https://github.com/avinashchby/skillpacks
```

This installs all 4 packs and all 20 skills into your Claude Code environment. Skills are then available as slash commands in any Claude Code session across any project.

## License

MIT
