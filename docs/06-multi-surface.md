# 06. Multi-Surface: one binary, multiple interfaces

## The problem

Agents invoke CLIs in many contexts:

- Direct terminal (Claude Code, Cursor, Codex, etc.)
- Structured calls via MCP (Claude Desktop and other MCP clients)
- As an extension inside an agent framework
- Unattended in CI/CD

Each context expects different invocation style, auth, and output. One binary should serve all these **surfaces**.

## Surface characteristics

### 1. CLI (terminal)

The default surface for humans and agents.

```bash
# Human
my-cli files list

# Agent
my-cli files list --output json --fields "files(id,name)"
```

Key: auto-switch output with TTY detection.

```bash
# TTY: human-friendly table
$ my-cli files list
ID          NAME              TYPE
abc123      Q1 Report.docx    document
def456      Budget.xlsx       spreadsheet

# Non-TTY (pipe/redirect): JSON
$ my-cli files list | cat
{"files":[{"id":"abc123","name":"Q1 Report.docx"},{"id":"def456","name":"Budget.xlsx"}]}
```

### 2. MCP (Model Context Protocol)

Agents call the CLI as structured tools via JSON-RPC over stdio.

```bash
# Start MCP server (optional service filter)
my-cli mcp --services drive,gmail
```

The MCP server:

- Maps CLI operations to JSON-RPC tools
- Exposes parameters as typed JSON Schema
- Lets agents call tools without parsing `--help`

```json
// MCP tool definition (auto-generated)
{
  "name": "files_list",
  "description": "List files in Drive",
  "inputSchema": {
    "type": "object",
    "properties": {
      "q": {"type": "string", "description": "Search query"},
      "fields": {"type": "string", "description": "Field mask"},
      "pageSize": {"type": "integer", "default": 100}
    }
  }
}
```

### 3. Extension / plugin

Installed directly into an agent platform.

```bash
# Example: Gemini extension install
gemini extensions install https://github.com/example/my-cli

# Natural language from the agent
"Find the Q1 Report file in Drive"
→ internally runs my-cli files list --json '{"q":"name contains '\''Q1 Report'\''"}'
```

### 4. Environment-based authentication

Shared across surfaces.

```bash
# Token injection
export MY_CLI_TOKEN="ya29.a0AfH6SM..."

# Service account key file
export MY_CLI_CREDENTIALS_FILE="/path/to/service-account.json"

# Agents cannot open a browser — env is often the only path
```

**Why environment variables?**

| Auth mechanism | Human | Agent |
|----------------|-------|-------|
| Browser OAuth redirect | Yes | No (no browser) |
| Interactive prompt | Yes | No (non-interactive stdin) |
| Environment variables | Yes | Yes |
| Service account key file | Yes | Yes |

## Design principle: single source of truth

Every surface should share the same core logic.

```
                    ┌─── CLI (terminal)
                    │
Discovery Doc ──→ Core Logic ──┼─── MCP (JSON-RPC)
                    │
                    └─── Extension
```

- **Discovery document** (or OpenAPI) is the authoritative schema
- Core logic owns validation, API calls, response shaping
- Each surface is a thin adapter

Benefits:

1. One place to update when the API changes
2. No duplicated validation per surface
3. New surfaces are a small adapter

## Implementation checklist

- [ ] `--output json` + TTY auto-detection
- [ ] `my-cli mcp` subcommand starts the MCP server
- [ ] MCP service filter (`--services`) limits exposed tools
- [ ] Environment auth (`TOKEN`, `CREDENTIALS_FILE`)
- [ ] Clear errors when auth fails in non-interactive mode
  ```json
  {"error": {"code": "AUTH_REQUIRED", "message": "Set MY_CLI_TOKEN or MY_CLI_CREDENTIALS_FILE environment variable"}}
  ```
