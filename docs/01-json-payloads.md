# 01. Raw JSON Payloads Over Bespoke Flags

## The problem

Human-first CLIs flatten every option into individual flags:

```bash
my-cli spreadsheet create \
  --title "Q1 Budget" \
  --locale "en_US" \
  --timezone "America/Denver" \
  --sheet-title "January" \
  --sheet-type GRID \
  --frozen-rows 1 \
  --frozen-cols 2 \
  --row-count 100 \
  --col-count 10 \
  --hidden false
```

This feels natural to humans but causes three problems for agents:

1. **No nested structure** — Flags are inherently flat. Collapsing something like `sheets[0].properties.gridProperties.frozenRowCount` into `--frozen-rows` introduces a translation layer.
2. **Mismatch with the API** — When agents build requests from API docs, different flag names than API field names cause mapping errors. Humans typo. Agents hallucinate.
3. **Combinatorial explosion** — Each API extension needs new flags, and the surface agents must learn keeps growing.

## Solution: pass JSON payloads directly

```bash
my-cli spreadsheet create --json '{
  "properties": {
    "title": "Q1 Budget",
    "locale": "en_US",
    "timeZone": "America/Denver"
  },
  "sheets": [{
    "properties": {
      "title": "January",
      "sheetType": "GRID",
      "gridProperties": {
        "frozenRowCount": 1,
        "frozenColumnCount": 2,
        "rowCount": 100,
        "columnCount": 10
      },
      "hidden": false
    }
  }]
}'
```

JSON maps 1:1 to the API schema. LLMs are strong at structured JSON, and this is effectively the same as calling the API directly without a translation layer.

## Implementation guidelines

### Support both paths

```bash
# Human path: convenience flags
my-cli users create --name "Alice" --role admin

# Agent path: raw JSON
my-cli users create --json '{"name": "Alice", "role": "admin"}'
```

In one binary, support both; when `--json` is provided, ignore individual flags or error on conflicts.

### Accept JSON from stdin

```bash
# Pipe input
echo '{"name": "Alice"}' | my-cli users create --json -

# Read from file
my-cli users create --json @payload.json
```

### Emit JSON as output

```bash
# Explicit flag
my-cli users list --output json

# Environment variable
OUTPUT_FORMAT=json my-cli users list

# TTY detection: auto JSON when stdout is not a TTY
my-cli users list | jq '.users[].name'
```

## Caveats

- Field names in the JSON payload must match the API schema exactly (unify camelCase vs snake_case).
- When agents send unknown fields, do not silently ignore; return a clear error:
  ```json
  {"error": {"code": "INVALID_FIELD", "message": "Unknown field 'titel' in properties. Did you mean 'title'?"}}
  ```
- JSON parse errors should include location information when possible.
