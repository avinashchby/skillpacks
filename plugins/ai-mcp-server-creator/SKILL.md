---
name: ai-mcp-server-creator
description: >
  Scaffolds a complete, production-ready Model Context Protocol (MCP) server from a natural
  language description. Generates tool definitions with JSON Schema validation, resource
  handlers, prompt templates, and transport configuration for both stdio and SSE. Includes
  typed request/response models, error handling, and a test suite. Trigger phrases: "create
  mcp server", "build mcp server", "mcp tool", "model context protocol", "scaffold mcp",
  "new mcp", "build a tool server".
version: 1.0.0
---

# MCP Server Creator

## When to Use

Activate this skill when the user:
- Says "create mcp server", "build mcp", "scaffold mcp", or "model context protocol"
- Wants to expose tools, resources, or prompts to an LLM via the MCP spec
- Asks to wrap an existing API, CLI, or library as an MCP server
- Needs stdio transport (for Claude Desktop) or SSE transport (for web clients)
- Wants a working server they can register in `claude_desktop_config.json` immediately

Do NOT activate for general "build an API" requests unless MCP is explicitly mentioned.

---

## Instructions

### Step 1 — Gather requirements

Before writing any code, confirm the following from the user's description:

1. **Language** — Default to Python (using `mcp` SDK). Ask if they prefer TypeScript.
2. **Transport** — Default to `stdio` for Claude Desktop integration. Ask if SSE is needed.
3. **Tools** — List each tool: name, purpose, input parameters, return type.
4. **Resources** — Any static or dynamic content the server exposes (files, DB rows, URLs).
5. **Prompts** — Any reusable prompt templates the server advertises.
6. **External dependencies** — APIs, databases, or SDKs the tools must call.

If the description is detailed enough, infer all of the above and state your assumptions explicitly before generating code.

### Step 2 — Design tool schemas

For every tool:
- Name: `snake_case`, verb-first (e.g., `search_documents`, `create_issue`)
- Description: One sentence that tells the LLM exactly when to call this tool
- Input schema: JSON Schema object with `required` array and typed `properties`
- Return: Always return a `list[TextContent | ImageContent | EmbeddedResource]`

Validation rules:
- String inputs that are constrained: add `enum` or `pattern`
- Numeric inputs: add `minimum`/`maximum` where sensible
- Optional fields: omit from `required`, add `default` in the description string

### Step 3 — Generate project structure

For Python MCP servers, produce this layout:

```
<server-name>/
├── pyproject.toml
├── README.md
├── src/
│   └── <server_name>/
│       ├── __init__.py
│       ├── server.py          # FastMCP app, tool/resource registration
│       ├── tools/
│       │   ├── __init__.py
│       │   └── <domain>.py    # One file per logical group of tools
│       ├── resources/
│       │   ├── __init__.py
│       │   └── <domain>.py
│       └── types.py           # Pydantic models for all inputs/outputs
└── tests/
    ├── conftest.py
    └── test_tools.py
```

For TypeScript MCP servers, produce this layout:

```
<server-name>/
├── package.json
├── tsconfig.json
├── src/
│   ├── index.ts               # Entry point, transport setup
│   ├── server.ts              # McpServer instance, registrations
│   ├── tools/
│   │   └── <domain>.ts
│   └── types.ts               # Zod schemas
└── tests/
    └── tools.test.ts
```

### Step 4 — Write server.py (Python)

Always use `FastMCP` from the `mcp` package (version ≥ 1.0). Structure:

```python
from mcp.server.fastmcp import FastMCP
from .tools.domain import register_tools

mcp = FastMCP(
    name="<server-name>",
    version="1.0.0",
    description="<one-sentence description>",
)

register_tools(mcp)

if __name__ == "__main__":
    mcp.run()  # defaults to stdio transport
```

For SSE transport, add:
```python
mcp.run(transport="sse", host="0.0.0.0", port=8080)
```

### Step 5 — Implement tools with error handling

Every tool implementation must:
1. Validate inputs using Pydantic before any external call
2. Wrap external calls in try/except; raise `McpError` with an `ErrorCode` on failure
3. Return structured `TextContent` or typed content; never return raw dicts
4. Log at DEBUG level before each external call for traceability

```python
from mcp.server.fastmcp import FastMCP
from mcp.types import ErrorCode, McpError

def register_tools(mcp: FastMCP) -> None:

    @mcp.tool()
    async def search_documents(query: str, limit: int = 10) -> str:
        """Search indexed documents by semantic similarity.

        Args:
            query: Natural language search query.
            limit: Maximum number of results to return (1-50).
        """
        if not 1 <= limit <= 50:
            raise McpError(ErrorCode.INVALID_PARAMS, "limit must be between 1 and 50")
        try:
            results = await _do_search(query, limit)
            return "\n\n".join(f"## {r.title}\n{r.snippet}" for r in results)
        except ConnectionError as e:
            raise McpError(ErrorCode.INTERNAL_ERROR, f"Search backend unavailable: {e}")
```

### Step 6 — Write pyproject.toml

```toml
[project]
name = "<server-name>"
version = "1.0.0"
requires-python = ">=3.11"
dependencies = [
    "mcp>=1.0.0",
    "pydantic>=2.0",
    "httpx>=0.27",        # if HTTP calls needed
]

[project.scripts]
<server-name> = "<server_name>.server:mcp.run"

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

### Step 7 — Write tests

Every tool must have at least one happy-path and one error-path test.

```python
import pytest
from mcp.types import ErrorCode, McpError
from unittest.mock import AsyncMock, patch
from <server_name>.tools.domain import register_tools
from mcp.server.fastmcp import FastMCP

@pytest.fixture
def mcp_server():
    server = FastMCP(name="test", version="0.0.1", description="test")
    register_tools(server)
    return server

@pytest.mark.asyncio
async def test_search_documents_returns_results(mcp_server):
    with patch("<server_name>.tools.domain._do_search", new_callable=AsyncMock) as mock:
        mock.return_value = [MockResult(title="Doc1", snippet="Content")]
        result = await mcp_server.call_tool("search_documents", {"query": "test"})
    assert "Doc1" in result

@pytest.mark.asyncio
async def test_search_documents_rejects_bad_limit(mcp_server):
    with pytest.raises(McpError) as exc:
        await mcp_server.call_tool("search_documents", {"query": "x", "limit": 0})
    assert exc.value.error.code == ErrorCode.INVALID_PARAMS
```

### Step 8 — Provide claude_desktop_config.json snippet

Always end with a ready-to-paste config block:

```json
{
  "mcpServers": {
    "<server-name>": {
      "command": "uv",
      "args": ["run", "--project", "/path/to/<server-name>", "<server-name>"],
      "env": {
        "API_KEY": "your-key-here"
      }
    }
  }
}
```

---

## Examples

### Example 1 — GitHub Issues MCP Server

**User input:** "Create an MCP server that can search GitHub issues, create new issues, and add comments."

**Inferred design:**
- Tools: `search_issues(repo, query, state)`, `create_issue(repo, title, body, labels)`, `add_comment(repo, issue_number, body)`
- Auth: `GITHUB_TOKEN` from environment
- Transport: stdio

**Generated tool schema (search_issues):**
```python
@mcp.tool()
async def search_issues(
    repo: str,
    query: str,
    state: Literal["open", "closed", "all"] = "open",
    limit: int = 20,
) -> str:
    """Search GitHub issues in a repository.

    Args:
        repo: Repository in 'owner/name' format (e.g., 'anthropics/anthropic-sdk-python').
        query: Search terms to filter issues by title or body.
        state: Filter by issue state.
        limit: Maximum results (1-100).
    """
```

**Config output:**
```json
{
  "mcpServers": {
    "github-issues": {
      "command": "uv",
      "args": ["run", "--project", "~/mcp-servers/github-issues", "github-issues"],
      "env": { "GITHUB_TOKEN": "ghp_..." }
    }
  }
}
```

---

### Example 2 — SQLite Query Server

**User input:** "Build an MCP server that lets Claude query a local SQLite database safely — read-only SELECT only."

**Key decisions:**
- Tools: `list_tables()`, `describe_table(name)`, `run_query(sql)` — enforces SELECT-only
- Resources: Expose schema as `resource://schema/tables`
- Validation: Parse SQL with `sqlglot`, reject any non-SELECT statement before execution

**Generated validation in run_query:**
```python
import sqlglot

@mcp.tool()
async def run_query(sql: str) -> str:
    """Execute a read-only SELECT query against the local database.

    Args:
        sql: A valid SQLite SELECT statement. INSERT/UPDATE/DELETE are rejected.
    """
    parsed = sqlglot.parse_one(sql, dialect="sqlite")
    if not isinstance(parsed, sqlglot.expressions.Select):
        raise McpError(ErrorCode.INVALID_PARAMS, "Only SELECT statements are permitted.")
    async with aiosqlite.connect(DB_PATH) as db:
        cursor = await db.execute(sql)
        rows = await cursor.fetchmany(200)
    return format_table(rows, [d[0] for d in cursor.description])
```

---

### Example 3 — SSE Transport for Web Deployment

**User input:** "Same GitHub server but I need it accessible from a web app over SSE, not just Claude Desktop."

**Changes from stdio baseline:**
```python
# server.py
import os
mcp.run(
    transport="sse",
    host="0.0.0.0",
    port=int(os.getenv("PORT", "8080")),
)
```

**Docker addition:**
```dockerfile
FROM python:3.12-slim
WORKDIR /app
COPY . .
RUN pip install uv && uv sync
EXPOSE 8080
CMD ["uv", "run", "github-issues"]
```

---

## Anti-patterns

1. **Returning raw Python dicts from tools.** MCP tools must return `str` or explicit content types (`TextContent`, `ImageContent`). Returning a dict causes a serialization error at the transport layer. Always convert to a formatted string or use `mcp.types` content wrappers.

2. **Putting all tools in server.py.** As the server grows, a monolithic `server.py` becomes impossible to test in isolation. Always split tools into domain modules under `tools/` and register them via a `register_tools(mcp)` function.

3. **Ignoring the `required` array in JSON Schema.** If you omit `required`, the MCP client cannot distinguish "user didn't provide it" from "user provided null". Always list every non-optional parameter in `required`. Use `default` values only for genuinely optional parameters.

4. **Using `os.environ` directly inside tool functions.** Read all secrets at startup into module-level constants or a `Settings` Pydantic model. Tool functions called at request time should never call `os.environ` — it makes the server impossible to test without real credentials set.

5. **Skipping error codes.** Raising a plain `Exception` from a tool causes the MCP client to receive an opaque internal error. Always raise `McpError(ErrorCode.INVALID_PARAMS, ...)` for bad input and `McpError(ErrorCode.INTERNAL_ERROR, ...)` for backend failures so the LLM can describe the problem accurately to the user.
