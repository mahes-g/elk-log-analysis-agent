# Debugging Production with AI: An ELK Log Analysis Skill Powered by MCP

*Draft for Medium — suggested tags: #elasticsearch #kibana #mcp #ai #devops #observability*

---

## The Problem

It's 2 AM. A customer reports that checkout is failing. You have an error report with a `traceId` and a `sessionId` — and a Kibana instance containing millions of log lines across a dozen microservices.

The manual workflow is painful:

1. Paste the trace ID into Kibana's search bar.
2. Hope the field name is right (`traceId`? `trace_id`? `@trace.id`?).
3. Scroll through hundreds of results across services.
4. Piece together the timeline by hand.
5. Repeat for the session ID to understand what the user did before the failure.

What if an AI agent could do all of this — query Elasticsearch directly, correlate events, and hand you a root-cause summary?

## Enter MCP: Giving AI Agents Access to Your Logs

The **Model Context Protocol (MCP)** is an open standard for connecting AI assistants to external tools and data sources. Elastic ships an official MCP server for Elasticsearch, which means agents like Claude can query your cluster directly — run searches, fetch mappings, aggregate errors — using natural language.

But "the agent can query Elasticsearch" isn't the same as "the agent is good at debugging." Left to its own devices, an agent will:

- Run unbounded queries that time out on large indices.
- Guess field names and silently get zero results.
- Deep-page with `from`/`size` until it hits `index.max_result_window`.
- Dump raw logs instead of a structured analysis.

This is where **skills** come in.

## What Is a Skill?

A skill is a packaged set of instructions (a `SKILL.md` file) that teaches an agent *how* to perform a specific task well. Instead of hoping the model figures out ELK best practices on every request, you encode them once:

- Which identifier type you're dealing with (trace vs session).
- Which fields to search.
- How to bound queries by time.
- How to paginate safely.
- What a useful output looks like.

## The ELK Log Analysis Skill

I built a skill that analyzes ELK logs via the ELK MCP server, correlated by `traceId` or `sessionId`. Here's what it encodes:

### 1. Identifier disambiguation

A `traceId` represents one request flowing through distributed services — a tight time window, a call-path analysis. A `sessionId` represents a user's entire session — potentially hours of activity across many requests. The skill asks which you're investigating and adjusts the query window and analysis depth accordingly.

### 2. Field discovery before querying

Different stacks instrument tracing differently: `traceId`, `trace_id`, `@trace.id`, `transaction.id`. A `term` query on the wrong field returns zero hits with no error — the worst kind of failure. The skill instructs the agent to verify field names via the mapping API first.

### 3. Bounded, efficient queries

Every query includes a `range` filter on `@timestamp`. For traces: ±24h or tighter. For sessions: up to 7 days but with service/level filters to control volume. No unbounded wildcard searches.

### 4. Safe pagination

`search_after` with sorting on `@timestamp` + `_shard_doc` (or a tiebreaker field) instead of deep `from` paging, which degrades badly past ~10k results.

### 5. Structured analysis output

The end product isn't a log dump — it's:

- **Timeline** — first/last seen, duration, key events in order.
- **Service map** — which services the request touched and in what order.
- **Error summary** — exceptions with relevant stack frames.
- **Root cause hypothesis** — the first ERROR in trace order is usually where to look.

## Example: What the Agent Actually Does

You say: *"Find all logs for traceId `4bf92f3577b34da6a3ce929d0e0e4736` and tell me what failed."*

Under the hood, the agent:

1. Confirms `traceId` is the right field (mapping check).
2. Runs a `bool` query: `term` on `traceId` + `range` on `@timestamp` for the last 24h, sorted ascending, size 200.
3. Gets 47 hits across 4 services.
4. Filters to ERROR level → finds a `NullPointerException` in `payment-service` at `CheckoutController.java:142`.
5. Traces backward: the preceding WARN shows a retry loop in `inventory-service` returning stale stock counts.
6. Reports: *"Payment failed because inventory-service returned a stale reservation. Root cause appears to be a cache TTL misconfiguration. Recommend checking `inventory.cache.ttl` in config."*

That entire investigation — what used to take 15–30 minutes of Kibana wrangling — takes seconds.

## Setting It Up

The full skill is open source. Setup is three steps:

1. **Configure the ELK MCP server** — point it at your cluster with an API key (see `mcp-config.example.json` in the repo).
2. **Install the skill** — drop `SKILL.md` into your agent's skills directory.
3. **Verify field names** — one mapping query to confirm your trace/session fields match the skill's defaults.

## Why Skills + MCP Is a Powerful Combo

MCP gives agents *access* to your infrastructure. Skills give them *judgment* about how to use that access. Together, they turn a general-purpose chatbot into a domain expert that follows your team's debugging playbook — consistently, every time, at 2 AM when you're not available to supervise.

The same pattern applies beyond ELK: database query skills, CI/CD debugging skills, cloud infrastructure skills. Anywhere an agent needs to both access a system and apply expert methodology, a skill is the missing piece.

## Try It

The skill and full setup instructions are on GitHub: **[link to repo]**

If you build skills for other tools — Datadog, Splunk, CloudWatch, your internal APIs — I'd love to hear how you structure them. What domain expertise would you encode first?

---

*About the author: Srikrishna Mahesh — platform engineer working on observability and AI-assisted operations.*
