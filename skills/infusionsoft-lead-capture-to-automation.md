---
name: keap-lead-capture-to-automation
description: Capture a new lead into Keap without creating a duplicate, tag it, and hand it to a marketing automation by achieving a goal. This is the canonical Keap onboarding flow — everything downstream (email sequences, opportunity creation, follow-up tasks) is driven by the campaign the goal starts.
api: infusionsoft:rest-v2
base_url: https://api.infusionsoft.com/crm/rest/v2
generated: '2026-08-13'
method: generated
source: openapi/infusionsoft-rest-v2-openapi.json
operations:
  - listContacts
  - createContact
  - updateContact
  - listTags
  - applyTags
  - achieveGoal
  - addContactsToAutomationSequence
---

# Capture a lead into Keap and start an automation

## Before you start

- Authenticate with `Authorization: Bearer <token>`. Any Keap token is a **full-account
  credential** — there are no granular OAuth scopes (`scope=full` is the only valid value).
  Do not assume a read-only mode exists.
- **Keap has no idempotency key.** If `createContact` times out, do not blindly retry —
  you will create a second contact. Use the duplicate-check path in step 2 instead.
- All list responses page with `page_token` / `page_size` and return `next_page_token`.

## 1. Look for the person before you create them

`listContacts` — `GET /rest/v2/contacts`

Use the `filter` parameter. The email match is the reliable one:

```
GET /rest/v2/contacts?filter=email==jane@example.com&page_size=10
```

`filter` also accepts `given_name`, `family_name`, `company_id`, `ids` (comma separated),
`start_update_time` / `end_update_time`, and `phone_number` — note that `phone_number`
**requires** `phone_fields` naming which of PHONE1..PHONE5 to search, or it will not match.

## 2. Create or update

`createContact` — `POST /rest/v2/contacts`

Pass `duplicate_option` on the query string. When supplied, Keap performs its own duplicate
check and **updates the existing contact instead of creating a second one** — this is the
closest thing Keap offers to an idempotent create, and it is the reason step 1 alone is not
enough under concurrency.

Body is `CreateUpdateContactRequest`. Two traps:

- Array fields (`addresses`, `email_addresses`, `phone_numbers`) are **replace, not merge**.
  Anything you omit is deleted; an empty array wipes the existing values. Read the contact
  first and send the full array if you are adding one entry.
- Custom fields are per-tenant. Call `retrieveContactModel` (`GET /rest/v2/contacts/model`)
  to resolve custom field ids for this account. Never hard-code a custom field id — the same
  id means something different in another Keap application.

If you already resolved an existing contact in step 1, use `updateContact`
(`PATCH /rest/v2/contacts/{contact_id}`) instead.

## 3. Tag the contact

`listTags` — `GET /rest/v2/tags` to resolve the tag id (tag ids are also per-tenant).

`applyTags` — `POST /rest/v2/tags/{tag_id}/contacts:applyTags`

```json
{ "contact_ids": ["123", "456"] }
```

`contact_ids` is required and is an array — apply the tag to a batch rather than looping,
which also keeps you well under the per-minute throttle. The inverse is `removeTags`
(`POST /rest/v2/tags/{tag_id}/contacts:removeTags`).

Note the tag ids you use here are also what a `contactGroup.applied` REST Hook event will
tell you about later — see the event-subscription skill.

## 4. Hand off to the automation

`achieveGoal` — `POST /rest/v2/automations/goals/achieve`

This is how an outside system starts a Keap campaign. The request body accepts **two mutually
exclusive addressing modes**, plus `contact_id`:

- `integration` + `call_name` — the API-goal pair configured in the Keap campaign builder.
  Use this one for integrations; it survives the campaign being rebuilt.
- `automation_id` + `goal_id` — direct ids. Brittle, because the ids change when the
  automation is republished.

Supplying both, or neither, is a 400.

If you need to place the contact at a specific point in an automation rather than trigger a
goal, use `addContactsToAutomationSequence` —
`POST /rest/v2/automations/{automation_id}/sequences/{sequence_id}:addContacts`.

## Errors and limits

- **429**: quota or throttle exhausted. Read `x-keap-product-throttle-available` and
  `x-keap-product-quota-available` on every response and stay ahead of it. There is **no
  `Retry-After` header** — back off exponentially with jitter. Limits are 1,500/min and
  150,000/day for an OAuth token, 240/min and 30,000/day for a PAT or Service Account Key,
  and 10,000/min and 250,000/day for the whole application instance.
- **401**: the Apigee gateway answers before the API, with a shape that appears in no spec:
  `{"fault":{"faultstring":"Invalid access token","detail":{"errorcode":"oauth.v2.InvalidAccessToken"}}}`.
  Parse for `fault` as well as the documented `{code, message, status, details}` envelope.
- **409**: conflicting state. Because there is no idempotency key, reconcile by reading
  before you write again rather than retrying.
- Refresh tokens **rotate**. Every refresh returns a new refresh token; persist it or the
  integration locks itself out.
