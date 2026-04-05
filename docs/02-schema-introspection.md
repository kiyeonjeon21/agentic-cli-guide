# 02. Schema Introspection

## The problem

Traditional ways agents learn CLI usage:

1. Read `--help` — unstructured text, many tokens
2. Read web docs — version skew risk, even more tokens
3. Inject docs into context — static; must refresh when the API changes

These approaches are **static**. When the API updates, docs lag and agents hallucinate parameters that do not exist.

## Solution: runtime introspection

The CLI exposes every supported method, parameter, and type in a machine-readable form.

```bash
# Schema for a specific method
my-cli schema users.create
```

Example output:

```json
{
  "method": "users.create",
  "httpMethod": "POST",
  "path": "/v1/users",
  "parameters": {
    "fields": {
      "type": "string",
      "description": "Selector specifying which fields to include in the response",
      "location": "query"
    }
  },
  "request": {
    "$ref": "User",
    "properties": {
      "name": {"type": "string", "required": true},
      "email": {"type": "string", "format": "email", "required": true},
      "role": {"type": "string", "enum": ["admin", "member", "viewer"], "default": "member"}
    }
  },
  "response": {
    "$ref": "User"
  },
  "scopes": [
    "https://www.googleapis.com/auth/admin.directory.user"
  ]
}
```

## Implementation patterns

### 1. Discovery document

As in Google Workspace CLI: load the API discovery document at runtime and resolve `$ref` dynamically.

```bash
# List available backing services
my-cli schema --list-services

# List methods in a service
my-cli schema --list-methods users

# Method detail
my-cli schema users.create
```

Benefit: when the API version moves, the schema updates without redeploying the CLI.

### 2. Bundled OpenAPI

Bundle an OpenAPI spec at build time and expose it with `--describe`.

```bash
my-cli users create --describe
```

Benefit: works offline, no network dependency.

### 3. Global `--describe` flag

Add `--describe` to commands so it returns the command’s schema instead of executing.

```bash
# Execute
my-cli users create --json '{"name":"Alice"}'

# Introspect (no execution)
my-cli users create --describe
```

## Effect on agents

| Static docs | Runtime introspection |
|-------------|----------------------|
| Must inject full docs (expensive tokens) | Fetch only the methods needed |
| May diverge from the API | Always reflects the current version |
| Agents may hallucinate parameter names | Exact parameter list |
| Types may be missing | Types, enums, defaults included |

## Design principles

- Schema output must be **JSON** or **JSON Schema**. Human-only prose burdens parsers.
- Clearly separate required vs optional fields.
- Always include `enum` values when they exist — reduces invalid generations.
- Include auth scopes / permission requirements.
