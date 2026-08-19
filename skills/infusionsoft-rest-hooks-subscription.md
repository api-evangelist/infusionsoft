---
name: keap-rest-hooks-subscription
description: Subscribe to Keap change events with REST Hooks, complete the verification handshake, and build a receiver that survives Keap's retry and deactivation policy. Use this instead of polling — and note that the entire event surface lives on REST v1 even if the rest of your integration is on v2.
api: infusionsoft:rest-v1
base_url: https://api.infusionsoft.com/crm/rest/v1
generated: '2026-08-13'
method: generated
source: openapi/infusionsoft-rest-v1-openapi.json + https://developer.keap.com/rest-hook-documentation/
operations:
  - list_hook_event_types
  - create_a_hook_subscription
  - list_stored_hook_subscriptions
  - retrieve_a_hook_subscription
  - update_a_hook_subscription
  - delete_a_hook_subscription
  - verify_a_hook_subscription
  - verify_a_hook_subscription_delayed
---

# Subscribe to Keap events with REST Hooks

## The one thing to know first

**REST Hooks exist only on REST v1.** There are eight hook operations under `/rest/v1/hooks`
and **zero** hook operations anywhere in the 399-operation v2 spec. A v2 integration still has
to reach back into v1 to manage its event subscriptions. Do not go looking for a v2 equivalent.

## 1. Discover what you can subscribe to

`list_hook_event_types` — `GET /rest/v1/hooks/event_keys`

Keap does **not** publish a static list of event keys. The catalogue is per-application and is
only available from this authenticated endpoint, so call it at setup time rather than
hard-coding keys. Keys follow a `resource.action` shape; `contactGroup.applied` and
`contactGroup.delete` are the two named in the public documentation.

## 2. Create the subscription

`create_a_hook_subscription` — `POST /rest/v1/hooks`

```json
{ "eventKey": "contactGroup.applied", "hookUrl": "https://your-app.example.com/keap/hooks" }
```

The response is a `RestHook`: `{ key, eventKey, hookUrl, status }` where `status` is one of
`Unverified`, `Verified`, `Inactive`. Store `key` — it is the handle for every later operation.

## 3. Complete the handshake

`verify_a_hook_subscription` — `POST /rest/v1/hooks/{key}/verify`

**Events are delivered only while `status` is `Verified`.** A freshly created subscription is
`Unverified` and will receive nothing. `verify_a_hook_subscription_delayed`
(`POST /rest/v1/hooks/{key}/delayedVerify`) is the delayed variant, for when your receiver is
not up yet at subscription time.

## 4. Build the receiver

Delivery payload — batched, up to **1,000 changed objects of the same event type** per delivery:

```json
{
  "event_key": "<event key>",
  "object_type": "<resource name>",
  "object_keys": [
    { "id": 123, "apiUrl": "<REST API URL for the resource>", "timestamp": "<event timestamp>" }
  ]
}
```

The payload carries **ids, not object state**. Your receiver must call back into the REST API
(`getContact`, `getOrder`, …) to read what actually changed. Budget those callbacks against
the same rate limits your foreground traffic uses.

Receiver requirements, in the order they will bite you:

1. **Respond within 30 seconds.** A slower response counts as a failure. Acknowledge first,
   process asynchronously.
2. **Respond 2xx–3xx.** Anything `<200` or `>=400` (except 410) counts as a failure.
3. **Never return 410 unless you mean it.** A single 410 marks the subscription `Inactive`
   immediately, with no retries.
4. **Expect duplicates and out-of-order batches.** Deliveries are batched and retried, and
   Keap provides no delivery id or sequence number — dedupe on `(event_key, id, timestamp)`.

## 5. Survive the retry policy

Four attempts, then deactivation:

| Attempt | When |
|---|---|
| 1 | 30–60s after the change (5–10 **minutes** for `contactGroup.applied` and `contactGroup.delete`) |
| 2 | 30–60s after the previous failure |
| 3 | 5 minutes after the previous failure |
| 4 | 30 minutes after the previous failure |

If the fourth attempt fails, the subscription becomes `Inactive` and **stays that way until you
call `/verify` again**. Keap will not tell you this happened. Poll
`list_stored_hook_subscriptions` (`GET /rest/v1/hooks`) on a schedule, and re-verify anything
that has drifted to `Inactive` — an unmonitored integration goes silently deaf after roughly
36 minutes of receiver downtime.

## Security — read this before you deploy

**Keap does not sign webhook deliveries.** There is no HMAC header, no `X-Hook-Secret`, no
shared secret, and no timestamp signature. The only thing separating a real delivery from a
forged one is the secrecy of your `hookUrl`.

Practical mitigations, since you cannot verify the sender cryptographically:

- Use a long unguessable path segment in `hookUrl` and treat it as a credential.
- Treat the payload as a **hint, not a fact** — always re-read the object from the API before
  acting on it. This is what makes forgery useless: the worst a forged delivery achieves is a
  wasted read.
- Never act on a delivery that references an id your account does not own.

## Housekeeping

- `retrieve_a_hook_subscription` — `GET /rest/v1/hooks/{key}`
- `update_a_hook_subscription` — `PUT /rest/v1/hooks/{key}` (change `eventKey` or `hookUrl`)
- `delete_a_hook_subscription` — `DELETE /rest/v1/hooks/{key}`
