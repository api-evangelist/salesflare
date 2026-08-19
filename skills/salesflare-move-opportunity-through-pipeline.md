---
name: Create and advance a Salesflare opportunity through a pipeline
description: >-
  Open a deal against an account, place it on the right pipeline stage, and move
  it forward or close it. Resolves pipeline and stage ids first, because stage is
  an integer id and guessing it silently misfiles the deal.
api: openapi/salesflare-opportunities-api-openapi.yml
operations:
  - getPipelines
  - getStages
  - getAccounts
  - postOpportunities
  - getOpportunities
  - putOpportunitiesId
generated: '2026-08-13'
method: generated
source: >-
  openapi/_original/salesflare-openapi.json +
  data-model/salesflare-data-model.yml
---

# Create and advance a Salesflare opportunity

Base URL: `https://api.salesflare.com`
Auth: `Authorization: Bearer {APIKEY}`

## Resolve the ids first

Salesflare ids are **bare integers with no type prefix**, so `3` could be a stage,
a user or an account. Never hardcode one; resolve it every run.

1. `getPipelines` — list the account's sales processes. Pick the pipeline the deal
   belongs to.
2. `getStages` — list stages. `getStagesStage_id` fetches one. Match by name and
   keep the integer `id`; that is what goes in the opportunity's `stage` field.
3. `getAccounts` with `domain=` or `name=` to resolve the company id. Do not
   create an opportunity against an account you have not confirmed exists.

## Create the opportunity

4. `postOpportunities`. Fields the contract accepts:
   `name`, `account`, `main_contact`, `stage`, `owner`, `creator`, `assignee`,
   `value`, `currency`, `probability`, `status`, `status_date`, `closed`,
   `close_date`, `lost_reason`, `lead_source`, `start_date`,
   `contract_start_date`, `contract_end_date`, `recurring_price_per_unit`,
   `frequency`, `units`, `files`, `tags`, `custom`.
   - `account` (account id) and `stage` (stage id) are the two that matter — they
     are what put the deal in the right column of the right board.
   - `main_contact` is a **contact id**; resolve it with `getContacts` filtered by
     `account`.
   - `currency` must be one Salesflare supports — list them with `getCurrencies`.
   - `owner` / `assignee` / `creator` are user ids.
5. **No idempotency.** A retried `postOpportunities` creates a second deal. If a
   create times out, call `getOpportunities` filtered on `account` and `name`
   before retrying.

## Advance or close it

6. `putOpportunitiesId` (`PUT /opportunities/{id}`) with the changed fields.
   - Advance a stage: send the new `stage` id.
   - Close won/lost: set `closed`, `close_date`, `status`, and `lost_reason` when
     lost.
   - `probability` and `value` are free to update at any stage.
7. Read back with `getOpportunitiesId` and confirm `stage` is what you set. The
   spec types responses as untyped strings, so do not assume the PUT response
   body echoes the record.

## Listing and filtering deals

`getOpportunities` takes `limit` (default **10**), `offset`, `order_by`
(`"value desc"`), `search`, `account`, `pipeline`, and the structured `q` filter
object. See `conventions/salesflare-conventions.yml` for the filter-rule shape —
and note that the fat rule form is marked deprecated in the spec in favour of the
slim `{id, operator, value}` form.

## Errors

`{statusCode, error, message}`. `400` bad body, `401` bad key, `404` unknown id,
`429` rate limited (no headers, back off exponentially), `500` server. No
operation declares any error response, so handle them by status code, not by
schema.
