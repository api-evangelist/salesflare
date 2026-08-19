---
name: Create a Salesflare account and attach a contact
description: >-
  Add a company to Salesflare and attach a person to it without creating
  duplicates. Covers the dedupe-before-create step that the Salesflare API does
  not do for you, because none of its create operations are idempotent.
api: openapi/salesflare-accounts-api-openapi.yml
operations:
  - getAccounts
  - postAccounts
  - postContacts
  - postAccountsAccount_idContacts
generated: '2026-08-13'
method: generated
source: >-
  openapi/_original/salesflare-openapi.json +
  conventions/salesflare-conventions.yml
---

# Create a Salesflare account and attach a contact

Base URL: `https://api.salesflare.com`
Auth: `Authorization: Bearer {APIKEY}` on every request. Keys are made in the
Salesflare app under Settings > API keys. The key is account-wide and unscoped —
it can do anything the creating user can do.

## Before you write anything: dedupe

**Salesflare has no idempotency key.** `postAccounts` and `postContacts` accept no
`Idempotency-Key` header and no client-supplied reference, so a retried or
double-submitted create makes a second record. Always search first.

1. `getAccounts` with `domain={company domain}` — the domain filter is a repeated
   query parameter and is the most reliable account identity signal Salesflare has.
   Fall back to `name={company name}` or `search={free text}` when you have no domain.
2. If the array comes back non-empty, use the existing `id`. Do not create.

Remember the list contract: bare JSON array, `limit` defaults to **10**, `offset`
is zero-based, and there is no total count and no next-page link. If you get
exactly `limit` records back, page with `offset` until you get fewer.

## Create the account

3. `postAccounts` with a JSON body. Fields the published contract accepts:
   `name`, `domain`, `website`, `picture`, `size`, `description`, `owner`,
   `parent_account`, `address`, `addresses`, `email`, `email_addresses`,
   `phone_number`, `phone_numbers`, `social_profiles`, `links`, `tags`,
   `customers`, `custom`.
   - `owner` is a **user id** (integer). Get it from `getMe` for "assign to me",
     or from `getUsers`.
   - `parent_account` is an **account id** — accounts nest.
   - `custom` is an object keyed by custom-field api_field name. Discover the
     tenant's custom schema with `getCustomfieldsItemclass` before you send it.
4. Keep the returned `id`. If the call fails mid-flight and you have to retry,
   re-run step 1 first.

## Create and attach the contact

5. `postContacts` for the person.
   **Contract gap:** the published spec declares the body parameter for
   `postContacts` with *no properties*, so the payload shape is not discoverable
   from the OpenAPI. Use the field set recorded in
   `json-schema/salesflare-schemas.json` (`prefix`, `firstname`, `middle`,
   `lastname`, `suffix`, `email[]`, `phone_number[]`, `account`, `owner`, `tags[]`)
   and treat unknown fields as unsupported rather than guessing.
6. Link the person to the company with `postAccountsAccount_idContacts`
   (`POST /accounts/{account_id}/contacts`). Setting `account` on the contact and
   using this route both express the same relationship; use the route when you are
   attaching an existing contact.

## Verify

7. `getContacts` with `account={account_id}` and confirm the person is there.
   This read-back is the substitute for the retry safety Salesflare does not give
   you.

## Errors

Salesflare returns a `{statusCode, error, message}` envelope — not RFC 9457, and
**no operation in the spec declares a 4xx or 5xx response**, so your generated
client will not model errors. Handle these by status:

- `400` — malformed JSON body or a bad value type.
- `401` — missing or invalid bearer key.
- `404` — the account or contact id does not exist for this API key's account.
- `429` — rate limited. Salesflare publishes no numeric limit, no `Retry-After`
  and no `RateLimit-*` headers. Back off exponentially.
- `500` — retry, then contact support@salesflare.com.
