---
name: uncodie-keys-webhooks-workflows
description: Provision a scoped Makinari API key, register a webhook so agents react instead of polling, and trigger a Temporal-backed workflow.
api: Makinari REST API
generated: '2026-08-13'
method: generated
source: https://docs.makinari.com/first-steps/api-keys
grounding: >-
  Endpoints and parameters transcribed from the provider's documentation:
  first-steps/api-keys, mcp-server/tools/webhooks, mcp-server/tools/workflows
  and the workflows reference. No OpenAPI is published.
operations:
  - POST /api/keys
  - GET /api/keys
  - DELETE /api/keys
  - POST /api/agents/tools/webhooks
  - POST /api/workflow/webhook
  - POST /api/workflow/leadGeneration
  - POST /api/workflow/enrichLead
  - POST /api/workflow/buildCampaigns
  - POST /api/workflow/deepResearch
  - webhooks
  - workflows
---

# Keys, webhooks and workflows

This is the plumbing skill: everything else assumes a key exists and that
long-running work is triggered rather than awaited.

## 1. Provision a key

`POST /api/keys`

```json
{
  "name": "My API Key",
  "scopes": ["read", "write"],
  "site_id": "your-site-uuid",
  "user_id": "your-user-uuid",
  "expirationDays": 90,
  "prefix": "my-key",
  "metadata": { "environment": "production" }
}
```

- `name`, `scopes`, `site_id`, `user_id` are required.
- `expirationDays` defaults to **90** — keys expire. Schedule rotation.
- The full key is returned **once**. Store it immediately.
- List: `GET /api/keys?site_id=...&user_id=...`
- Revoke: `DELETE /api/keys?id=...&site_id=...&user_id=...`

Send it as `Authorization: Bearer <key>` on every request. The Secure Tokens
endpoints are the exception — they take `x-api-key` instead
(`POST /api/secure-tokens/{id}/decrypt`). The MCP server accepts either
`Authorization: Bearer` or `X-API-Key`, using the same keys as the REST API;
there is no MCP-specific credential and no OAuth flow
(`/.well-known/oauth-authorization-server` returns 404).

## 2. Register a webhook

Call the `webhooks` tool (`POST /api/agents/tools/webhooks`):

- `action: "list"` — all webhooks for the site.
- `action: "create"` — requires `url` and `events` (array of strings).

Documented example events are `lead.created` and `conversation.started`. **A
complete event catalogue is not published**, and no signature-verification
scheme is documented — verify the payload against your own state before acting
on it, and treat the endpoint as unauthenticated input.

Makinari also runs an inbound workflow webhook, `POST /api/workflow/webhook`,
which receives DB change events, finds active subscriptions by site and table,
and dispatches a Temporal workflow asynchronously.

## 3. Trigger a workflow

Either call the `workflows` MCP tool with `action` and a `payload` object, or
POST directly to the named route. Documented workflows include:

`agentMessage`, `analyzeSite`, `assignLeads`, `buildCampaigns`, `buildContent`,
`buildSegments`, `buildSegmentsICP`, `customerSupport`, `dailyStandUp`,
`deepResearch`, `enrichLead`, `humanIntervention`,
`idealClientProfileMining`, `keyAccountGeneration`, `leadFollowUp`,
`leadFollowUpManagement`, `leadGeneration`, `leadInvalidation`,
`leadQualificationManagement`, `leadResearch`, `promptRobot`, `startRobot`,
`stopRobot`, `syncEmails` — each at `POST /api/workflow/<name>`.

Workflows execute on Temporal and return asynchronously. Several agent
endpoints accept a `webhook` parameter (for example
`POST /api/agents/sales/leadGeneration` takes `siteId`, `userId`, `maxLeads`,
`priority`, `company`, `webhook`) — pass your callback URL there rather than
polling.

## 4. Check health before you trust a result

`GET https://docs.makinari.com/api/status` returns machine-readable per-system
health with 24h/7d/30d SLA for 21 systems. This is worth calling: on both
2026-07-21 and 2026-08-13 the Agents API and every AI provider system reported
**0% uptime over 24h and 30d** while the core REST surfaces reported 100%. An
agent that checks status first can degrade gracefully instead of timing out.

## Conventions

- No idempotency support. Retried triggers start duplicate workflows.
- 429 exists; no limits, no `Retry-After`, no `RateLimit-*` headers.
- Pagination is `limit`/`offset` (defaults 50 / 0).
- No deprecation policy and no `Sunset` header support.
