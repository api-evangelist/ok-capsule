---
name: ok-capsule-track-order-to-delivery
description: >-
  Follow an OK Capsule order from Pending through manufacturing to a tracked shipment, retrieve the
  shipping label, and poll at a rate the provider actually supports.
generated: '2026-08-26'
method: generated
source: >-
  openapi/ok-capsule-core-api-v2-openapi.yaml,
  https://docs.okcapsule.app/docs/recipes/order-lifecycle,
  https://docs.okcapsule.app/docs/recipes/track-fulfillments
api: OK Capsule Core API V2
base_url: https://na1-prod.okcapsule.app
operations:
  - getOrder
  - getOrderByClientCustomId
  - listOrders
  - getOrderFulfillments
  - listFulfillments
  - getFulfillment
  - getFulfillmentLabelUrl
  - listOrderTransactionLogs
  - listStatuses
---

# Track an order to delivery

OK Capsule publishes **no webhooks and no AsyncAPI**. There is no event stream to subscribe to, so
tracking is polling, and polling this API too hard is both useless and rate-limited.

## The status ladder

`listStatuses` (`GET /v2/statuses`) returns the live vocabulary. The documented ladder:

| Status | Meaning | Editable | Cancelable |
|---|---|---|---|
| `Pending` | Received, awaiting the nightly batch | yes | yes |
| `On Hold` | Paused for review | yes | yes |
| `Accepted` | Validated, queued for production | no | support only |
| `In Production` | Being manufactured | no | no |
| `Shipped` | Handed to carrier, tracking available | no | no |
| `Delivered` | Confirmed delivery | no | no |
| `Canceled by Client` | You requested cancellation | — | — |
| `Canceled by OKC` | OK Capsule canceled it | — | — |
| `Needs Changes` | Validation failed during processing | — | — |

Two timings drive everything: orders move `Pending` → `Accepted` in a **nightly batch at midnight
PST**, and statuses are otherwise **updated roughly every 6 hours** during business operations.

## Poll correctly

`listOrders` (`GET /v2/orders`) with the filter syntax the docs demonstrate:

```
GET /v2/orders?filter[status]=Shipped&filter[updated_at][gte]=<iso8601>&limit=50
```

Be aware that neither `filter[...]` nor `limit` is declared in the OpenAPI — they are documented but
not contracted, so a generated client will not know about them and they are not guaranteed.

**Do not poll more than a few times a day.** The provider states outright that polling more frequently
than every few hours is unnecessary and may be rate limited. A once-per-hour sweep filtered on
`updated_at` is more than enough; a per-order tight loop is not.

For a single order use `getOrder` (`GET /v2/orders/{id}`), or `getOrderByClientCustomId`
(`GET /v2/orders/by-client-order-id/{clientCustomOrderId}`) when you only kept your own correlation id.

## When it ships

A fulfillment record is created when the order ships.

- `getOrderFulfillments` (`GET /v2/orders/{id}/fulfillments`) — fulfillments for one order.
- `listFulfillments` (`GET /v2/fulfillments`) — across orders.
- `getFulfillment` (`GET /v2/fulfillments/{id}`) — one record, with carrier tracking.
- `getFulfillmentLabelUrl` (`GET /v2/fulfillments/{id}/label`) — a **signed** label URL. Treat it as
  short-lived and single-purpose: fetch it when you need it rather than caching it, and never paste it
  into a channel a customer can read.

A fulfillment carries `order_id`, `consumer_id`, `status_id`, `shipment_order_id` and
`external_fulfillment_id` — the last is the carrier-side handle, useful when reconciling against a
3PL.

## When something looks wrong

`listOrderTransactionLogs` (`GET /v2/order-transaction-logs`) is the event history. Use it before
telling a human what happened — it is the only audit surface, and it is what distinguishes "the order
was never accepted" from "the order was canceled".

If an order sits in `Needs Changes`, read its `note` field. The documented causes are an inactive or
unapproved packaging design and a product that is no longer available; both need OK Capsule support,
and the resolution is a new order, not a repair of this one.

## What you cannot do

- There is no webhook or callback. Do not promise a human real-time notification.
- There is no delivery-estimate endpoint. Carrier tracking on the fulfillment is the only forward-looking
  signal.
- Once `In Production`, nothing about the order can be changed or stopped through the API.
