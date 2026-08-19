---
name: uncodie-quote-to-cash
description: Run Makinari's commerce flow end to end — catalog item, quotation with line items, checkout order, Stripe payment link, sales order and entitlement. This surface is MCP-only; there is no REST equivalent.
api: Makinari MCP Server
generated: '2026-08-13'
method: generated
source: https://docs.makinari.com/mcp-server/tools
grounding: >-
  Actions and parameters transcribed from the provider's MCP tool pages for
  catalog_commerce, quotations, quotation_items, checkout, salesOrder, sales
  and entitlements. No REST path is published for any of them — see
  mcp/uncodie-tool-crosswalk.yml.
operations:
  - catalog_commerce
  - quotations
  - quotation_items
  - checkout
  - salesOrder
  - sales
  - entitlements
  - subscription_plan_items
---

# Quote to cash

**Read this first.** Makinari's entire commerce cluster is reachable **only**
through the MCP server. `catalog_commerce`, `quotations`, `quotation_items`,
`checkout`, `salesOrder`, `entitlements`, `subscription_plan_items`,
`purchases`, `purchase_items`, `reservations` and `reservation_schedules`
publish no REST route on any documentation page. If the client cannot speak
MCP JSON-RPC to `https://backend.makinari.com/api/mcp`, this flow is not
available to it.

## Steps

1. **Put something sellable in the catalog.** Call `catalog_commerce` with
   `resource: "item"`, `action: "create"`, `name`, and — this is the part that
   is easy to miss — `target_sale_price` **and** `is_purchasable: true`. An
   item without those is not sellable. Other fields: `sku`, `cost`,
   `lowest_sale_price`, `currency`, `kind`, `digital_subtype`, `is_recurring`,
   `is_reservation`, `category_id`, `image_url`, `status`.
   Use `resource` = modifier group / modifier for product options
   (`modifier_group_id`, `catalog_item_id`, `min_select`, `max_select`).

2. **Open a quotation.** Call `quotations` with `action: "create"` and
   `lead_id` (required). Optionally attach it to a `deal_id`, name a
   `buyer_user_id`, apply a `price_list_id`, and set `valid_until`, `currency`
   and `notes`.

3. **Add lines.** Call `quotation_items` with `action: "create"`,
   `quotation_id`, `catalog_item_id`, `quantity`, `unit_price`. Creating,
   updating or deleting a line **automatically recalculates the quotation
   totals** — do not compute totals yourself and do not write them back.

4. **Move the quotation through its states.** `quotations` with
   `action: "update"` and `status` ∈ `draft`, `sent`, `rejected`, `expired`.

5. **Create the order.** Call `checkout` with `action: "create_order"`,
   `site_id`, `lines`, and the buyer identity (`lead_id`, `buyer_user_id` or
   `customer_email`). This creates a **pending** `sale_order`.

6. **Collect payment.** Call `checkout` with `action: "create_payment_link"`
   and the `order_id`. It returns a Stripe Checkout URL — give that URL to the
   buyer. Poll `checkout` with `action: "get_order"` and `order_id` to see the
   state change; do not assume payment from the link being created.

7. **Record the sale.** Use `salesOrder` (`action: "create"`, `customer_id`,
   `product_ids`, `total_amount`, `payment_method`, plus optional `discount`,
   `tax`, `shipping_address`, `delivery_date`) and `sales` for the sale record.

8. **Grant access, carefully.** For digital goods, read entitlements with
   `entitlements` `action: "list"` / `"get"` by `buyer_user_id` or `status`.
   The tool page is explicit: **do not invent grants here** — use `update`
   only to change `status` (e.g. to `revoked` or `used`) or `expires_at`.
   Recurring bundles are mapped with `subscription_plan_items`
   (`plan_catalog_item_id` → `digital_catalog_item_id`).

## Conventions you must respect

- **No idempotency key.** This is a money flow with no idempotency support
  anywhere in the documentation. A retried `create_order` or `salesOrder
  create` creates a second order. Always `get_order` before retrying, and
  carry your own client-side dedupe key in `notes`/`metadata`.
- **No published rate limits and no `Retry-After`.** Back off on 429.
- Errors are `{"message": "..."}` or
  `{"success":false,"error":{"code":"...","message":"..."}}`. Neither is
  `application/problem+json`.
- `site_id` is the tenancy root; `owner_site_id` distinguishes the selling
  site in marketplace flows.
