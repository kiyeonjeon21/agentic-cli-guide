# Agentic CLI checklist

A checklist for making an existing CLI agent-friendly.
Items are prioritized; apply from top to bottom when practical.

---

## P0 — Required (minimum for agents to use the CLI)

### Output
- [ ] Support `--output json` or `--format json`
- [ ] Auto JSON when stdout is not a TTY (or via `OUTPUT_FORMAT=json`)
- [ ] Write errors to stderr as JSON (`{"error": {"code": "NOT_FOUND", "message": "..."}}`)
- [ ] Use meaningful exit codes (0=success, 1=general error, 2=invalid args, ...)

### Input
- [ ] Accept raw JSON via `--json` or stdin
- [ ] Validate path traversal: block inputs like `../../.ssh`
- [ ] Reject control characters: block below ASCII 0x20 (as you define policy)
- [ ] Block query-param injection in resource IDs: reject `?`, `#`, `%` when inappropriate

### Authentication
- [ ] Environment-based auth (`MY_CLI_TOKEN`, `MY_CLI_CREDENTIALS_FILE`)
- [ ] A path to authenticate without browser redirects

---

## P1 — Strongly recommended (large agent UX wins)

### Introspection
- [ ] `schema` or `--describe` to fetch parameter/response schema
- [ ] Schema output in JSON (types, required fields, OAuth scopes)
- [ ] Schema stays in sync with API version (e.g. discovery document)

### Context protection
- [ ] `--fields` or field masks to limit response fields
- [ ] Pagination for list operations
- [ ] NDJSON option (`--page-all` for streamed output)

### Safety
- [ ] `--dry-run` validates mutating work locally without applying
- [ ] Dry-run output states what *would* happen if executed

---

## P2 — Recommended (ecosystem integration)

### Skill / context files
- [ ] Ship `CONTEXT.md` or `SKILL.md`
- [ ] State invariants (e.g. "always `--dry-run` before mutating calls")
- [ ] Document prerequisites: binaries, env vars, permissions

### Multi-surface
- [ ] MCP (Model Context Protocol) surface: JSON-RPC over stdio
- [ ] MCP service filtering (`--services drive,gmail`, etc.)
- [ ] Extension / plugin install path documented

### Security
- [ ] `--sanitize` to filter prompt injection in API responses
- [ ] Declare reachable resource scope in `AGENTS.md`
- [ ] Prevent leaking secrets (tokens, passwords) on stdout

---

## Example usage

```bash
# P0: JSON output + raw JSON input
my-cli users list --output json --fields "id,name,email"
my-cli users create --json '{"name": "Alice", "role": "admin"}'

# P1: schema + dry-run
my-cli schema users.create
my-cli users create --json '{"name": "Alice"}' --dry-run

# P2: MCP surface
my-cli mcp --services users,roles
```
