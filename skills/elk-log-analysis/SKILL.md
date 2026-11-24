---
name: elk-log-analysis
description: Analyze ELK (Elasticsearch/Kibana) logs using the ELK MCP server, correlating events by traceId or sessionId and producing a detailed root cause analysis (RCA) summary covering errors, downstream issues, timeouts, network/database latencies, code issues, and system errors with recommended solutions. Use when investigating errors, tracing requests across services, or debugging production issues in Kibana.
---

# ELK Log Analysis Skill

Analyze logs stored in Elasticsearch via the **ELK MCP server** by correlating events on `traceId` or `sessionId`.

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

## Modes

The skill supports three analysis modes. Infer the mode from the user's request, or ask if ambiguous.

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Quick triage** | "what failed", "quick look", "is it still happening" | Steps 1–5 only. Return error summary + timeline, no full RCA. Fast, small `size`. |
| **Standard analysis** | "analyze", "investigate", "why did X fail" | Steps 1–7. Full RCA summary with classification and recommendations. |
| **Deep dive** | "deep dive", "full RCA", "postmortem" | Standard + wider time window, cross-reference related traceIds/sessionIds found in the data, check for recurrence patterns (same error across other IDs), and compare against baseline error rates. |

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
