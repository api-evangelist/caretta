---
name: Caretta
description: Use when connecting AI agents to sales call data, automating call data delivery to external systems via webhooks, or integrating Zoom meeting links into calendar invitations. Reach for this skill when working with call transcripts, AI-generated notes, evaluated metrics, or managing todos from sales calls.
metadata:
    mintlify-proj: caretta
    version: "1.0"
---

# Caretta Skill

## Product summary

Caretta is realtime AI for sales calls. It provides three integration patterns: (1) **MCP server** at `https://gateway.caretta.app/mcp` for AI agents to query calls, transcripts, and todos with OAuth-based scoped access; (2) **Webhooks** to push call data (transcripts, notes, metrics) to your HTTPS endpoint as signed events; (3) **Zoom integration** to auto-create scheduled Zoom meetings and add join links to calendar invitations. All setup happens in Caretta's Settings panel (MCP, Webhooks, or Integrations). No API keys required—authentication is browser-based OAuth.

## When to use

- **MCP integration**: When an AI agent (Claude, Codex, etc.) needs to search calls, retrieve transcripts, list or manage todos, or answer questions about sales conversations. The agent queries Caretta data directly with scoped permissions.
- **Webhooks**: When you need to automatically move call data into your own systems (CRM, data warehouse, automation platform) after a call finishes processing. Use for reliable, signed delivery of transcripts, AI notes, and metrics.
- **Zoom integration**: When you want Caretta to create scheduled Zoom meetings and embed join links in calendar invitations sent from Caretta.

## Quick reference

### MCP Endpoint and Scopes

| Scope | Grants access to |
| --- | --- |
| `calls:read` | Call metadata, summaries, transcripts you can access |
| `todos:read` | Todos created from your calls |
| `todos:write` | Create and modify todos on accessible calls |

### Available MCP Tools

| Tool | Purpose | Scope |
| --- | --- | --- |
| `caretta_list_calls` | List recent calls (owned, attended, shared) | `calls:read` |
| `caretta_list_my_calls` | List only your calls with cursor pagination | `calls:read` |
| `caretta_search_transcripts` | Search transcripts and summaries | `calls:read` |
| `caretta_get_call` | Retrieve call with transcript and todos | `calls:read` + `todos:read` |
| `caretta_list_todos` | List todos, optionally filtered by call or state | `todos:read` |
| `caretta_create_todo` | Create a todo on a call | `todos:write` |
| `caretta_update_todo` | Update todo text, owner, due date, or state | `todos:write` |

### Webhook Events

| Event | Sent when | Key data |
| --- | --- | --- |
| `call.completed` | Call finishes processing | Call envelope, transcript |
| `call.notes_ready` | AI notes become available | Markdown notes, summary, next steps |
| `call.metrics_ready` | Metric evaluation finishes | Evaluated metrics or skip reason |
| `call.ready` | All selected components ready (bundled mode) | Transcript + notes/metrics |
| `webhook.test` | Admin sends test from Settings | Test message |

### Webhook Verification Headers

| Header | Purpose |
| --- | --- |
| `X-Caretta-Signature` | HMAC-SHA256 signature: `v1=` + hex digest |
| `X-Caretta-Timestamp` | Unix timestamp (seconds); reject if >5 min old |
| `X-Caretta-Delivery-Id` | Idempotent delivery ID (reused on retries) |
| `X-Caretta-Event-Id` | Stable event ID for deduplication |

## Decision guidance

### When to use MCP vs Webhooks

| Use MCP | Use Webhooks |
| --- | --- |
| Agent needs to query calls on demand | Data must flow automatically to external systems |
| Interactive questions about transcripts | One-way push of call data after processing |
| Agent manages todos | Reliable delivery with signature verification |
| Real-time agent access | Asynchronous processing of call events |

### Delivery mode: Per-event vs Bundled

| Per-event | Bundled (`call.ready`) |
| --- | --- |
| Lowest latency; each component sent immediately | Single payload after all components ready |
| Subscribe to `call.completed`, `call.notes_ready`, `call.metrics_ready` individually | Requires all selected components before delivery |
| Use `call.completed` for guaranteed receipt of every call | Notes are best-effort (~3–4% may not generate) |
| Use `call.metrics_ready` for evaluation results including skip reasons | Simpler downstream processing |

**Gotcha**: If bundled mode requires notes and notes fail to generate, the bundle is never sent. Add a per-event `call.completed` endpoint as a safety net if you must receive every call.

## Workflow

### Connect an AI agent via MCP

1. **Get the endpoint**: `https://gateway.caretta.app/mcp`
2. **Add to your client**: Paste the URL into your MCP client's settings (Claude Code, Codex, etc.).
3. **Authorize**: Your browser opens Caretta sign-in. Log in and review the client name.
4. **Choose scopes**: Select only the scopes the agent needs (`calls:read` for queries, add `todos:read`/`todos:write` if managing todos).
5. **Test**: Ask the agent "List my five most recent Caretta calls."
6. **Manage access**: In Caretta **Settings → Caretta MCP**, view all authorized clients and revoke any as needed.

### Set up webhook delivery

1. **Add endpoint**: In Caretta **Settings → Webhooks → Add endpoint**, enter your public HTTPS URL.
2. **Choose delivery mode**: Select per-event (individual events) or bundled (`call.ready`).
3. **Save secret**: Copy the signing secret immediately—it is shown once. Store in a secrets manager.
4. **Send test**: Click **Send test** to receive a `webhook.test` event.
5. **Verify signature**: In your handler, verify `X-Caretta-Signature` against the raw request body using HMAC-SHA256.
6. **Return 2xx**: Respond within 10 seconds. Caretta retries failed requests with backoff for ~2 hours.
7. **Deduplicate**: Use `X-Caretta-Event-Id` to detect and skip duplicate deliveries.

### Connect Zoom for calendar invitations

1. **Enable invitations**: In Caretta **Settings → Buttons**, turn on **Send Calendar Invites**.
2. **Select Zoom**: Under **Meeting link platform**, choose **Zoom**.
3. **Authorize Zoom**: If prompted, complete Zoom's OAuth flow (grants `meeting:write:meeting` only).
4. **Authorize Google Calendar**: Complete Google Calendar write access if prompted.
5. **Send invitation**: After a call, use Caretta's calendar action, enter title/time/attendees, send.
6. **Verify**: Caretta creates a scheduled Zoom meeting and embeds the join link in the calendar event.

## Common gotchas

- **MCP scope creep**: Existing authorizations cannot silently expand permissions. If an agent needs a new scope, revoke it in **Settings → Caretta MCP** and reconnect with the new scope selected.
- **Webhook signature verification**: Always verify against the **raw request body**, not parsed JSON. Reparsing changes whitespace/key order and breaks verification.
- **Webhook delivery is at-least-once**: Use `X-Caretta-Event-Id` to deduplicate. Event ordering is not guaranteed; correlate with `data.call.id`.
- **Notes are best-effort**: ~3–4% of calls never generate notes (e.g., desktop app closed before generation). Bundled mode requiring notes may never deliver. Use per-event `call.completed` as a fallback.
- **Eligible calls only**: Calls ≤60 seconds and deleted calls do not trigger webhook events.
- **Webhook timeout**: Return a 2xx response within 10 seconds. Perform slow processing asynchronously after responding.
- **Zoom admin approval**: Your Zoom administrator may need to approve Caretta before you can connect. Always start from **Settings → Integrations → Zoom**; do not use raw Zoom URLs.
- **MCP client support**: Your client must support remote HTTP servers and OAuth. Check client documentation.

## Verification checklist

Before submitting work with Caretta integrations:

- [ ] **MCP**: Test the agent can list calls and retrieve transcripts. Verify scopes are minimal (no unnecessary `todos:write`).
- [ ] **Webhooks**: Send a test event and confirm your endpoint returns 2xx within 10 seconds. Verify signature validation logic with the test event.
- [ ] **Webhooks**: Confirm you stored the signing secret securely and can rotate it without downtime.
- [ ] **Webhooks**: Check delivery mode matches your use case (per-event for guaranteed receipt, bundled for simplicity).
- [ ] **Zoom**: Confirm Zoom meeting appears in your Zoom account and join link is in the calendar event.
- [ ] **Zoom**: Verify Google Calendar write access is enabled if using calendar invitations.
- [ ] **All**: Confirm you can revoke/disconnect integrations from Caretta Settings without breaking production.

## Resources

- **Full documentation**: https://www.caretta.so/docs/llms.txt (comprehensive page-by-page navigation for agents)
- **MCP integration**: https://www.caretta.so/docs/caretta-mcp
- **Webhooks**: https://www.caretta.so/docs/webhooks
- **Zoom integration**: https://www.caretta.so/docs/zoom

---

> For additional documentation and navigation, see: https://www.caretta.so/docs/llms.txt