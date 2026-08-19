---
name: Review a Salesflare account timeline and leave an internal note
description: >-
  Pull the full activity history for a company — emails, meetings, notes, web
  visits — summarize it, and post an internal note with @mentions back to the
  account. The read half of what Salesflare's own MCP server advertises.
api: openapi/salesflare-accounts-api-openapi.yml
operations:
  - getAccounts
  - getAccountsAccount_idFeed
  - getAccountsAccount_idMessages
  - getMe
  - postMessages
generated: '2026-08-13'
method: generated
source: >-
  openapi/_original/salesflare-openapi.json +
  mcp/salesflare-tool-crosswalk.yml
---

# Review a Salesflare account timeline

Base URL: `https://api.salesflare.com`
Auth: `Authorization: Bearer {APIKEY}`

## Resolve the account

1. `getAccounts` with `domain=`, `name=` or `search=`. Take the integer `id`.
   Useful narrowing filters on this operation: `tag`, `tag.name`,
   `address.country`, `address.city`, `min_size`/`max_size`, `hotness`
   (1 room temp, 2 hot, 3 on fire), and `modification_after` /
   `modification_before` for "what moved recently".

## Read the timeline

2. `getAccountsAccount_idFeed` (`GET /accounts/{account_id}/feed`) — the activity
   stream: emails, meetings, calls, web visits, notes.
3. `getAccountsAccount_idMessages` (`GET /accounts/{account_id}/messages`) — the
   internal notes thread specifically.

Both are list operations: `limit` defaults to **10**, `offset` is zero-based,
`order_by` takes `"key"` or `"key desc"`. There is no total count and no next-page
link, so keep paging until a page returns fewer records than `limit`. For a full
timeline you will almost always need to raise `limit` and page — the default of 10
will silently give you a partial history, which is the single easiest way to
summarize an account wrongly.

## Summarize honestly

Response bodies are typed as untyped strings in the published contract — there are
zero named schemas in the whole spec. Read the JSON you actually receive; do not
assume field names that the spec never declared. If a field you need is absent,
say it is absent rather than inferring it.

## Post the note back

4. `getMe` to resolve your own user id if you need to mention or attribute.
5. `postMessages` with `{account, body, date, mentions}`:
   - `account` — the account id from step 1.
   - `body` — the note text.
   - `mentions` — an array of **user ids**. Resolve names to ids with `getUsers`;
     never guess an id.
6. **No idempotency.** A retried `postMessages` posts the note twice. On a
   timeout, re-read `getAccountsAccount_idMessages` before retrying.

## Errors

`{statusCode, error, message}`. `401` bad key, `404` unknown account,
`429` rate limited — no `Retry-After` and no `RateLimit-*` headers are published,
so back off exponentially. No operation in the spec declares an error response.
