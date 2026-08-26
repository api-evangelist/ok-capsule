---
name: ok-capsule-operate-catalog-via-mcp
description: >-
  Operate an OK Capsule brand through its MCP server — browse the catalog, design a pack, manage
  consumers and orders — with the scope and confirmation discipline the destructive tools require.
generated: '2026-08-26'
method: generated
source: >-
  https://okcapsule.com/mcp/developers,
  https://storefront.okcapsule.app/.well-known/oauth-authorization-server,
  https://storefront.okcapsule.app/.well-known/oauth-protected-resource, mcp/ok-capsule-mcp.yml
api: OK Capsule MCP Server
base_url: https://storefront.okcapsule.app/mcp
operations:
  - okc_authenticate
  - okc_list_brands
  - okc_list_products
  - okc_get_catalog
  - okc_get_product_intelligence
  - okc_validate_recommendation
  - okc_render_pack_builder
  - okc_pack_builder_url
  - okc_list_consumers
  - okc_upsert_consumer
  - okc_create_order
  - okc_get_order
  - okc_cancel_order
  - okc_confirm_pending_action
  - okc_list_fulfillments
  - okc_get_shipping_label
---

# Operate an OK Capsule brand through MCP

The MCP server is the **supported** contract. OK Capsule says so directly: "The MCP tools are the
supported contract. The underlying REST endpoints are reference-only and can change without notice."
If you are choosing between the two, choose this one.

Endpoint: `https://storefront.okcapsule.app/mcp` — Streamable HTTP JSON-RPC, `Authorization: Bearer`.

## Connect

Any conformant MCP host does the whole handshake for you. An unauthenticated request returns 401 with
a `WWW-Authenticate` challenge carrying `resource_metadata`, and the client follows it into dynamic
registration and browser sign-in.

- Claude Code: `claude mcp add --transport http okcapsule https://storefront.okcapsule.app/mcp`
- Claude / ChatGPT / Gemini / Perplexity: add it as a custom connector with that URL.

Sign-in is a **staff email one-time code**. There are no API keys, no passwords, and no
`client_credentials` grant — a human must sign in, so this cannot run fully unattended. The access
token is a 1-hour RS256 JWT; the refresh token rotates and lasts 30 days. After 30 days a human signs
in again. Every token is bound to exactly one workspace and nothing crosses workspaces.

## Discover the tools at runtime

**Call `tools/list`. Do not hardcode schemas.** The provider states the surface grows and argument
shapes can be extended, and the scope list "is not final and will grow". The authoritative scope list
is `scopes_supported` in
`https://storefront.okcapsule.app/.well-known/oauth-authorization-server` — read it, do not assume it.

## Know which scopes you were actually granted

Read access is the default grant. Everything consequential is opt-in at the consent screen:

| Granted by default | Opt-in only |
|---|---|
| `catalog:read`, `recommendations:read`, `orders:read`, `consumers:read`, `consumers:write`, `fulfillments:read`, `meta:read`, `documents:write` | `orders:write`, `orders:cancel`, `consumers:delete` |

If `okc_create_order` fails, check whether `orders:write` was actually consented before assuming a bug.

## Design a pack

1. `okc_list_brands` — the brand's product lines.
2. `okc_get_catalog` or `okc_list_products` — what is orderable.
3. `okc_get_product_intelligence` — ingredient and product intelligence per product. Use it to explain
   each choice and to flag combinations that should not be taken together. This has **no REST
   equivalent**; it exists only here.
4. `okc_validate_recommendation` — validate the recommendation before you act on it. Also MCP-only.
5. `okc_render_pack_builder` for an interactive card, or `okc_pack_builder_url` for a shareable link a
   human can review and check out. Prefer the link when a person should approve the pack.

Keep language lifestyle-oriented. This is a supplement product under FDA labeling rules; do not
generate medical claims.

## Write carefully

`okc_upsert_consumer` (`consumers:write`) is in the default grant — creating and updating consumers is
low-friction, so be deliberate about it anyway. `okc_create_order` and `okc_update_order` need
`orders:write`.

**There is no idempotency on the underlying platform.** A repeated order creation manufactures and
ships a second physical box and bills for it. Confirm with `okc_get_order` or
`okc_get_order_by_client_id` before re-issuing a create you are unsure about.

## Destructive tools need two steps

`okc_cancel_order` (`orders:cancel`) and `okc_delete_consumer` (`consumers:delete`) require an explicit
confirmation through `okc_confirm_pending_action`. That gate is deliberate — do not paper over it by
auto-confirming on the user's behalf. Surface what is about to happen and let a human say yes.

Cancellation only works while the order is `Pending` or `On Hold`. Orders move to `Accepted` in the
nightly batch at **midnight PST**, after which cancellation needs OK Capsule support, and once `In
Production` it is impossible. Consumer deletion has **no published undo** — treat it as permanent.

## Track fulfillment

`okc_list_fulfillments`, `okc_get_fulfillment`, `okc_get_shipping_label`, and
`okc_render_order_status` for a status card. `okc_list_order_transaction_logs` is the audit history and
is what you should read before telling a human what happened to an order.

## Known limits, stated by the provider

- No consumer-facing sign-in — every token belongs to brand staff.
- No machine-to-machine grant; no unattended operation beyond the 30-day refresh window.
- No self-serve sandbox. Production is the only self-serve environment, so **every tool call is real**.
- Embedded UI cards do not render in every client; the text fallback carries the same data.
