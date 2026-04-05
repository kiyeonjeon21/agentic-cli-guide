# 05. Skill Files — conveying tacit knowledge to agents

## The problem

Agents do not learn CLIs by exploring `--help`. They learn from **injected context files**.

Experienced developers follow unwritten rules:

- "Run dry-run before mutating"
- "Always apply a field mask on list calls"
- "This API is strict on rate limits — batch requests"

That **implicit knowledge** is rarely in `--help`. Skill files capture it in a structured form agents can read.

## What a skill file is

YAML frontmatter plus Markdown body. Agent frameworks load it into context.

Skill files follow the [Agent Skills](https://agentskills.io) open standard (OpenClaw), which works across multiple AI tools. The Google Workspace CLI ships over 100 skill files using this format.

Core pieces:

- **Frontmatter**: metadata (name, version, prerequisites)
- **Rules**: invariants the agent must follow
- **Examples**: correct usage patterns

## Writing SKILL.md

YAML frontmatter fields depend on the agent framework. Two major formats exist.

### OpenClaw / Agent Skills format

The format used by the original blog post and the Google Workspace CLI:

```yaml
---
name: gws-drive-upload
version: 1.0.0
metadata:
  openclaw:
    requires:
      bins: ["gws"]
---
```

### Claude Code format

Claude Code extends the Agent Skills standard with additional fields. The supported fields are:

| Field | Required | Description |
|-------|----------|-------------|
| `name` | No | Lowercase letters, numbers, hyphens (max 64 chars). Defaults to directory name. |
| `description` | Recommended | What the skill does and when to use it. Front-load the key use case (≤250 chars). |
| `disable-model-invocation` | No | `true` to prevent automatic loading — manual `/name` only. |
| `allowed-tools` | No | Tools the agent may use without per-call approval. Space-separated or YAML list. |
| `context` | No | `fork` to run in an isolated subagent. |
| `agent` | No | Subagent type when `context: fork` (e.g. `Explore`, `Plan`). |
| `user-invocable` | No | `false` to hide from the `/` menu (background knowledge only). |
| `model` | No | Model override while the skill is active. |
| `effort` | No | Effort level override (`low`, `medium`, `high`, `max`). |
| `argument-hint` | No | Hint shown during autocomplete, e.g. `[issue-number]`. |
| `paths` | No | Glob patterns limiting when the skill activates. |

Example frontmatter:

```yaml
---
name: my-cli-drive-operations
description: Rules and patterns for using my-cli to manage Drive files. Use when interacting with Drive via my-cli.
---
```

Markdown body (excerpt):

```markdown
# my-cli Drive Operations

## Rules

1. Always use `--dry-run` before executing mutating operations (create, update, delete)
2. Always confirm with the user before executing write/delete commands
3. Always add `--fields` to every list/get call to minimize response size
4. Use `--output json` for all commands
5. Never hardcode file IDs — always resolve by name first
```

Authentication example:

```bash
# Auth via environment variables (preferred in agent environments)
export MY_CLI_TOKEN="$TOKEN"
export MY_CLI_CREDENTIALS_FILE="/path/to/service-account.json"
```

Find → download:

```bash
# 1. Search by name (field mask required)
my-cli files list --json '{"q": "name = '\''Report.pdf'\''"}' \
  --fields "files(id,name,mimeType)" --output json

# 2. Validate download with dry-run
my-cli files export --json '{"fileId": "RESOLVED_ID"}' \
  --output-file ./report.pdf --dry-run

# 3. Actual download
my-cli files export --json '{"fileId": "RESOLVED_ID"}' \
  --output-file ./report.pdf
```

Upload:

```bash
# 1. Validate upload with dry-run
my-cli files create --json '{"name": "Budget.xlsx", "parents": ["FOLDER_ID"]}' \
  --upload-file ./budget.xlsx --dry-run

# 2. Run after user confirms
my-cli files create --json '{"name": "Budget.xlsx", "parents": ["FOLDER_ID"]}' \
  --upload-file ./budget.xlsx
```

**Error handling**

| Error code | Meaning | Agent response |
|------------|---------|----------------|
| 404 | Not found | Re-check ID; search by name again |
| 403 | Forbidden | Ask user to grant permissions |
| 429 | Rate limited | Exponential backoff, retry |

## Writing CONTEXT.md

CONTEXT.md is not skill-specific; it is the global agent guide for the whole CLI.

```markdown
# my-cli Agent Context

## Overview

my-cli wraps the SaaS platform’s HTTP API.

## Global Rules

1. Use `--output json` for all output
2. Run `--dry-run` before any mutating operation
3. Use `--fields` on list/get to request only needed fields
4. Never guess resource IDs — resolve via search

## Available Services

- `files` — file CRUD
- `users` — user administration
- `permissions` — permission management

## Schema Introspection

(See shell examples below.)

## Security Model

- Show `--dry-run` results and obtain approval before executing
- Deletes always need explicit user confirmation
- Do not operate outside the resource scope defined in `AGENTS.md`
```

Schema introspection examples for that section:

```bash
# All methods for a service
my-cli schema --list-methods files

# Parameters for a specific method
my-cli schema files.create
```

## Impact of skill files

| Aspect | Without skills | With skills |
|--------|----------------|-------------|
| Token cost | Fail → retry → burn context | Correct pattern on first try |
| Safety | Easy to skip dry-run | Invariants enforce behavior |
| Consistency | Per-agent variation | Standard patterns |
| Debugging | Opaque choices | Traceable to documented rules |

## Distribution

1. **Repo root**: ship `CONTEXT.md`, `SKILL.md` with CLI source
2. **Packages**: include with `npm install`, `pip install`, etc.
3. **Runtime**: `my-cli context` prints CONTEXT.md
4. **Agent integration**: MCP server embeds skill content in tool descriptions
