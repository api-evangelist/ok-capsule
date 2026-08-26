---
name: ok-capsule-create-personalized-supplement-order
description: >-
  Create a personalized daily supplement pack order on OK Capsule for a named consumer, from catalog
  lookup through order confirmation, without double-shipping on a retry.
generated: '2026-08-26'
method: generated
source: >-
  openapi/ok-capsule-core-api-v2-openapi.yaml, https://docs.okcapsule.app/docs/getting-started,
  https://docs.okcapsule.app/docs/recipes/error-handling,
  https://docs.okcapsule.app/docs/recipes/troubleshooting
api: OK Capsule Core API V2
base_url: https://na1-prod.okcapsule.app
operations:
  - createAuthenticationToken
  - listProducts
  - getProduct
  - listConsumers
  - createConsumer
  - getOrderByClientCustomId
  - createOrder
  - getOrder
---

# Create a personalized supplement order

This writes to a physical manufacturing line. A successful `createOrder` causes supplements to be
packed and shipped, and it is billed per pack. Read the whole skill before the first write.

## Before you start

- Credentials are not self-serve. The account and at least one Product Line must exist, and an OK
  Capsule representative issues the API credentials.
- A Packaging Asset Group must be configured and Active for the product line, or the order will land in
  `Needs Changes` instead of being produced.
- Use `https://na1-stage.okcapsule.app` while developing. Stage credentials do not work in production
  and vice versa. There is no test mode on the production host and no key prefix to tell them apart —
  the only difference is the base URL, so check it before every write.

## 1. Authenticate

`POST /v2/authentication/token` with `{"username": "<email>", "password": "<password>"}`.

Keep both `access_token` and `refresh_token`. The access token lives 24 hours. Refresh proactively at
roughly 80% of TTL with `POST /v2/authentication/refresh-token`; do not wait for a 401. Send
`Authorization: Bearer <access_token>` on everything below.

## 2. Resolve the products

`listProducts` (`GET /v2/products`) returns the brand's own catalog. You need the `id` of each product
the pack will contain — a bare UUID, no prefix.

Only order products where `active` is `true`. Ordering an inactive one returns
`The following client products are inactive and cannot be ordered: [names]`. If the *underlying* OK
Capsule master product has been discontinued you get the parallel `The following OKC products are
inactive` error, and that one you cannot fix yourself — it needs OK Capsule support.

If you were given a product id from elsewhere, confirm it with `getProduct` (`GET /v2/products/{id}`)
before putting it in an order. It is cheaper than a 422.

## 3. Resolve or create the consumer

Two mutually exclusive patterns, and mixing them is the most common error on this API:

- **Existing consumer** — pass `consumer: {id: "<uuid>"}`. Find it with `listConsumers`
  (`GET /v2/consumers`).
- **New consumer** — pass `consumer: {first_name, last_name, email}`. Both names are required; sending
  `first_name` alone returns `"last_name" is required`.

Do not send `id` *and* names. `createConsumer` (`POST /v2/consumers`) exists if you would rather create
the consumer as its own step, but the order endpoint will do it inline.

## 4. Check for a duplicate before you write

**This API has no idempotency key.** The string does not appear in the contract or the docs. If you
retry `createOrder` after a timeout, an ambiguous 5xx, or a 401-refresh-and-retry, and the first
request actually succeeded, you have manufactured and shipped two boxes and the brand is billed for
both.

Build your own guard. Set `client_custom_order_id` to a value you generate and can reproduce, then
before any retry call `getOrderByClientCustomId`
(`GET /v2/orders/by-client-order-id/{clientCustomOrderId}`). If it returns an order, the write landed —
do not send it again.

This is a read-then-write check, not real idempotency. It closes the retry case, not a genuine race.

## 5. Create the order

`createOrder` (`POST /v2/orders`). The required shape:

- `consumer` — per step 3.
- `shipping_address` — `address1`, `city`, `country_name` are required; `province_name` and
  `postal_code` should be supplied. `address1` must be under 100 characters and `city` under 50.
- `order_lines` — at least one. Each line carries a `duration` (days, e.g. 30) and a `pouches` array.
- Each pouch carries `time_of_administration` (e.g. `"Morning"`) and **either** `pack_id` (an
  assembly) **or** `contents` (a list of `{client_product_id, serving_size}`). Never both — sending
  both returns `Either pack_id or contents required`.
- `client_custom_order_id` — your correlation key from step 4.

The response returns the order with `id` and `status`, which will be `Pending`.

## 6. Confirm and hand off

Call `getOrder` (`GET /v2/orders/{id}`) and report the real status back rather than assuming success.

If the status is `Needs Changes`, read the order's `note` field — the usual causes are an unapproved
packaging design or a product that has gone inactive. Neither is fixable through the API; escalate to
OK Capsule and create a fresh order once resolved.

## Error handling

Field-level causes arrive in the error body. Note the contract and the docs disagree on its shape: the
OpenAPI declares `ValidationError` with `errors[]` (each carrying `message` and a Joi `type`), while
the docs demonstrate `details[]` (each carrying `field` and `message`). Parse defensively for both.

- **401** — token expired. Refresh, then retry. Because there is no idempotency key, run the step 4
  duplicate check before retrying a write.
- **403** — the signed-in user's role lacks the permission. An admin fixes this in the client portal.
- **422** — validation. Read the field-level detail; see `errors/ok-capsule-problem-types.yml` in this
  repo for the full catalog of documented messages and their fixes.
- **429** — back off. Honour `Retry-After`; the provider prescribes 1s / 2s / 4s exponential backoff.
  `X-RateLimit-Limit`, `-Remaining` and `-Reset` are returned but no numeric quota is published.

## Undo

You can cancel while the order is `Pending` or `On Hold`, with
`PUT /v2/orders/{id}` and `{"status": "Canceled by Client"}`.

The window is real and it is short: orders move to `Accepted` in a nightly batch at **midnight PST**.
After that, cancellation requires contacting OK Capsule support, and once the order is `In Production`
it cannot be canceled at all. If a human may want to reverse this order, tell them that deadline when
you report the order was placed.
