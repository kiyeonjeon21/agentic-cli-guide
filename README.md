# Agentic CLI Guide

> A practical guide to designing CLIs that AI agents can use effectively
>
> Based on [You Need to Rewrite Your CLI for AI Agents](https://justin.poehnelt.com/posts/rewrite-your-cli-for-ai-agents/) by Justin Poehnelt

---

## Core principles

**Human DX** optimizes discoverability and forgiveness.
**Agent DX** optimizes predictability and defense-in-depth.

Because these two paradigms are fundamentally different, bolting agent support onto a human-first CLI is not enough.

---

## Human DX vs Agent DX

| Aspect | Human DX | Agent DX |
|--------|----------|----------|
| **Input format** | `--title "Q1" --locale en_US` (bespoke flags) | `--json '{"properties":{"title":"Q1"}}'` (raw JSON payload) |
| **Documentation** | `--help`, man page, web docs | `schema` / `--describe` (runtime introspection) |
| **Output format** | Human-friendly tables/color | `--output json`, NDJSON, field masks |
| **Error handling** | "Did you mean...?" suggestions | Clear error codes + JSON error responses |
| **Input validation** | Lenient parsing, auto-correction | Strict validation (block path traversal, control chars) |
| **How to operate** | Tutorials, examples | Skill files (YAML frontmatter + invariants) |
| **Authentication** | Browser OAuth redirect | Environment variables / credential file injection |
| **Dangerous operations** | Confirmation prompts | `--dry-run` + explicit safeguards |
| **Integration** | Pipes `\|`, shell scripts | MCP (JSON-RPC), extensions, structured calls |

---

## Seven design principles

| # | Principle | Keywords | Details |
|---|-----------|----------|---------|
| 1 | [Raw JSON Payloads](docs/01-json-payloads.md) | Nested structure, 1:1 API schema mapping | JSON instead of bespoke flags |
| 2 | [Schema Introspection](docs/02-schema-introspection.md) | Runtime self-description, discovery document | Replaces static docs |
| 3 | [Context Window Discipline](docs/03-context-window.md) | Field masks, NDJSON, token savings | Control response size |
| 4 | [Input Hardening](docs/04-input-hardening.md) | Hallucination defense, path traversal, double encoding | Distrust agent input |
| 5 | [Skill Files](docs/05-skill-files.md) | CONTEXT.md, SKILL.md, invariants | Tacit knowledge for agents |
| 6 | [Multi-Surface](docs/06-multi-surface.md) | CLI + MCP + extension + env auth | One binary, multiple interfaces |
| 7 | [Safety Rails](docs/07-safety-rails.md) | dry-run, sanitize, prompt-injection defense | Safe agent execution |

---

## Quick start: making an existing CLI agent-friendly

```
1. Add --output json                     → machine-readable output
2. Harden input validation                → block control chars, path traversal
3. Implement schema / --describe          → runtime introspection
4. Support --fields / field masks         → protect context window
5. Add --dry-run                          → safe change rehearsal
6. Ship CONTEXT.md / skill files         → invariants for agents
7. Expose an MCP surface                  → structured tool calls
```

See the full checklist at [checklist/CHECKLIST.md](checklist/CHECKLIST.md).

---

## FAQ

**Q: Do I need to rewrite my CLI from scratch?**
No. Most patterns can be added incrementally. Start with `--output json` and input validation, then layer on schema introspection and skill files.

**Q: What if my CLI doesn't wrap a REST API?**
The principles still apply. Any CLI that agents invoke needs machine-readable output, input hardening, and explicit documentation of invariants. Schema introspection is most valuable for API-backed CLIs, but `--describe` or `--help --json` works for anything.

**Q: How do I handle auth for agents?**
Environment variables for tokens and credential file paths. Service accounts where possible. Avoid flows that require a browser redirect.

**Q: Is MCP worth the investment?**
If your CLI wraps a structured API, yes. MCP eliminates shell escaping, argument parsing ambiguity, and output parsing. The agent calls a typed function instead of constructing a string.

**Q: How do I test that my CLI is agent-safe?**
Fuzz your inputs with the kinds of mistakes agents make — path traversals, embedded query params, double-encoded strings, control characters. `--dry-run` should catch issues before they hit your API.

---

## Reference implementation

- [Google Workspace CLI (gws)](https://github.com/googleworkspace/cli) — open-source reference that applies all principles in this guide

---

## Repository layout

```
agentic-cli-guide/
├── README.md                       # This document
├── checklist/
│   └── CHECKLIST.md                # Prioritized checklist
├── docs/
│   ├── 01-json-payloads.md         # Raw JSON vs bespoke flags
│   ├── 02-schema-introspection.md  # Runtime schema introspection
│   ├── 03-context-window.md        # Field masks + NDJSON
│   ├── 04-input-hardening.md       # Hallucination defense, fuzz patterns
│   ├── 05-skill-files.md           # Writing SKILL.md / CONTEXT.md
│   ├── 06-multi-surface.md         # CLI + MCP + env var auth
│   └── 07-safety-rails.md          # dry-run + prompt injection defense
└── examples/
    ├── context-files/
    │   ├── CONTEXT.md              # Agent context template
    │   └── AGENTS.md               # Security model declaration template
    └── skill-files/
        └── SKILL.md                # Skill file template with YAML frontmatter
```

---

## License

This guide is derived from [You Need to Rewrite Your CLI for AI Agents](https://justin.poehnelt.com/posts/rewrite-your-cli-for-ai-agents/) by Justin Poehnelt, licensed under [CC BY-SA 4.0](https://creativecommons.org/licenses/by-sa/4.0/).

This derivative work is also licensed under CC BY-SA 4.0.
