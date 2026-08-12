---
generated: '2026-08-12'
method: generated
name: Serve post-purchase offers and record the resulting events
description: Rank Disco advertiser offers for a shopper on an order-confirmation or
  order-tracking surface, then record the impression, click and conversion events
  that make the placement billable.
api: openapi/disconetwork-partner-api.yml
operations: [getDiscoAdvertiserRecommendations, createAnEventUsedToRecordUserActions,
  createABatchOfEventsUsedToRecordUserActions]
source: >-
  operationIds verified verbatim in openapi/disconetwork-partner-api.yml;
  cross-cutting rules from conventions/disconetwork-conventions.yml,
  authentication/disconetwork-authentication.yml,
  errors/disconetwork-problem-types.yml and sandbox/disconetwork-sandbox.yml.
---

# Serve post-purchase offers and record the resulting events

This is the Partner Integration flow for a publisher that renders its own UI instead of
mounting Disco's DiscoFeed widget. You ask Disco which offers to show, you show them,
and you tell Disco what the shopper did.

## Auth
- Static API key in the `x-api-key` request header. See `authentication/disconetwork-authentication.yml`.
- Keys are **environment-bound**. A staging key against `https://partners.disconetwork.com`
  returns 401 `API key environment does not match service environment`. Staging is
  `https://partners.disconetwork-staging.com` — see `sandbox/disconetwork-sandbox.yml`.
- There is no self-serve signup; a key is issued by a Disco representative.

## Required on every call
- A `version` header is **required** on all three operations (example `1.0.0`). It is
  the only versioning signal this API carries — omitting it is a 400, not a default.

## Steps
1. **Rank the offers** — `getDiscoAdvertiserRecommendations` (`POST /recommendations`)
   with a `RecommendationsRequest` carrying `user_details` and `placement_details`.
   `user_details` is a `oneOf`: supply exactly one identifier — `email`, `email_hash`
   (SHA-256 of the lowercased email), `phone`, or `external_guid`. Render the returned
   offers in your own UI.
2. **Record what the shopper did** — `createAnEventUsedToRecordUserActions`
   (`POST /events`) for a single action, carrying the same `user_details` +
   `placement_details` shape plus the event. A 201 means the event was created.
3. **Or batch them** — `createABatchOfEventsUsedToRecordUserActions`
   (`POST /events/batch`) for 1–20 events in one call. Read the response code before
   the body:
   - `200` — every event accepted.
   - `207` — **partial success**. Walk `results` and retry only the failed members.
     Retrying the whole batch here double-counts the ones that succeeded.
   - `502` — every event failed; retry the whole batch.

## Idempotency
- **There is none.** Disco publishes no idempotency key, no replay window and no
  retry-safety contract on any surface (`conventions/disconetwork-conventions.yml`).
  Because events drive attribution and therefore payout, a blind retry of a `POST /events`
  that actually succeeded will over-report. Track your own per-event dedupe key and
  prefer the batch endpoint's per-item `results` over whole-request retries.

## Errors
- `400` bad request, `401` unauthenticated, `403` unauthorized, `500` server error on all
  three operations, plus `502` on the batch endpoint. Envelopes and remediation are in
  `errors/disconetwork-problem-types.yml`.
- No RFC 9457 `application/problem+json` anywhere on this API.

## Rate limits
- No rate-limit response headers, no documented 429 and no per-endpoint quota are
  published for this surface. See `rate-limits/disconetwork-rate-limits.yml`.
