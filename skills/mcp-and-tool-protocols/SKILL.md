---
name: mcp-and-tool-protocols
description: Engineering Model Context Protocol servers and clients, and choosing between MCP, plain function-calling, and code-that-calls-APIs. Load when designing/reviewing an MCP server, picking transports or auth for tool integrations, hardening against tool-poisoning/confused-deputy attacks, or reasoning about the agent protocol landscape (MCP vs A2A).
---

# MCP and Tool Protocols

## Version note (verified July 2026) — read this first

Your training-era picture of MCP is one to two revisions stale. Current stable spec revision: **2025-11-25** (not 2025-06-18). A **2026-07-28 release candidate** was locked in May 2026 and finalizes July 28, 2026 — the largest revision since launch (**stateless core**). Target 2025-11-25 semantics today, but avoid patterns the RC deprecates: **roots, sampling, and protocol-level logging all enter a 12+ month Deprecated window**. Anything below marked "RC" is release-candidate behavior; confirm final status after July 2026.

## Core mental model

1. **You are writing an interface for a model to read.** Tools = model-invoked actions; resources = application-attached context (URI-addressed); prompts = user-invoked templates. Descriptions, schemas, and error strings ARE the prompt; error messages must tell the model what to do differently.
2. **Installing a server = granting its author influence over your context and granting the model the server's privileges.** The three canonical attack classes: tool poisoning (malicious metadata), injection via tool results, confused deputy (server exercises privileges the requesting user shouldn't have).
3. **Statefulness is being engineered out of the protocol — design for it now.** History: HTTP+SSE (2024-11-05) → Streamable HTTP (2025-03-26) → the 2026-07-28 RC **removes the `initialize` handshake and `Mcp-Session-Id` entirely**; client info/capabilities travel in `_meta` on every request, so any request can hit any instance behind a plain load balancer. The pre-RC reflex — sticky sessions or a Redis session store keyed on `Mcp-Session-Id` — is building on a header that is going away. Keep state in arguments, a database, or resource URIs.
4. **Two-layer landscape:** MCP = agent-to-tool; **A2A** = agent-to-agent (Linux Foundation since June 2025; **IBM's ACP merged into it August 2025**; 150+ member orgs, SDKs in Python/JS/Java/Go/.NET, production deployments announced April 2026). If the remote thing executes a bounded operation and returns, wrap it in MCP even if it's internally agentic; reach for A2A only for genuine peer semantics (independent principals, long-lived tasks, cross-org delegation). Most "we need A2A" designs are one orchestrator calling subagents in-process — which needs neither protocol. The bodies have committed to interop; don't bet on either absorbing the other this year.

## Decision frameworks

### MCP vs plain function-calling vs code-executes-API

1. One consumer you control → function-calling in your harness; MCP adds a process boundary for zero benefit. Multiple hosts/third parties → MCP.
2. Code knows the sequence → write code that calls the API; tools are for steps the *model* chooses at runtime.
3. Bulk data (10k rows) → don't return it through a tool call; give the agent code-execution against the API. Tool results transit the context window; code execution doesn't.
4. Migrate prototype→MCP when the second consumer appears, not preemptively.

### Primitives and granularity (compressed — the parts models already know)

- Who decides usage? Model → tool; application → resource; user → prompt. Pragmatic override: tools enjoy universal host support, resources/prompts don't — verify your target hosts before shipping a resource, fall back to a read-style tool documenting why.
- Fewer, task-shaped tools (a 30-endpoint API → ~5 intent-level tools). Merge overlapping lookups; split only where blast radius differs. Description = what/when/when-NOT/returns/example.
- Annotate `readOnlyHint`/`destructiveHint`/`idempotentHint` for host confirmation UX — but treat *other* servers' annotations as unverified claims, not security.

### Transport

- **stdio** for local: zero network surface, user's OS identity. stderr-only logging — stdout writes corrupt the JSON-RPC stream (the #1 rookie bug).
- **Streamable HTTP** for remote: single `/mcp` endpoint, POST + optional SSE stream. RC adds required **`Mcp-Method`/`Mcp-Name` headers** so gateways can route/rate-limit without body inspection, plus `MCP-Protocol-Version: 2026-07-28`.
- The old two-endpoint HTTP+SSE transport is legacy; never build new servers on it.

### Auth (Streamable HTTP only) — where the baseline is stale

Settled model as of 2025-11-25: MCP server = OAuth 2.1 **resource server**. The flow: RFC 9728 Protected Resource Metadata (`/.well-known/oauth-protected-resource`; `WWW-Authenticate` now optional with well-known fallback) → AS discovery via RFC 8414 **or OpenID Connect Discovery (OIDC support added 2025-11-25)** → PKCE mandatory → **RFC 8707 resource indicators mandatory** (audience-bound tokens).

**The correction most models need:** client registration is no longer Dynamic Client Registration by default. **Client ID Metadata Documents (CIMD)** — a URL identifying the client — is the recommended mechanism as of 2025-11-25, displacing DCR for most cases (enterprises can still pre-register). If you recommend RFC 7591 DCR as "the spec's baseline," you're one revision behind.

Never pass through tokens (spec-explicit MUST NOT): exchange via RFC 8693 or hold your own credentials, scoped to the *requesting user*. Don't hand-roll PRM/PKCE — use the official SDK auth middleware; hand-rolled flows are where audits find holes.

(RC additions: mandatory `iss` validation per RFC 9207, `application_type` at registration, credentials bound to issuing server's `issuer`, documented OIDC refresh-token flow.)

### Long-running operations — the baseline is wrong here

The common model answer is "MCP has no first-class async primitive; return a job ID and poll." **Outdated:** the 2025-11-25 spec ships a **Tasks extension** — a task handle plus `tasks/get` polling (RC redesign: server answers `tools/call` directly with a task handle; `tasks/list` removed). Use Tasks where host support exists; the job-id + `check_job(id)` tool remains the portable fallback. Never hold an SSE stream open for a 10-minute job.

## How an expert thinks through it

*Scenario: wrap an internal ticketing REST API (~30 endpoints) for support agents' Claude.*

Multiple hosts → MCP, remote, Streamable HTTP (rejected: stdio with per-user API keys — no central authz, key sprawl). Tool surface: five intent tools (search/get/update/comment/escalate), not 30 wrappers and not a generic `query(endpoint, params)` passthrough — the model would have to learn your REST API from nothing and permissions couldn't be scoped per-operation. `search_tickets` returns id|status|title|one-line summary, max 20 — 30 full tickets is 50k tokens of pollution; drill-down via `get_ticket(id)`. Auth: OAuth against the IdP, tokens exchanged per requesting user — a server-level service account is the confused deputy (any injected instruction in a ticket body could read every ticket in the org). Ticket bodies are attacker-controlled: separate read/write scopes, host confirmation on writes, untrusted content wrapped in a marked block — documented as reducing, not eliminating, risk. Stop at five tools until a transcript shows the model needing more; tool surface is API surface, supported and secured forever.

## Testing MCP servers

1. **Handler tests:** valid/boundary/malformed args; assert on the *text* of error returns — the error string is product surface.
2. **Protocol tests:** real client session (`mcp` SDK client or `npx @modelcontextprotocol/inspector`). Catches serialization mismatches, stdout pollution, oversized results, auth challenge flows.
3. **Transcript tests — the layer that predicts production and that most teams skip:** wire into a real host, run ~10 fixed tasks *including 3 designed to fail* (missing entity, permission denied, ambiguous request); assert expected tool + acceptable args, and that failure tasks end in graceful reporting, not retry loops. **Run the suite with a mid-tier model:** if the cheap model navigates your descriptions, the frontier model certainly will — you've bought headroom. Every production misuse becomes a transcript test before you fix it.

## Failure modes and pitfalls

- **stdout logging in stdio servers** — stderr only. (RC formalizes: protocol-level logging deprecated in favor of stderr/OpenTelemetry.)
- **Raw API responses** — shape results for the model's next decision; add a drill-down tool.
- **Stack-trace errors** — return model-actionable text ("Ticket PROJ-12 has no assignee; use assign_ticket first"). Most "model retries the same broken call" reports trace here; highest-leverage fix available.
- **Token passthrough / shared service credentials** — the canonical confused deputy.
- **Trusting third-party tool metadata.** MCPTox-style research (2025–26) showed most agents follow malicious instructions embedded in descriptions of *unrelated* tools and most clients do no validation. Pin/hash tool definitions and alert on change (rug-pull detection); review descriptions at install like dependencies; prefer registries/allowlists.
- **Building on sampling or roots.** The baseline advice is "optional, poorly supported — degrade gracefully." Stronger as of 2026: both are **Deprecated-track in the 2026-07-28 RC**. New code should use direct LLM API calls server-side and explicit tool parameters instead — not graceful degradation around primitives being removed.
- **Stateful `Mcp-Session-Id` assumptions** — breaks behind round-robin LBs today; header removed in the RC.
- **Skipping `outputSchema`/`structuredContent`** (2025-06-18+) — downstream code re-parses prose. RC upgrades to full JSON Schema 2020-12 (`oneOf`/conditionals).
- **Held connections for long work** — use the Tasks extension / job-id polling (see above).

## Worked micro-example

A well-shaped tool (Python, official `mcp` SDK, FastMCP style):

```python
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("tickets")

@mcp.tool(
    annotations={"readOnlyHint": True},
)
async def search_tickets(
    query: str,
    status: Literal["open", "pending", "closed", "any"] = "open",
    limit: int = 10,
) -> str:
    """Search support tickets by keyword.

    Use this to FIND tickets; use get_ticket(id) to read one in full.
    Do not use for listing a user's own tickets (use my_tickets).
    Returns up to `limit` matches as: id | status | title | 1-line summary.
    Example: search_tickets(query="login 500 error", status="open")
    """
    rows = await api.search(query, status=None if status == "any" else status, limit=min(limit, 25))
    if not rows:
        return f"No {status} tickets match {query!r}. Try broader keywords or status='any'."
    return "\n".join(f"{r.id} | {r.status} | {r.title} | {r.summary}" for r in rows)
```

Load-bearing parts: enum-constrained params (bad calls fail at validation with a message), when-NOT-to-use, capped result size, empty-result message proposing the next action.

**Spec-conformant auth discovery flow:**

```text
1. Client → POST https://mcp.example.com/mcp          (no token)
2. Server → 401 + WWW-Authenticate: Bearer
            resource_metadata="https://mcp.example.com/.well-known/oauth-protected-resource"
3. Client → GET that metadata → { "authorization_servers": ["https://auth.example.com"] }
4. Client → GET /.well-known/oauth-authorization-server   (RFC 8414)
            (or /.well-known/openid-configuration — OIDC discovery, since 2025-11-25)
5. Authorization-code + PKCE, with
            resource=https://mcp.example.com          (RFC 8707 — audience-bound)
            client identified by CIMD URL or pre-registration   (NOT default DCR)
6. Client → POST /mcp with Bearer <token scoped to THIS server>
7. Server validates audience + scope; calls downstream with ITS OWN exchanged
   credential for THIS user — never the inbound bearer token.
```

Skip step 3 (hardcoded AS), step 5's `resource` param, or break rule 7, and it works in demos and fails security review.

## Verification and self-check

- Unit-test handlers, then transcript-test with a real host and a mid-tier model, including tasks that should fail.
- Fault-injection: malformed args, empty results, downstream 500s, oversized responses — every path returns model-actionable text.
- Security pass in frequency order: token passthrough/shared credentials? per-user scoping enforced server-side? untrusted result content marked as data? destructive tools annotated + gated? descriptions clean?
- Ship when a mid-tier model completes the task set from descriptions alone and every injected fault yields a recoverable transcript. More tools past that are liabilities.

## Delta notes (vs Opus 4.8 baseline, audited 2026-07)
- Probed 12 claims: 5 baseline (cut/compressed), 5 partial (sharpened), 2 delta (expanded).
- Biggest baseline gaps: believes stable spec is 2025-06-18 and knows nothing of the 2026-07-28 stateless-core RC (roots/sampling/protocol-logging deprecation, session-ID removal); claims "no first-class async primitive — poll via job-id tool" (2025-11-25 Tasks extension exists); recommends RFC 7591 DCR for client registration (CIMD displaced it in 2025-11-25).
- Solid baseline (kept only as compressed anchors): token-passthrough/confused-deputy, stdout-corrupts-stdio, task-shaped tool granularity, tool-poisoning attack taxonomy, primitive mapping + host-support override.
