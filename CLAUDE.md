# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A personal Claude Code plugin marketplace. Each top-level directory (e.g., `plan-harness/`) is a standalone plugin that can be installed via `claude plugins install`. The root `.claude-plugin/marketplace.json` registers all plugins for CLI discovery.

## Architecture

```
canwa-claude-plugins/
  .claude-plugin/marketplace.json   # Root marketplace registry — lists all plugins
  plan-harness/                     # Plugin: structured project planning
    .claude-plugin/
      plugin.json                   # Plugin metadata (name, version, description)
      marketplace.json              # Plugin-level marketplace entry
    .mcp.json                       # MCP server config (wires plan-harness MCP to Claude Code)
    skills/                         # One SKILL.md per skill (11 total)
      plan-init/SKILL.md
      plan-design/SKILL.md
      ...
    prompts/                        # Agent role system prompts (architect, PM, tester, writer, frontend-dev, backend-dev)
    contexts/                       # Built-in context templates (.md) — copied to workspace on first use
    local-proxy/                    # Node.js MCP server (stdio transport)
      start.js                      # Bootstrap: auto-installs deps, then imports src/index.js
      src/
        index.js                    # MCP server — registers 6 tools via @modelcontextprotocol/sdk
        plan-manager.js             # File-system operations: scenario CRUD, completion checking, codebase context
        web-server.js               # HTTP dashboard server (node:http, no external deps beyond SDK)
        templates/base.js           # Self-contained HTML template system (inline CSS/JS, dark/light theme)
    docs/overview.html              # Static overview doc
```

### Key Design Decisions

- **MCP server communicates over stdio** — all user-visible logging must go to `stderr` (`console.error`), because `stdout` is reserved for the JSON-RPC channel.
- **HTML output is fully self-contained** — every generated plan document embeds all CSS and JS inline; no external dependencies, no CDN links.
- **Templates use CSS custom properties** for dark/light theming with `[data-theme]` attribute switching.
- **Plan files live in `plans/<scenario-name>/` directories** within the target repo (not this repo). The MCP server scans for these at runtime.
- **start.js auto-installs npm deps** on first run — no manual `npm install` required.
- **Contexts are composable markdown files** — like CLAUDE.md but for specific scenarios. Each context `.md` file (stored in `plans/.contexts/`) provides project knowledge, generation rules, or both. `/plan-init` multi-selects contexts; they compose in order (later overrides earlier). Contexts range from broad (whole project) to narrow (specific pages/APIs). The more specific, the better the LLM output.
- **Context is mandatory** — `/plan-init` always presents a context multi-selector. No auto-detection, no skip. Built-in templates (feature-planning, performance-audit, lean) ship in `contexts/` and are copied to workspace on first use.

## MCP Server Tools (plan-harness)

Six tools exposed via `src/index.js`: `plan_list_scenarios`, `plan_create_scenario`, `plan_get_files`, `plan_check_completion`, `plan_get_context`, `plan_serve_dashboard`.

## Development

```bash
# Run the MCP server directly (for testing)
node plan-harness/local-proxy/start.js

# Install deps manually (if needed)
cd plan-harness/local-proxy && npm install
```

No test suite or linter is configured. The MCP server has a single dependency: `@modelcontextprotocol/sdk`.

## Adding a New Plugin

1. Create a directory at root: `my-plugin/`
2. Add `.claude-plugin/plugin.json` with metadata
3. Add skills in `skills/`, MCP server in `local-proxy/`, agent prompts in `prompts/`
4. Register in root `.claude-plugin/marketplace.json` under `plugins[]`
