---
name: keap-order-and-subscription-billing
description: Create a Keap order for a contact, add line items, record a payment, and stand up a recurring subscription against a product's subscription plan. Covers the e-commerce half of Keap, where the absence of an idempotency key is most expensive.
api: infusionsoft:rest-v2
base_url: https://api.infusionsoft.com/crm/rest/v2
generated: '2026-08-13'
method: generated
source: openapi/infusionsoft-rest-v2-openapi.json
operations:
  - listProducts
  - getProduct
  - listSubscriptionPlans
  - createOrder
  - getOrder
  - createOrderItem
  - listOrderPayments
  - createPaymentForAnOrder
  - applyTax
  - listSubscriptions
  - createSubscription
  - invoiceSubscription
  - cancelSubscription
---

# Take an order and set up recurring billing in Keap

## Money plus no idempotency: read this first

Keap exposes **no idempotency key** on any of its 585 published operations. On a billing flow
that matters more than anywhere else: a `createOrder` or `createPaymentForAnOrder` that times
out may well have succeeded. **Never blind-retry a write in this skill.** The safe pattern is:

1. Generate your own external reference and put it in the order `title` or `notes`.
2. On any timeout or 5xx, call `listOrders` (`GET /rest/v2/orders`) filtered to the contact and
   look for your reference **before** retrying.
3. Only create if the read proves nothing landed.

## 1. Resolve the product

`listProducts` — `GET /rest/v2/products` (pages with `page_token` / `page_size`)
`getProduct` — `GET /rest/v2/products/{product_id}`

Product ids are per-tenant. Resolve them at runtime; do not carry them between Keap accounts.

For recurring billing, read the plans attached to the product:
`listSubscriptionPlans` — `GET /rest/v2/products/{product_id}/subscriptions`.
A Keap subscription is always created against a **plan on a product**, never against a bare price.

## 2. Create the order

`createOrder` — `POST /rest/v2/orders`

Body is `RestCreateOrderRequest`: `contact_id`, `order_title`, `order_time`, `order_items[]`,
plus `title`, `terms` and `notes`. Passing `order_items` inline creates the order and its lines
in **one** call — prefer that over creating an empty order and adding items, because it is one
non-idempotent write instead of several.

Add lines later, if you must, with `createOrderItem` —
`POST /rest/v2/orders/{order_id}/items`.

Apply tax with `applyTax` — `POST /rest/v2/orders/{order_id}:applyTax`. Note the colon-suffixed
custom verb; v2 uses these for actions rather than PUT/PATCH, and calling the plain path with
the wrong verb returns 405.

## 3. Record the payment

`createPaymentForAnOrder` — `POST /rest/v2/orders/{order_id}/payments`
`listOrderPayments` — `GET /rest/v2/orders/{order_id}/payments`

Always `listOrderPayments` before recording a payment on an order you have already touched.
This is the read-before-write guard from the top of this skill, applied where a duplicate
costs a customer real money.

Keap publishes **no decline-code reference**. The payment result comes back in the order and
payment records themselves; there is no documented registry of gateway decline reasons to map
against, so surface the message you get rather than trying to classify it.

## 4. Stand up the subscription

`createSubscription` — `POST /rest/v2/subscriptions`
`listSubscriptions` — `GET /rest/v2/subscriptions`
`getSubscription` — `GET /rest/v2/subscriptions/{subscription_id}`
`updateSubscription` — `PATCH /rest/v2/subscriptions/{subscription_id}`

Bill an existing subscription immediately with `invoiceSubscription` —
`POST /rest/v2/subscriptions/{subscription_id}:invoice`.

Cancel with `cancelSubscription` —
`POST /rest/v2/subscriptions/{subscription_id}:deactivate`. Note the verb is `:deactivate`,
not a DELETE — deleting is not how a Keap subscription ends.

## 5. Custom fields

Orders and subscriptions both carry per-tenant custom field models:

- `retrieveOrderCustomFieldModel` — `GET /rest/v2/orders/model`
- `retrieveSubscriptionCustomFieldModel` — `GET /rest/v2/subscriptions/model`

Read the model to resolve ids before writing custom values, every time, in every account.

## Errors and limits

- **429** on any of the above: read `x-keap-product-throttle-available` and
  `x-keap-product-quota-available`, back off exponentially with jitter. There is no
  `Retry-After`.
- **401**: the gateway fault envelope
  (`{"fault":{"faultstring":...,"detail":{"errorcode":...}}}`) precedes the documented
  `{code, message, status, details}` error object. Handle both.
- **409**: state conflict. Reconcile by reading; do not retry.
- The v2 spec declares 400/401/403/404/405/409/500/501 on every operation and **429 on none of
  them**, so your 429 handling has to come from the docs, not the contract.
