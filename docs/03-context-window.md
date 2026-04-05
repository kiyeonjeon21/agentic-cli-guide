# 03. Context Window Discipline

## The problem

Agent context windows are finite. A single API response returning thousands of lines of JSON:

- Burns the token budget quickly
- Degrades reasoning (signal buried in noise)
- Spikes cost

For example, listing Google Drive files without a field mask can return 50+ fields per file. One hundred files can cost tens of thousands of tokens by themselves.

## Solution 1: field masks

Request only the fields you need.

```bash
# All fields (bad)
my-cli files list --output json

# Only needed fields (good)
my-cli files list --output json --fields "files(id,name,mimeType)"
```

Comparison:

```json
// Without --fields: ~50 fields per file
{
  "id": "abc123",
  "name": "Q1 Report.docx",
  "mimeType": "application/vnd.google-apps.document",
  "kind": "drive#file",
  "starred": false,
  "trashed": false,
  "parents": ["folder123"],
  "webViewLink": "https://...",
  "iconLink": "https://...",
  "thumbnailLink": "https://...",
  "createdTime": "2024-01-15T...",
  "modifiedTime": "2024-03-20T...",
  // ... 40+ more fields
}

// --fields "files(id,name,mimeType)": 3 fields per file
{
  "id": "abc123",
  "name": "Q1 Report.docx",
  "mimeType": "application/vnd.google-apps.document"
}
```

### Implementation guidance

```bash
# Google-style field mask
my-cli files list --fields "files(id,name),nextPageToken"

# GraphQL-style
my-cli files list --select "id,name,mimeType"

# jq-style post-processing (client-side, not server-side)
my-cli files list --output json | jq '{id, name}'
```

Prefer **server-side** field masks when available — less bandwidth and often less server work.

## Solution 2: NDJSON streaming

Returning huge lists as one JSON array forces agents to buffer everything. NDJSON (newline-delimited JSON) prints one JSON object per line for stream processing.

```bash
# Classic JSON array (must buffer all)
my-cli files list --output json
# => {"files": [{"id":"a"}, {"id":"b"}, ... ]}

# NDJSON (line-oriented stream)
my-cli files list --output ndjson --page-all
# => {"id":"a","name":"file1"}
# => {"id":"b","name":"file2"}
# => {"id":"c","name":"file3"}
```

### Why NDJSON helps

1. **Incremental processing** — handle lines without waiting for the full response
2. **Memory efficient** — no need to buffer the full array
3. **Pagination abstracted** — `--page-all` can fetch pages while streaming lines
4. **Pipe-friendly** — `head -n 10` for the first 10 items

### Implementation guidance

```bash
# Auto-pagination + NDJSON
my-cli files list --page-all --output ndjson --fields "id,name"

# Cap how many items to pull
my-cli files list --page-all --output ndjson --max-results 50

# Compose with pipes
my-cli files list --page-all --output ndjson | head -n 5
```

## Invariants to state in skill files

Include the following in skills or CONTEXT.md for agents:

```markdown
## Rules

- Always use `--fields` when listing or getting resources
- Prefer NDJSON (`--output ndjson`) for list operations
- Never fetch all fields when only a subset is needed
- Use `--max-results` to limit response size when full listing is unnecessary
```
