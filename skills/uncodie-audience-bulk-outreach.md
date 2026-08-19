---
name: uncodie-audience-bulk-outreach
description: Build a reusable Makinari audience from lead filters and send a merged email or WhatsApp message to every lead in it, respecting the WhatsApp 24-hour template rule.
api: Makinari MCP Server
generated: '2026-08-13'
method: generated
source: https://docs.makinari.com/mcp-server/tools
grounding: >-
  Tool names, actions, parameters and the 24-hour template rule are
  transcribed from the provider's MCP tool documentation pages. No OpenAPI is
  published, so there are no operationIds.
operations:
  - audience
  - sendBulkMessages
  - sendEmail
  - sendWhatsApp
  - whatsappTemplate
  - configureEmail
  - configureWhatsApp
  - POST /api/agents/tools/sendEmail
  - POST /api/agents/tools/sendWhatsApp
  - POST /api/agents/tools/whatsapp-templates
  - POST /api/agents/tools/configureEmail
  - POST /api/agents/tools/configureWhatsApp
---

# Audience-based bulk outreach

## Before you start

- Channel accounts are per plan. The POC tier is "bring your own Twilio and
  email"; a Makinari-provided email account, WhatsApp account and phone number
  start at the Startup tier. Check `configureEmail`
  (`action: get_config`-style domain/inbox setup) and `configureWhatsApp`
  (`get_config`) before sending anything.
- Authenticate with `Authorization: Bearer <api-key>`.

## Steps

1. **Build the audience.** Call `audience` with `action: "create"` and the
   lead filters you want frozen: `segment_id`, `campaign_id`, `assignee_id`,
   `status`, `origin`, `search`. The tool stores the matched leads as a named,
   reusable audience and pages them (default 50 leads per page). Keep the
   returned `audience_id`.

2. **Inspect before sending.** Call `audience` with `action: "list"` (or the
   read action) plus `page` / `page_size` and read the size back to the user.
   Bulk sends are irreversible and metered.

3. **Send.** Call `sendBulkMessages` with `audience_id`, `channel`
   (`whatsapp` or `email`), `message`, and — for email — `subject` and
   optionally `from` / `audience_email_mode`. Use `{{lead.*}}` merge tokens in
   `message` and `subject`: `{{lead.name}}`, `{{lead.first_name}}`,
   `{{lead.email}}`, `{{lead.phone}}`, `{{lead.position}}`. Set
   `placeholder_policy` deliberately — it decides what happens when a lead is
   missing a merge field.
   Leads missing the channel's required contact field (phone for WhatsApp,
   email for email) are skipped; the tool tracks per-lead delivery status.

4. **One-to-one follow-up.** For a single recipient use `sendEmail`
   (requires `email`, `subject`, `message`; pass `lead_id` to enable merge
   fields) or `sendWhatsApp` (requires `phone_number`, `message`).

5. **Handle the WhatsApp 24-hour window.** WhatsApp will not deliver a free-form
   message more than 24 hours after the contact's last message. If
   `sendWhatsApp` returns `template_required: true`, call `whatsappTemplate`
   with `action: "create_template"` (`phone_number`, `message`); if it returns
   a `template_id`, call it again with `action: "send_template"`
   (`template_id`, `phone_number`, `original_message`). Do not retry the
   free-form send — it will keep failing.

## Conventions you must respect

- **No idempotency key exists.** A retried `sendBulkMessages` re-sends to the
  whole audience. Record the call and its result; never retry blindly.
- **No published rate limits.** 429 exists with no headers and no numbers.
- Credits are consumed per message-generating action; tiers include 20 / 100 /
  500 credits per month with metered overage (see
  `plans/uncodie-plans-pricing.yml`).
- Errors: `{"message": "..."}` or
  `{"success":false,"error":{"code":"...","message":"..."}}` — not RFC 9457.
