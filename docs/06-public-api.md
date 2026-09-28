# 06 — Public API Contract

Machine-readable sketch: [`api/openapi.yaml`](api/openapi.yaml).

## Principles

- **REST over JSON**, resource-oriented, canonical shapes from [02](02-canonical-data-model.md).
- **Versioned in the path**: `/v1/`. Breaking changes mean `/v2/`, with `/v1/`
  supported for a published deprecation window.
- **Additive changes are not breaking**: clients must tolerate new fields. Stated
  explicitly in the contract so we can ship without a version bump.
- **Provider-neutral surface.** No endpoint is named after a CRM. Provider detail
  appears only in `sources[]`, capability documents, and connection resources.

## Endpoint groups

### Connections

```
GET    /v1/connections                     list this tenant's connections
POST   /v1/connections                     begin a connect flow → returns authorize_url or key-request instructions
GET    /v1/connections/{id}                status, instance profile, health
PATCH  /v1/connections/{id}                rename, enable/disable
DELETE /v1/connections/{id}                revoke credentials and stop syncing
POST   /v1/connections/{id}/test           run a health check now
GET    /v1/connections/{id}/capabilities   the full capability matrix — see doc 03
GET    /v1/connections/{id}/sync-status    per-entity cursor, lag, last error
POST   /v1/connections/{id}/resync         trigger backfill or targeted resync
GET    /v1/providers                       supported providers + their connect requirements
```

`POST /v1/connections` returns different next steps by provider, because the flows
genuinely differ ([04](04-authentication-and-tenancy.md)):

```jsonc
// Salesforce / Blackbaud — self-service OAuth
{ "connection_id": "conn_…", "status": "pending",
  "next": { "kind": "redirect", "authorize_url": "https://…" } }

// Bonterra EveryAction — human-in-the-loop key issuance
{ "connection_id": "conn_…", "status": "pending",
  "next": { "kind": "manual_key_request",
            "instructions_url": "https://docs.conconnect.example/connect/everyaction",
            "expected_lead_time": "5-15 business days",
            "submit_key_at": "/v1/connections/conn_…/credentials" } }
```

Surfacing the lead time in the API response, not just in a help article, is
deliberate: the onboarding UI can then set an honest expectation up front.

### Core entities

Uniform CRUD per entity: `constituents`, `organizations`, `households`,
`relationships`, `designations`, `gifts`, `commitments`, `activities`, `notes`.

```
GET    /v1/{entity}                list (filter, sort, cursor-paginate)
POST   /v1/{entity}                create
GET    /v1/{entity}/{id}           read
PATCH  /v1/{entity}/{id}           partial update
DELETE /v1/{entity}/{id}           delete (subject to capability)
POST   /v1/{entity}/search         complex queries exceeding a query string
```

Relationship sub-resources where it aids clarity:

```
GET  /v1/constituents/{id}/gifts
GET  /v1/constituents/{id}/commitments
GET  /v1/constituents/{id}/activities
GET  /v1/constituents/{id}/relationships
GET  /v1/commitments/{id}/gifts          the installments paid against it
```

### Batch

```
POST /v1/batch          up to 100 operations, one connection, one request
```

Non-transactional by default and explicit about it — only Salesforce can offer
true atomicity ([03](03-provider-adapter-contract.md)). The response reports per-item
outcome:

```jsonc
{ "results": [
    { "index": 0, "status": "succeeded", "id": "ccn_gft_…" },
    { "index": 1, "status": "failed",
      "error": { "code": "validation_failed",
                 "message": "splits do not sum to amount",
                 "field": "splits" } }
  ],
  "summary": { "succeeded": 1, "failed": 1 },
  "atomic": false }
```

Request `atomic: true` and the API returns 422 `capability_unsupported` on providers
that cannot honour it, rather than pretending.

### Webhooks (outbound, to our customers)

```
GET    /v1/webhooks
POST   /v1/webhooks          subscribe to canonical events
DELETE /v1/webhooks/{id}
POST   /v1/webhooks/{id}/test
```

Events are canonical, not provider-shaped: `constituent.created`,
`constituent.updated`, `gift.created`, `gift.refunded`, `commitment.status_changed`,
`connection.needs_reconnect`, `sync.failed`. Signed with HMAC-SHA256 over the raw
body plus a timestamp, with a replay window, so customers can verify authenticity.

## Pagination

Cursor-based everywhere. Offset pagination over a table that is being actively
synced skips and duplicates records.

```
GET /v1/gifts?limit=50&cursor=eyJhZnRlciI6…
```

```jsonc
{ "data": [ … ],
  "pagination": { "next_cursor": "eyJhZnRlciI6…", "has_more": true, "limit": 50 } }
```

`limit` default 50, max 200. Cursors are opaque, signed, and expire after 24 hours.
No total count on list endpoints by default — computing it is expensive and almost
never used; `POST /v1/{entity}/search` can return one on request.

## Filtering

```
GET /v1/gifts?gift_date[gte]=2026-01-01&gift_date[lt]=2026-04-01
             &amount[gte]=10000&status=received
             &designation_id=ccn_desg_01HQ…&sort=-gift_date
```

Operators: `eq` (implicit), `ne`, `gt`, `gte`, `lt`, `lte`, `in`, `contains`,
`starts_with`, `is_null`. A filter a provider cannot push down is applied by us
post-fetch where sound, and rejected with `filter_unsupported` where post-filtering
would produce an incorrect page — silently returning a wrong subset is worse than an
error.

## Consistency

```
GET /v1/constituents/{id}?consistency=cached   (default)
GET /v1/constituents/{id}?consistency=live
```

Every response carries freshness metadata:

```jsonc
{ "data": { … },
  "meta": { "consistency": "cached",
            "synced_at": "2026-03-02T17:04:50Z",
            "staleness_seconds": 412 } }
```

`live` costs provider quota and may 429 or 503 under pressure; it is documented as
such, and the throttle gives it top priority ([05](05-sync-engine.md#rate-limit-governance)).

## Error model

One envelope, always.

```jsonc
{ "error": {
    "code": "capability_unsupported",
    "message": "Human-readable, safe to show an end user.",
    "detail": "Longer technical explanation for a developer.",
    "request_id": "req_01HQ…",
    "field": "splits",
    "provider": "blackbaud_renxt",
    "provider_error": { "code": "…", "message": "…" },
    "retryable": false,
    "retry_after_seconds": null,
    "docs": "https://docs.conconnect.example/errors/capability_unsupported" } }
```

| HTTP | Code | Meaning |
|---|---|---|
| 400 | `invalid_request` | Malformed syntax or parameters |
| 401 | `unauthenticated` | Missing/invalid ConConnect credential |
| 403 | `forbidden` | Authenticated but out of scope |
| 404 | `not_found` | No such resource in this tenant |
| 409 | `conflict` | Version conflict; re-read and retry |
| 409 | `duplicate` | Idempotency key reused with a different payload |
| 422 | `validation_failed` | Canonical invariant violated (e.g. splits do not balance) |
| 422 | `capability_unsupported` | Provider cannot do this — see doc 03 |
| 424 | `connection_unavailable` | Connection is `needs_reconnect` or `disabled` |
| 429 | `rate_limited` | Our limit. `Retry-After` set |
| 502 | `provider_error` | Provider returned an unexpected error; `provider_error` populated |
| 503 | `provider_unavailable` | Provider down or circuit open. `Retry-After` set |
| 504 | `provider_timeout` | Provider did not respond in time |

Distinguishing 429 (ours) from 503 with a provider quota reason matters for client
behaviour: the first is a short wait, the second may be an hour.

Provider errors are **always** mapped to a canonical code, with the original
preserved in `provider_error`. A caller must never need a `switch` on Salesforce
error strings to handle a failure — that would defeat the entire purpose of the
service.

## Idempotency

`Idempotency-Key` on every mutating request. Keys are stored 24 hours with their
response. A replay with the same key returns the stored response; the same key with a
different body returns 409 `duplicate`.

For `POST /v1/gifts` the key should be derived from the payment processor's
transaction reference where one exists. That makes double-posting a donation
structurally impossible rather than merely unlikely.

## Headers

| Header | Direction | Purpose |
|---|---|---|
| `Idempotency-Key` | in | Mutation deduplication |
| `X-ConConnect-Connection-Id` | in | Target connection (alternative to the query parameter) |
| `X-ConConnect-On-Unsupported` | in | `fail` \| `drop` \| `custom` — see doc 03 |
| `X-Request-Id` | in/out | Correlation; echoed and logged |
| `X-RateLimit-Limit` / `-Remaining` / `-Reset` | out | Our limits |
| `X-Provider-Quota-Remaining` | out | Best-effort provider budget visibility |

## Rate limiting (ours)

Default 1,000 requests/minute per tenant, burst 100/second, with `live` reads on a
separate tighter bucket because each one consumes provider quota. These are *our*
limits and independent of the provider ceilings in [05](05-sync-engine.md).
