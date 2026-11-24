# ELK Log Analysis Skill

An agent skill that analyzes Elasticsearch/Kibana logs through the **ELK MCP server**, correlating events by **`traceId`** or **`sessionId`** to reconstruct request flows, user sessions, and root causes of failures.

## What It Does

Given a trace ID or session ID, the skill:

1. **Resolves the identifier type** — `traceId` (single distributed request) vs `sessionId` (multi-request user session).
2. **Discovers the correct field name** — mappings differ across stacks (`traceId` vs `trace_id` vs `@trace.id`).
3. **Builds a bounded query** — always time-ranged to keep searches fast and cheap.
4. **Paginates safely** — `search_after` over deep `from` paging.
5. **Correlates and summarizes** — timeline, service call path, errors, and probable root cause.

## Repository Structure

```
elk-log-analysis-skill/
├── skills/
│   └── elk-log-analysis/
│       └── SKILL.md              # The skill definition (agent instructions)
├── mcp-config.example.json       # Example ELK MCP server configuration
└── README.md                     # This file
```

## Prerequisites

- An MCP-compatible agent/IDE (e.g. Claude Desktop, Windsurf, Cursor).
- The [Elastic MCP server for Elasticsearch](https://www.elastic.co/docs/explore-analyze/ai-features/mcp-server) (or equivalent ELK MCP) configured and reachable.
- Knowledge of your index pattern (e.g. `logs-*`) and the field names used for trace/session IDs.

## Setup

### 1. Configure the ELK MCP server

Copy `mcp-config.example.json` and fill in your Elasticsearch URL and credentials. For Claude Desktop, place it under `mcpServers` in `claude_desktop_config.json`; for other agents, follow their MCP config conventions.

```json
{
  "mcpServers": {
    "elk": {
      "command": "npx",
      "args": ["-y", "@elastic/mcp-server-elasticsearch"],
      "env": {
        "ES_URL": "https://your-elasticsearch-host:9200",
        "ES_API_KEY": "your-api-key"
      }
    }
  }
}
```

### 2. Install the skill

Copy the `skills/elk-log-analysis/` folder into your agent's skills directory:

- **Claude Code / Claude skills**: `~/.claude/skills/elk-log-analysis/SKILL.md`
- **Other agents**: consult your agent's skill/plugin docs; the format is a `SKILL.md` with YAML frontmatter.

### 3. Verify field names

Before first use, confirm your trace/session field names via Kibana Dev Tools or the mapping API:

```json
GET logs-*/_mapping/field/traceId,sessionId
```

Update the "Default Field Names" table in `SKILL.md` if yours differ.

## Usage

Once installed, ask your agent things like:

- "Find all logs for traceId `4bf92f3577b34da6a3ce929d0e0e4736` in the last 24 hours and tell me what failed."
- "Reconstruct the session for sessionId `sess_abc123` — what was the user doing before the 500 errors?"
- "Show me the error timeline for session `xyz` across all services."

## How It Works

The skill encodes best practices that generic MCP usage often misses:

- **Identifier disambiguation** — trace vs session implies different time windows and analysis depth.
- **Field discovery** — prevents silent zero-hit queries from wrong field names.
- **Bounded queries** — `range` on `@timestamp` avoids timeouts on large indices.
- **Safe pagination** — `search_after` instead of deep `from` paging.
- **Structured output** — timeline, call path, errors, root cause.

## Customization

- Edit `SKILL.md` to match your index patterns, field names, and severity conventions.
- Add organization-specific runbooks (e.g. known noisy services to exclude).

## License

MIT
