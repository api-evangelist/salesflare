---
name: Build a filtered, sorted Salesflare list with custom fields
description: >-
  Use Salesflare's structured q filter object, the per-entity filter-field
  registry, and the custom-field registry to build a precise list — the query
  surface every Salesflare list operation shares.
api: openapi/salesflare-filter-fields-api-openapi.yml
operations:
  - getFilterfieldsEntity
  - getCustomfieldsTypes
  - getCustomfieldsItemclass
  - getCustomfieldsItemclassCustomfieldapifieldOptions
  - getAccounts
  - getContacts
  - getOpportunities
generated: '2026-08-13'
method: generated
source: >-
  openapi/_original/salesflare-openapi.json +
  conventions/salesflare-conventions.yml
---

# Build a filtered Salesflare list

Base URL: `https://api.salesflare.com`
Auth: `Authorization: Bearer {APIKEY}`

Filtering in Salesflare is not per-endpoint — it is one shared query surface that
every list operation exposes. Learn it once.

## Discover what is filterable

1. `getFilterfieldsEntity` (`GET /filterfields/{entity}`) — the registry of fields
   you may filter on for that entity. Call this before building a filter instead
   of guessing field names.
2. `getCustomfieldsTypes` then `getCustomfieldsItemclass`
   (`GET /customfields/{itemClass}`) — the tenant's own custom fields. Every
   Salesflare account has a different custom schema, so this is a required step
   for any filter that touches custom data.
3. `getCustomfieldsItemclassCustomfieldapifieldOptions` — the allowed values for a
   dropdown-style custom field.

## The shared query parameters

On `getAccounts`, `getContacts`, `getOpportunities`, `getTasks` and the other list
operations:

- `q` — the **structured filter**. A JSON object passed as a query-string value:
  `{"condition": "AND" | "OR", "rules": [ {"id": ..., "operator": ..., "value": ...} ]}`.
  The spec marks the fat rule form (`customfield_id`, `enabled`, `label`, `type`,
  `query_builder_id`) as *deprecated*: **use the slim `{id, operator, value}` form**.
  Salesflare open-sources the transformer for this format as `@salesflare/optimus`
  on npm, though that package has not been published since 2023-08-02.
- `search` — free-text.
- `custom` — a JSON object keyed by custom-field `api_field`.
- `order_by` — repeated parameter. `"name"` ascending, `"value desc"` descending.
- `limit` — **default 10**, minimum 1.
- `offset` — zero-based.
- Typed shortcuts that are simpler than `q` when they cover your case: `id`,
  `name`, `domain`, `email`, `account`, `pipeline`, `tag`, `tag.name`,
  `address.country`, `address.city`, `address.state_region`,
  `creation_after`/`creation_before`, `modification_after`/`modification_before`.
- `export` — present on five list operations. The spec types it only as a string
  and does not publish the accepted values, so do not rely on it.

## Page correctly

The response is a bare JSON array. No envelope, no total, no `Link` header. Page
by incrementing `offset` by `limit` until a page returns fewer than `limit`
records. Under concurrent writes, offset pagination can skip or repeat rows — for
an exact set, prefer a narrow `q` filter over deep paging.

## Errors

`{statusCode, error, message}`. A malformed `q` object returns `400`. `429` means
back off exponentially; Salesflare publishes no rate-limit headers and no numeric
limit.
