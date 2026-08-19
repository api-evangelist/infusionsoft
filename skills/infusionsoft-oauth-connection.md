---
name: keap-oauth-connection
description: Connect an application to a Keap account and keep the connection alive — authorization code exchange, the rotating refresh token that locks integrations out when mishandled, and when to use a Personal Access Token or Service Account Key instead.
api: infusionsoft:rest-v2
base_url: https://api.infusionsoft.com/crm
generated: '2026-08-13'
method: generated
source: https://developer.keap.com/getting-started-oauth-keys/ + https://developer.keap.com/pat-and-sak/ + openapi/infusionsoft-rest-v2-openapi.json
operations: []
---

# Connect to a Keap account and stay connected

## Pick the credential type first

| | OAuth 2.0 token | Personal Access Token | Service Account Key |
|---|---|---|---|
| Who creates it | end user, via consent | any app user, in API Settings | administrators only |
| Spans accounts | yes — one app, many Keap accounts | one application | one application |
| Permissions | full account | the creating user's visibility and edit rights | **admin access to everything** |
| Rate limit | 1,500/min · 150,000/day | 240/min · 30,000/day | 240/min · 30,000/day |
| Use it for | a product other people install | your own single-tenant integration | server automation with no user context |

All three are sent identically as `Authorization: Bearer <token>`, with **no prefix that
distinguishes them**. A server receiving one cannot tell which it holds — so your own code must
track it.

Legacy Infusionsoft API keys are **retired** and are no longer accepted.

## The OAuth flow

**1. Send the user to authorize:**

```
https://accounts.infusionsoft.com/app/oauth/authorize
  ?client_id=<your client id>
  &redirect_uri=<your HTTPS callback>
  &response_type=code
  &scope=full
```

`redirect_uri` **must be HTTPS**. `response_type=code` is the only valid value.
`scope=full` is the only valid scope value — see the warning below.

**2. Exchange the code:**

```
POST https://api.infusionsoft.com/token
Authorization: Basic base64(client_id:client_secret)
Content-Type: application/x-www-form-urlencoded

client_id=...&client_secret=...&code=...&grant_type=authorization_code&redirect_uri=...
```

Client credentials go in the **HTTP Basic header**, not only the body.

**3. Call the API:**

```
Authorization: Bearer <access_token>
```

against `https://api.infusionsoft.com/crm/rest/v2/...` (or `.../rest/v1/...`).
`https://api.keap.com/crm/...` serves the same API and is the host Keap's own generated SDKs
are built against.

## Refresh tokens rotate — this is the failure mode

Every refresh returns a **new refresh token**, and the old one stops working.

```
POST https://api.infusionsoft.com/token
Authorization: Basic base64(client_id:client_secret)
Content-Type: application/x-www-form-urlencoded

grant_type=refresh_token&refresh_token=<current refresh token>
```

The way integrations die here, in order of frequency:

1. **Persisting the access token but not the new refresh token.** The next refresh fails and
   the only recovery is sending the user through consent again. Write the new refresh token to
   durable storage *before* you use the new access token.
2. **Two workers refreshing concurrently.** The second refresh presents an already-rotated
   token and fails, and depending on ordering you can lose the good one. Serialise refresh
   behind a lock or a single owner per connection.
3. **Retrying a failed refresh with the same token.** It is already spent. Treat a failed
   refresh as terminal for that token and escalate to re-consent.

Access-token lifetime is whatever `expires_in` says on the response — Keap publishes no fixed
value, so read the field rather than hard-coding a TTL.

## The scope warning

Keap has **no granular OAuth scopes**. `full` is the only accepted value, and every published
spec declares the oauth2 scheme with an empty scopes object. A user who connects your
integration grants complete read and write access to their CRM: contacts, orders,
subscriptions, emails, automations, affiliates and settings. There is no read-only grant and no
incremental consent.

Consequences you should design around:

- Say plainly in your own consent screen what the integration will do, because Keap's will not.
- If you only need narrow access, create a **Personal Access Token under a limited (non-admin)
  Keap user**. The PAT inherits that user's visibility and editing permissions, which is the
  only least-privilege mechanism Keap actually offers. It is a user-permission workaround, not
  an API scope.
- Never use a Service Account Key where a PAT would do — a SAK is admin over all stored data
  by construction.

## Where the keys live

- Create applications and client credentials: <https://keys.developer.keap.com/>
- Create a developer account: <https://keys.developer.keap.com/accounts/create>
- Free sandbox tenant to test against: <https://sandbox.keap.com/> — note it uses the **same**
  API hosts as production and there is no test-mode key prefix, so guard against pointing
  production credentials at test code by tracking the application instance yourself.

## Handling the auth failures

The Apigee gateway rejects bad tokens before the API is reached, returning a shape documented
in **no** Keap OpenAPI:

```json
{"fault":{"faultstring":"Invalid access token","detail":{"errorcode":"oauth.v2.InvalidAccessToken"}}}
```

Parse for `fault.detail.errorcode` alongside the documented v2 `{code, message, status, details}`
envelope. A `401` with `oauth.v2.InvalidAccessToken` means refresh; a `403` means the token is
valid but the user or service account lacks permission, and refreshing will not help.
