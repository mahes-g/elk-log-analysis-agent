---
name: elk-log-analysis
description: Analyze ELK (Elasticsearch/Kibana) logs using the ELK MCP server, correlating events by traceId or sessionId. Use when investigating errors, tracing requests across services, or debugging production issues in Kibana.
---

# ELK Log Analysis Skill

Analyze logs stored in Elasticsearch via the **ELK MCP server** by correlating events on `traceId` or `sessionId`.

## Prerequisites

- ELK MCP server configured and reachable (see `mcp-config.example.json`).
- Index pattern known (e.g. `logs-*`, `applogs-*`).
- Field names for trace/session IDs (defaults below; adjust to your mapping).

## Default Field Names

| Purpose      | Field candidates (check your mapping)              |
|--------------|-----------------------------------------------------|
| Trace ID     | `traceId`, `trace_id`, `@trace.id`, `transaction.id` |
| Session ID   | `sessionId`, `session_id`, `@session.id`            |
| Timestamp    | `@timestamp`                                        |
| Level        | `level`, `log.level`, `severity`                    |
| Message      | `message`, `msg`, `log`                             |
| Service      | `service.name`, `kubernetes.labels.app`, `host`     |

## Workflow

### 1. Resolve the identifier type

Ask the user (or infer) whether the ID is a **traceId** (distributed tracing, per-request) or **sessionId** (user session, multi-request).

- `traceId` → typically one request across services; short time window.
- `sessionId` → a user session; may span minutes/hours and many requests.

### 2. Discover the correct field name

If unsure which field holds the ID, run a mapping/field-capabilities query via the ELK MCP tools to confirm the exact field name before searching. Common pitfall: `traceId` vs `trace_id` vs nested `@trace.id` — a wrong field returns zero hits silently.

### 3. Build the query

Use a `match`/`term` query on the resolved field. Always bound the time range — never run unbounded queries on large indices.

Trace lookup (tight window, e.g. ±5 min around known time, or last 24h if unknown):

```json
{
  "query": {
    "bool": {
      "must": [
        { "term": { "traceId": "<TRACE_ID>" } },
        { "range": { "@timestamp": { "gte": "now-24h" } } }
      ]
    }
  },
  "sort": [{ "@timestamp": "asc" }],
  "size": 200
}
```

Session lookup (wider window):

```json
{
  "query": {
    "bool": {
      "must": [
        { "term": { "sessionId": "<SESSION_ID>" } },
        { "range": { "@timestamp": { "gte": "now-7d" } } }
      ]
    }
  },
  "sort": [{ "@timestamp": "asc" }],
  "size": 500
}
```

### 4. Paginate safely

If `total > size`, use `search_after` (preferred) or `from`/`size` with small pages. Do not deep-page with large `from` values — it is slow and can hit `index.max_result_window` (default 10,000).

### 5. Analyze and correlate

- Sort by `@timestamp` ascending to reconstruct the event timeline.
- Group by `service.name` / `host` to see the request path across services.
- Filter to `level: ERROR|WARN` first for a quick failure summary, then widen.
- For traceIds: look for the span where `duration` spikes or an exception stack trace appears; the first ERROR in trace order is usually the root cause.
- For sessionIds: reconstruct the user journey — page views, API calls, errors — and correlate errors with the actions immediately preceding them.

### 6. Summarize findings

Report:

- **Timeline** of key events (first/last seen, duration).
- **Services involved** and the call path.
- **Errors/exceptions** with stack traces (trim to relevant frames).
- Probable **root cause** and suggested next steps.

## Common Issues

- **Zero hits**: wrong field name, wrong index pattern, or ID outside the time window. Verify field via mapping API and widen `gte`.
- **Too many hits (sessionId)**: add filters — service, level, or a narrower time range around the incident.
- **Timeouts**: reduce `size`, tighten the time range, or query a specific index instead of a wildcard pattern.
- **Case sensitivity**: `term` queries are exact; use `match` for analyzed text fields, but prefer `keyword` subfields for IDs.

## Example Prompts

- "Find all logs for traceId `4bf92f3577b34da6a3ce929d0e0e4736` in the last 24 hours and tell me what failed."
- "Reconstruct the session for sessionId `sess_abc123` — what was the user doing before the 500 errors?"
