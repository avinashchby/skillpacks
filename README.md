# skillpacks

**20 production-ready Claude Code skills in 4 themed packs.**

skillpacks is a curated collection of Claude Code plugins that accelerate real development workflows — from smart commits and PR creation to MCP server scaffolding and LLM evaluation. Each skill is a focused, trigger-driven command you invoke directly from Claude Code, with no configuration required.

---

## Installation

```bash
claude install https://github.com/avinashchby/skillpacks
```

This installs all 4 packs and all 20 skills into your Claude Code environment.

---

## Packs

### Solo Developer Essentials

Everything a solo developer needs for a professional git workflow without the overhead of a full team toolchain.

**Use cases:**
- Drafting conventional commits from staged changes without writing them manually
- Creating structured GitHub PRs with descriptions and reviewer suggestions in seconds
- Auditing dependency staleness before a release
- Generating changelogs directly from commit history
- Verifying that a new machine or CI environment is correctly configured for the project

---

### Code Quality

Automated code review, testing, documentation, and performance analysis — applied consistently on every change.

**Use cases:**
- Running a security, performance, and accessibility checklist before merging
- Generating unit tests with edge cases for untested functions
- Adding JSDoc or docstrings to all exported symbols in a module
- Getting targeted refactoring suggestions for complex or duplicated code
- Spotting N+1 queries, memory leaks, and hot paths in a codebase

---

### Full-Stack Builder

Scaffold the structural components of a full-stack application — APIs, database migrations, UI components, auth, and deployment — from natural language descriptions.

**Use cases:**
- Generating a fully typed CRUD REST API from a data model description
- Creating database migration files from a before/after schema description
- Scaffolding React or Vue components with TypeScript types, Storybook stories, and tests
- Setting up JWT or OAuth flows with middleware and route guards
- Generating a production-ready Dockerfile, GitHub Actions workflow, and deployment manifest

---

### AI/MCP Developer

Purpose-built tools for developers building with LLMs — MCP servers, prompt optimization, agent debugging, RAG pipelines, and evaluation harnesses.

**Use cases:**
- Scaffolding a complete MCP server with typed tools and resources from a spec
- Iterating on prompts to reduce token usage while preserving output quality
- Diagnosing agent loops, failed tool calls, and reasoning failures in multi-step workflows
- Wiring up a full RAG pipeline with chunking, embedding, and vector store integration
- Creating evaluation harnesses that measure LLM output quality across prompt variants

---

## Skill Catalog

| Skill | Pack | Description | Trigger Phrase |
|---|---|---|---|
| `solo-git-workflow` | Solo Developer Essentials | Smart conventional commits from staged changes | `/commit` |
| `solo-pr-creator` | Solo Developer Essentials | AI-generated GitHub PRs with title, description, and reviewer suggestions | `/pr` |
| `solo-dependency-updater` | Solo Developer Essentials | Check and update outdated dependencies with breaking change warnings | `/deps` |
| `solo-release-notes` | Solo Developer Essentials | Generate release notes grouped by conventional commit type | `/release` |
| `solo-env-setup` | Solo Developer Essentials | Detect project type and verify dev environment setup | `/env` |
| `quality-review-checklist` | Code Quality | Systematic code review covering security, performance, and accessibility | `/review` |
| `quality-test-generator` | Code Quality | Generate comprehensive unit tests with edge case coverage | `/tests` |
| `quality-doc-generator` | Code Quality | Generate JSDoc/docstrings for exported functions with examples | `/docs` |
| `quality-refactor-advisor` | Code Quality | Identify code smells and suggest targeted refactorings | `/refactor` |
| `quality-perf-profiler` | Code Quality | Identify N+1 queries, memory leaks, and performance bottlenecks | `/perf` |
| `builder-api-scaffolder` | Full-Stack Builder | Generate full CRUD API from data model description | `/api` |
| `builder-db-migration` | Full-Stack Builder | Generate database migrations from schema diffs | `/migrate` |
| `builder-component-gen` | Full-Stack Builder | Generate typed UI components with stories and tests | `/component` |
| `builder-auth-setup` | Full-Stack Builder | Set up JWT or OAuth authentication with middleware | `/auth` |
| `builder-deploy-config` | Full-Stack Builder | Generate Dockerfile, CI/CD, and deployment configs | `/deploy` |
| `ai-mcp-server-creator` | AI/MCP Developer | Scaffold complete MCP servers with tools and resources | `/mcp` |
| `ai-prompt-engineer` | AI/MCP Developer | Optimize prompts for token efficiency and consistency | `/prompt` |
| `ai-agent-debugger` | AI/MCP Developer | Debug agent workflows and identify stuck loops | `/agent` |
| `ai-rag-pipeline` | AI/MCP Developer | Set up complete RAG pipeline with vector store | `/rag` |
| `ai-eval-harness` | AI/MCP Developer | Create LLM evaluation harnesses with metrics and comparisons | `/eval` |

---

## Quick Start

After installation, skills are available as slash commands inside any Claude Code session.

**Example — create a commit from staged changes:**

```
/commit
```

Claude will analyze your staged diff, infer the conventional commit type and scope, and propose a commit message for your approval before running `git commit`.

**Example — scaffold an API:**

```
/api

User model with id, email, hashed_password, created_at. Needs CRUD endpoints.
```

Claude will generate route handlers, validation, and controller logic for the described model in the framework it detects from your project.

**Example — generate tests:**

```
/tests src/utils/parser.ts
```

Claude will read the file, identify all exported functions, and write unit tests covering happy paths, edge cases, and error conditions.

---

## Create Your Own

Skills are plain Markdown files with structured metadata. You can write a custom skill for any repeated workflow in your project.

Full documentation: [https://docs.anthropic.com/en/docs/claude-code](https://docs.anthropic.com/en/docs/claude-code)

---

## Contributing

Contributions are welcome. If you have a skill that solves a real workflow problem and fits cleanly into one of the existing packs, open a PR with the skill file and a brief description of the problem it solves. New packs require at least 5 cohesive skills and a clear target use case that does not overlap with an existing pack.

---

## License

MIT. See [LICENSE](./LICENSE).
