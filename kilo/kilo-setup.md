---
type: Note
related_to: "[[tolaria]]"
status: Active
_width: wide
_organized: true
---

# Kilo Setup

## Overview

Kilo is an AI coding agent CLI. Matt runs multiple concurrent Kilo sessions across many repos. This note documents the full global configuration so it's easy to audit, reproduce, or hand off.

## Config locations

| What | Path |
|---|---|
| Global `kilo.json` | `~/.config/kilo/kilo.jsonc` |
| Global rules (system prompt injections) | `~/.kilocode/rules/*.md` |
| Kilocode plugin | `~/.kilocode/package.json` |
| Status notes vault | `~/src/matt_notes/` |
| Status notes — kilo setup | `~/src/matt_notes/kilo/kilo-setup.md` (this file) |

---

## Global kilo.jsonc (`~/.config/kilo/kilo.jsonc`)

Key settings as of 2026-07-11:

- **Default model**: `openrouter/openai/gpt-5.3-codex`
- **MCP server**: `codebase-memory-mcp` — runs locally at `/Users/mattperzel/.local/bin/codebase-memory-mcp serve`
- **`bash`**: globally allowed
- **`edit`**: globally allowed
- **`skill`**: globally allowed
- **`task`**: globally allowed
- **`external_directory`**: default `ask`, with a large allow-list of project paths (see below)

### External directory allow-list

All of the following paths are pre-approved (no prompt):

```
/Users/mattperzel/src/geotab_shipping_parsing/**
/Users/mattperzel/src/aoportal/**
/Users/mattperzel/src/billing_api/**
/Users/mattperzel/src/inventory-order-widget/packages/myadmin/src/v2/**
/Users/mattperzel/src/advone-sdk/packages/geotab_admin_lib/src/lib/**
/Users/mattperzel/financeFlows/2026-04/**
/Users/mattperzel/src/advone-internal/packages/shipping_notification_lambda/**
/Users/mattperzel/.local/share/kilo/plans/**
/Users/mattperzel/src/mastra/**
/var/folders/.../T/kilo/**   (macOS temp)
/Users/mattperzel/src/engineHoursAudit/**
/Users/mattperzel/src/bulkOrderTool/**
/Users/mattperzel/kiewitenginehouers/AlternateData/engine-hours/2026/07/03/**
/tmp/**
/Users/mattperzel/src/matt_notes/**   ← added for status notes
```

### MCP tool permissions

All `codebase-memory-mcp_*` tools (`search_graph`, `get_code_snippet`, `search_code`, `trace_path`, `list_projects`, `get_architecture`) are globally allowed.

---

## Global rules (`~/.kilocode/rules/`)

Rules are markdown files loaded as system prompt injections by the `@kilocode/plugin` package. All rules are global (apply to every session).

### `codebase-memory-mcp.md`

Instructs Kilo to always prefer the codebase knowledge graph MCP tools over grep/glob/file-search for code discovery. Priority order: `search_graph` → `trace_path` → `get_code_snippet` → `query_graph` → `get_architecture`. Falls back to grep/glob only for string literals, config values, and non-code files.

### `project-status.md`

**Added 2026-07-11.** Instructs Kilo to maintain a per-project status markdown in `~/src/matt_notes/<repo-name>.md` (this Tolaria vault). Purpose: handoff notes and cross-session awareness since Matt runs many concurrent Kilo sessions.

- Session start: read existing status file for prior context
- After significant tasks: update in-progress / completed / next-steps sections
- Session end: write final summary

Status file format uses Tolaria-compatible frontmatter (`type: ProjectStatus`, `related_to: "[[tolaria]]"`).

---

## Plugin (`~/.kilocode/`)

```json
{
  "dependencies": {
    "@kilocode/plugin": "7.2.20"
  }
}
```

The `@kilocode/plugin` package is what loads `~/.kilocode/rules/*.md` into the system prompt. It is installed via `npm`/`bun` in `~/.kilocode/`.

---

## Intent / why this setup exists

Matt runs a large number of simultaneous Kilo sessions across many different repos. The goals of this global config are:

1. **Knowledge graph first** — `codebase-memory-mcp` is always available so Kilo can navigate large codebases structurally rather than grepping.
2. **Status tracking** — every session automatically maintains a living status note in the Tolaria vault so work is never lost between sessions and context can be handed off or resumed.
3. **Broad bash/edit permissions** — reduces interruptions; most repos are already in the allow-list.
4. **Single model default** — `gpt-5.3-codex` via OpenRouter as the global default.
