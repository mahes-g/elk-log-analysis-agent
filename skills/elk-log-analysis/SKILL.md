---
name: elk-log-analysis
description: Analyze ELK (Elasticsearch/Kibana) logs using the ELK MCP server or Elasticsearch REST API, correlating events by traceId, sessionId, or request/correlation ID and producing a detailed root cause analysis (RCA) summary covering errors, downstream issues, timeouts, network/database latencies, code issues, system errors, recurrence patterns, and recommended solutions. Use when investigating errors, tracing requests across services, or debugging production issues in Kibana.
---

# ELK Log Analysis Skill

Analyze logs stored in Elasticsearch via the **ELK MCP server** (or the Elasticsearch REST API as a fallback) by correlating events on `traceId`, `sessionId`, or request/correlation IDs.

## Prerequisites

- ELK MCP server configured and reachable (see `mcp-config.example.json`).
- Index pattern known (e.g. `logs-*`, `applogs-*`).
- Field names for trace/session IDs (defaults below; adjust to your mapping).

## MCP Requirements

This skill expects an MCP server that provides Elasticsearch access.

Typical operations used by this skill:

- Discover mappings
- Discover fields
- Search documents
- Aggregate results
- List indices

The exact MCP tool names depend on the implementation.

### Supported MCP implementations

This skill works with any MCP server exposing Elasticsearch search capabilities. Two common setups:

- **Standalone Elasticsearch MCP server** — e.g. the reference server described in [Connect Agents to Elasticsearch with Model Context Protocol](https://www.elastic.co/search-labs/blog/model-context-protocol-elasticsearch). Configure it in your agent's MCP settings (see `mcp-config.example.json`).
- **Elastic Agent Builder MCP tools** — if you run [Elastic Agent Builder](https://www.elastic.co/docs/explore-analyze/ai-features/agent-builder), you can expose Elasticsearch operations as MCP tools via an [MCP connector](https://www.elastic.co/docs/reference/kibana/connectors-kibana/mcp-action-type). See [MCP tools in Elastic Agent Builder](https://www.elastic.co/docs/explore-analyze/ai-features/agent-builder/tools/mcp-tools) for how tool calls are routed: Agent Builder fetches the tool's input schema, calls the MCP server, and returns the raw result to the LLM. Tools can be imported individually or in bulk, and unhealthy tools (connector down, tool removed, execution failure) are flagged in the Agent Builder UI — check tool health first if queries unexpectedly fail.

### MCP behavior notes

- The MCP server returns results **directly to the LLM with no post-processing** — the skill's query design (field names, time bounds, pagination) is what guarantees quality.
- Prefer resolving field names via mapping/field-capabilities calls **before** searching, matching the "lookup index mapping first, then construct the query" pattern recommended in Elastic's MCP guidance.
- If using Agent Builder, verify the MCP connector is healthy before starting analysis; an unavailable connector produces tool-call failures, not empty results.

### Fallback: direct Elasticsearch REST API

If no MCP server is available but you have access to the Elasticsearch REST API (HTTP endpoint + credentials/API key), perform the same operations directly. All MCP operations used by this skill map to REST equivalents:

| MCP operation | REST API equivalent |
|---------------|---------------------|
| List indices | `GET /_cat/indices/<pattern>?format=json` or `GET /_resolve/index/<pattern>` |
| Discover mappings | `GET /<index-pattern>/_mapping` or `GET /<index-pattern>/_mapping/field/<field>` |
| Discover fields | `GET /<index-pattern>/_field_caps?fields=*` |
| Search documents | `POST /<index-pattern>/_search` (same query DSL as the MCP path) |
| Aggregate results | `POST /<index-pattern>/_search` with an `aggs` block |

Example trace lookup via REST:

```bash
curl -s -X POST "https://<es-host>:9200/logs-*/_search" \
  -H "Authorization: ApiKey <api-key>" \
  -H "Content-Type: application/json" \
  -d '{
    "query": {"bool": {"must": [
      {"term": {"traceId": "<TRACE_ID>"}},
      {"range": {"@timestamp": {"gte": "now-24h"}}}
    ]}},
    "sort": [{"@timestamp": "asc"}],
    "size": 200
  }'
```

Notes for the REST path:

- Use `POST` for `_search` (avoids URL-length issues with large queries).
- Auth: `Authorization: ApiKey <key>` header, or Basic auth. Never embed credentials in committed files.
- If only Kibana is reachable (no direct ES access), use Kibana's **Dev Tools Console** — it proxies the same REST API and is the closest equivalent.
- The query DSL, field discovery, pagination (`search_after`), and RCA workflow in this skill are identical regardless of transport — only the invocation mechanism changes.

## Default Field Names

| Purpose      | Field candidates (check your mapping)              |
|--------------|-----------------------------------------------------|
| Trace ID     | `traceId`, `trace_id`, `@trace.id`, `transaction.id` |
| Session ID   | `sessionId`, `session_id`, `@session.id`            |
| Request/Correlation ID | `requestId`, `correlationId`, `correlation_id`, `x-request-id`, `spanId`, `span_id` |
| Timestamp    | `@timestamp`                                        |
| Level        | `level`, `log.level`, `severity`                    |
| Message      | `message`, `msg`, `log`                             |
| Service      | `service.name`, `kubernetes.labels.app`, `host`     |

## Workflow

### 1. Resolve the identifier type

Ask the user (or infer) whether the ID is a **traceId** (distributed tracing, per-request), **sessionId** (user session, multi-request), or a **request/correlation ID** (`requestId`, `correlationId`, `spanId` — single-request identifiers used when tracing isn't instrumented).

- `traceId` → typically one request across services; short time window.
- `sessionId` → a user session; may span minutes/hours and many requests.
- `requestId`/`correlationId`/`spanId` → single request or single span; narrowest scope. If only a `spanId` is available, search it first, then expand to the parent `traceId` if present in the same document.

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

### 6. Classify the failure

Tag every significant event into one or more RCA categories:

| Category | Signals to look for |
|----------|---------------------|
| **Application errors** | Exception stack traces, `ERROR`/`FATAL` logs, HTTP 5xx responses, unhandled rejections |
| **Downstream dependency issues** | Errors/timeouts calling other services, DBs, caches, queues, third-party APIs — identify *which* dependency and whether the failure is upstream or downstream of it |
| **Timeouts** | `timeout`, `deadline exceeded`, `ReadTimeout`, `ETIMEDOUT`, `504 Gateway Timeout`, retry exhaustion, circuit-breaker open events |
| **Network issues** | Connection refused/reset, DNS resolution failures, TLS/SSL handshake errors, socket errors, latency spikes without app errors |
| **Database issues** | Slow query logs, connection pool exhaustion (`HikariPool`, `pool timeout`), deadlocks, lock wait timeouts, replication lag, `SQLException`/`MongoError`/`RedisError` |
| **Code issues** | `NullPointerException`, `IndexOutOfBounds`, serialization/deserialization failures, recent-deploy correlation, logic errors visible in stack frames |
| **System/resource issues** | OOM kills, CPU/memory pressure, disk full, thread pool exhaustion, GC pauses, pod restarts, node pressure |

For each category found, record: first occurrence timestamp, affected service(s), frequency, and whether it appears to be cause or symptom.

### 7. Produce the RCA summary

Deliver a structured report with these sections:

#### Executive Summary
One paragraph: what failed, when, blast radius (how many requests/users affected), and current status.

#### Timeline
Chronological table of key events: timestamp → service → event → significance. Highlight the inflection point where things went wrong.

#### Failure Classification
For each RCA category triggered, list the evidence (log lines, error messages, stack frames) and mark it as **root cause**, **contributing factor**, or **symptom**.

#### Root Cause Analysis
- **Probable root cause**: the earliest failure in the causal chain, with reasoning.
- **Causal chain**: how the root cause propagated (e.g. DB slow query → connection pool exhaustion → API timeouts → user-facing 504s).
- **Confidence level**: high/medium/low and what evidence is missing.

#### Impact Assessment
- Services and endpoints affected.
- Error rate / latency impact during the window.
- Whether the issue is ongoing, resolved, or intermittent.

#### Recommended Solutions
Provide actionable fixes ordered by priority:

- **Immediate mitigation** — e.g. restart affected service, scale up, roll back recent deploy, increase timeout/pool size, enable circuit breaker.
- **Short-term fix** — code/config changes addressing the root cause.
- **Long-term prevention** — alerting on the detected signal, adding timeouts/retries where missing, capacity planning, test coverage for the failure mode.

Where the logs suggest a specific config key, dependency, or code location (file/line from stack traces), cite it explicitly.

## Evidence Rules

Every claim in the RCA summary must be backed by evidence from the logs. Follow these rules:

- **Cite every finding** with at least: timestamp, service name, and the relevant log message or field value. Example: `payment-service @ 02:15:47 — "HikariPool-1 - Connection is not available"`.
- **Quote, don't paraphrase, errors** — include the exception class and message verbatim; trim stack traces to the frames relevant to the failure.
- **Distinguish observed vs inferred** — label conclusions as *observed* (directly in logs) or *inferred* (deduced from patterns). Never present inference as fact.
- **No evidence, no claim** — if a category has no supporting log entries, do not list it. If evidence is partial, lower the confidence level and state what is missing.
- **Correlate before concluding** — a single ERROR line is not a root cause; verify it is the earliest failure in the causal chain and not itself a symptom of an upstream issue.
- **Respect data gaps** — if the relevant logs are in a different index, retention window, or system (metrics, APM traces), say so explicitly rather than guessing.

## Recurrence Analysis

Required for **Deep dive** mode; recommended for Standard mode when the error looks systemic. Determine whether the issue is a one-off or a recurring pattern:

1. **Extract the error signature** — normalize the error into a stable signature: exception class + message pattern + failing service (strip IDs, timestamps, and variable values). E.g. `HikariPool-1 - Connection is not available` in `payment-service`.
2. **Count occurrences** — run a `terms` aggregation on the signature components (service + level + a keyword on the normalized message if available) over an extended window (e.g. 7–30 days) to get occurrence counts per day.
3. **Check other identifiers** — search for the same signature under *different* traceIds/sessionIds to measure blast radius beyond the reported incident.
4. **Compare to baseline** — aggregate the same signature over a preceding baseline period (e.g. prior week) to compute whether the rate is elevated, new, or chronic.
5. **Report findings** in the RCA summary under a **Recurrence** subsection: first seen, last seen, total occurrences, affected services, trend (new / recurring / resolved / chronic), and whether prior incidents share the same signature.

## Modes

The skill supports three analysis modes. Infer the mode from the user's request, or ask if ambiguous.

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Quick triage** | "what failed", "quick look", "is it still happening" | Steps 1–5 only. Return error summary + timeline, no full RCA. Fast, small `size`. |
| **Standard analysis** | "analyze", "investigate", "why did X fail" | Steps 1–7. Full RCA summary with classification and recommendations. |
| **Deep dive** | "deep dive", "full RCA", "postmortem" | Standard + wider time window, cross-reference related traceIds/sessionIds found in the data, run Recurrence Analysis, and compare against baseline error rates. |

## Sample Commands

```
# Quick triage
"Quick triage: traceId 4bf92f3577b34da6a3ce929d0e0e4736 — what failed?"

# Standard RCA
"Investigate sessionId sess_abc123 — users report checkout errors. Full RCA please."

# Deep dive / postmortem
"Deep dive on traceId abc123 from the 2 AM incident. Check if the same error hit other sessions and whether it's recurring."

# Filtered investigation
"Analyze traceId xyz but only the payment-service errors in the last 6 hours."

# Compare modes
"First give me a quick triage on sessionId sess_abc123, then if it's a real issue do the full RCA."
```

## Sample RCA Response

Below is an example of the expected output format for a Standard/Deep-dive analysis.

---

### RCA: traceId `4bf92f3577b34da6a3ce929d0e0e4736`

#### Executive Summary
Checkout API requests started failing at **2026-11-24 02:14 UTC** with 504 Gateway Timeouts. The failure originated in `payment-service` due to connection pool exhaustion against the `orders-db` PostgreSQL instance, caused by a slow query introduced in deploy `v2.14.3` at 02:10 UTC. Approximately **340 requests** were affected over 47 minutes. Service recovered at 03:01 UTC after the pool was manually resized; the slow query remains in production.

#### Timeline

| Time (UTC) | Service | Event | Significance |
|------------|---------|-------|--------------|
| 02:10:32 | payment-service | Deploy `v2.14.3` completed | New query added to `OrderRepository.findPending()` |
| 02:14:03 | orders-db | Slow query warning: `findPending()` — 8.2s | First sign of regression |
| 02:14:11 | payment-service | WARN: HikariPool — 18/20 connections in use | Pool pressure begins |
| 02:15:47 | payment-service | ERROR: `HikariPool-1 - Connection is not available` | Pool exhausted |
| 02:16:02 | api-gateway | 504 returned to client for `/checkout` | User-facing failure begins |
| 02:58:44 | orders-db | Slow query resolved after manual `VACUUM ANALYZE` | DB pressure relieved |
| 03:01:19 | payment-service | Pool usage back to 4/20, errors stop | Recovery |

**Inflection point**: 02:14:03 — the slow query is the earliest abnormal event.

#### Failure Classification

- **Database issues — ROOT CAUSE**: `findPending()` query missing index on `status` column, seq-scanning 2.1M rows (evidence: slow query log, execution plan in DB logs).
- **Timeouts — CONTRIBUTING**: payment-service HTTP client timeout (30s) exceeded while waiting on DB connections; retries amplified pool pressure.
- **Downstream dependency issues — SYMPTOM**: api-gateway 504s are downstream of payment-service failures.
- **Code issues — CONTRIBUTING**: deploy `v2.14.3` introduced the unindexed query; no query timeout guard.

#### Root Cause Analysis

**Probable root cause**: Missing index on `orders.status` caused `OrderRepository.findPending()` to degrade from ~40ms to 8+s after deploy `v2.14.3`. Each call held a pooled connection ~200x longer, exhausting the 20-connection Hikari pool. With the pool empty, all checkout requests queued until the 30s client timeout → 504s.

**Causal chain**: unindexed query → slow DB calls → connection pool exhaustion → request queueing → client timeouts → 504s at gateway.

**Confidence**: **High** — slow query log, pool metrics, and deploy timing all align. Missing evidence: DB CPU metrics during window (not in ELK).

#### Impact Assessment

- **Affected**: `/checkout` endpoint, all users; ~340 failed requests, 47 min duration.
- **Latency**: p99 on `/checkout` spiked from 180ms to 30s (timeout ceiling).
- **Status**: Mitigated, **not resolved** — slow query still in code; will recur under load.

#### Recurrence

- **Signature**: `HikariPool-1 - Connection is not available` in `payment-service`
- **First seen**: 2026-11-24 02:15:47 UTC · **Last seen**: 2026-11-24 03:01:19 UTC
- **Occurrences**: 1,204 log lines across ~340 traces in the incident window; 0 in the prior 7 days
- **Trend**: **New** — no prior occurrences of this signature; correlated with deploy `v2.14.3` at 02:10 UTC
- **Prior incidents**: none with the same signature

#### Recommended Solutions

**Immediate mitigation** (done):
- Resized Hikari pool 20 → 40 and ran `VACUUM ANALYZE` — restored service.

**Short-term fix** (next sprint):
- Add index: `CREATE INDEX CONCURRENTLY idx_orders_status_created ON orders(status, created_at);`
- Add a 5s `statement_timeout` on the payment-service DB role so slow queries fail fast instead of holding connections.
- Reduce client timeout 30s → 10s and cap retries at 2 to prevent retry storms.

**Long-term prevention**:
- Alert on Hikari pool usage >80% and DB slow-query threshold (>1s).
- Add query regression tests for new repository methods in CI.
- Deploy `v2.14.4` with the index + timeout changes; verify `findPending()` returns to <100ms.

---

## Common Issues

- **Zero hits**: wrong field name, wrong index pattern, or ID outside the time window. Verify field via mapping API and widen `gte`.
- **Too many hits (sessionId)**: add filters — service, level, or a narrower time range around the incident.
- **Timeouts**: reduce `size`, tighten the time range, or query a specific index instead of a wildcard pattern.
- **Case sensitivity**: `term` queries are exact; use `match` for analyzed text fields, but prefer `keyword` subfields for IDs.

## Example Prompts

- "Find all logs for traceId `4bf92f3577b34da6a3ce929d0e0e4736` in the last 24 hours and tell me what failed."
- "Reconstruct the session for sessionId `sess_abc123` — what was the user doing before the 500 errors?"
