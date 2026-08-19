---
name: uncodie-icp-to-qualified-leads
description: Define an ideal customer profile in Makinari's Finder, size it before you spend, mine it into a segment, then create and qualify the resulting leads.
api: Makinari MCP Server
generated: '2026-08-13'
method: generated
source: https://docs.makinari.com/mcp-server/tools
grounding: >-
  Every tool name and parameter below is transcribed from the provider's own
  MCP tool documentation pages. Makinari publishes no OpenAPI, so operations
  are named by MCP tool + the REST path the tool page prints, not by
  operationId.
operations:
  - getFinderCategoryIds
  - analyzeICPTotalCount
  - createIcpMining
  - segments
  - leads
  - POST /api/agents/tools/getFinderCategoryIds
  - POST /api/agents/tools/analyzeICPTotalCount
  - POST /api/agents/tools/createIcpMining
  - POST /api/agents/tools/segments/create
  - POST /api/agents/tools/leads/create
  - POST /api/agents/tools/leads/qualify
---

# ICP to qualified leads

Makinari's prospecting flow has a strict order, and getting it wrong is the
most common and most expensive mistake: **ICP filters are numeric IDs, not
words**, and mining costs money per lead.

## Before you start

- Get an API key. Keys are created per site and per user:
  `POST /api/keys` with `{name, scopes:["read","write"], site_id, user_id}`.
  The full key is shown **once**. Default expiry is 90 days.
  (https://docs.makinari.com/first-steps/api-keys)
- Send it on every call as `Authorization: Bearer <key>`.
- API access is **not** included on the POC tier — it starts at Startup
  ($99/month). Confirm the account's plan before building on this.
- Mining is billed per lead ($0.50 per highly qualified lead, $0.01 per API
  call / deep research). Always size before you mine.

## Steps

1. **Resolve every filter word into an ID.** Call `getFinderCategoryIds` with
   `category` (one of `industries`, `organizations`, `organization_keywords`,
   `locations`, `person_skills`, `web_technologies`) and `q` (the search text).
   It returns `{id, text}` pairs. The tool page says this explicitly: use it
   **before** `analyzeICPTotalCount` or `createIcpMining`. Passing raw strings
   where IDs are expected silently produces the wrong audience.

2. **Size the ICP before spending.** Call `analyzeICPTotalCount` with the ID
   arrays: `person_industries`, `person_locations`, `person_skills`,
   `organization_industries`, `organization_locations`,
   `organization_keywords`, `organization_web_technologies` (number ID
   arrays), `organization_domains` (string array), plus `role_title`,
   `role_description` and `site_id`. All filters may be empty for a total
   count. Read the count back to the user before mining.

3. **Create the segment that will hold the results.** Call `segments` with
   `action: "create"` and a `name`; keep the returned `segment_id`.

4. **Mine.** Call `createIcpMining` with the **same** filter arrays, plus
   `segment_id` and `total_targets`. `total_targets` is the spend control —
   set it deliberately, do not default it.

5. **Materialise leads.** Call `leads` with `action: "create"` (requires
   `name` and `email`) for anything you add by hand, or `action: "list"` with
   `segment_id`, `limit` and `offset` to page the mined results. Pagination is
   offset-based; defaults are `limit: 50`, `offset: 0`.

6. **Qualify.** Call `leads` with `action: "qualify"` and a `status`.
   Documented statuses include `cold` and `not-qualified` (added in release
   v1.3.0). Set `lead_score`, `interest_level` and `assignee_id` on the same
   call rather than issuing a second update.

## Conventions you must respect

- **No idempotency.** Makinari documents no idempotency key on any endpoint.
  A retried `create` will create a second record. Track the returned id
  yourself and prefer `list` + match over blind retry.
- **Errors are not RFC 9457.** The documented shape is `{"message": "..."}`;
  the live platform also returns
  `{"success":false,"error":{"code":"...","message":"..."}}`. Handle both.
  See `errors/uncodie-problem-types.yml`.
- **Rate limits are undocumented.** A `429 Rate limit exceeded` exists, but no
  limit, window or `Retry-After`/`RateLimit-*` header is published. Back off
  exponentially on 429 and do not assume a budget.
  See `rate-limits/uncodie-rate-limits.yml`.
- **`site_id` scopes everything.** It is required on nearly every tool and is
  the tenancy boundary; a key issued for one site cannot read another.

## Where this runs

- Hosted MCP: `POST https://backend.makinari.com/api/mcp` (JSON-RPC 2.0).
- REST equivalents are printed on each tool's docs page under
  `/api/agents/tools/...` on `https://backend.makinari.com`.
- The documented base `https://api.makinari.io/v1` does **not** resolve in DNS.
