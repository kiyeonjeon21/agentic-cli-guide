# {{CLI_NAME}} Agent Security Model

<!-- Declares what resources and operations the agent may access. Agent frameworks can read this to constrain tool use. Replace placeholders with real values. -->

## Allowed Resources

Define the scope the agent may touch.

### Read

| Service | Resource | Filter |
|---------|----------|--------|
| `{{SERVICE_A}}` | All resources | None |
| `{{SERVICE_B}}` | Owned by caller only | `owner = me` |

### Write (create / update)

| Service | Allowed | Constraint |
|---------|-----------|------------|
| `{{SERVICE_A}}` | create, update | Only within specific folder/namespace |
| `{{SERVICE_B}}` | create | Max 10 per day |

### Delete

| Service | Allowed | Condition |
|---------|---------|-----------|
| `{{SERVICE_A}}` | Conditional | Only resources the agent created; user confirmation required |
| `{{SERVICE_B}}` | Denied | — |

## Denied Operations

The agent must not:

- Change end-user account settings
- Change sharing permissions (especially external sharing)
- Call admin APIs
- Perform billing / payment operations
- Delete other users’ resources

## Authentication Scope

Minimum privileges for the agent:

```yaml
scopes:
  - "{{SERVICE_A}}.readonly"      # default: read-only
  - "{{SERVICE_A}}.files"         # file CRUD (folder-bound)
  # - "{{SERVICE_A}}.admin"       # never grant
```

## Rate Limits

| Operation | Limit |
|-----------|-------|
| Read | 100 / minute |
| Write | 20 / minute |
| Delete | 5 / minute |

## Audit

All mutating agent actions are logged:

```json
{
  "timestamp": "2025-01-15T10:30:00Z",
  "agent": "claude-code",
  "action": "files.create",
  "resource": "abc123",
  "dryRun": false,
  "userConfirmed": true
}
```
