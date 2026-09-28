# Provider: Blackbaud Raiser's Edge NXT (SKY API)

**Provider id**: `blackbaud_renxt`

The best-fitting data model of the three — Raiser's Edge was built for fundraising,
so constituents, gifts, splits, soft credits and tributes all exist natively. The
difficulty is not modelling; it is **operational**: a shared rate limit and a
change-detection model with a sharp edge.

## API surface

Base: `https://api.sky.blackbaud.com/` (**VERIFY** exact host and per-product path
prefixes).

Product areas relevant to us:

| Area | Covers |
|---|---|
| **Constituent API** | Constituents and related entities: addresses, phones, emails, online presence, notes, constituent codes, relationships |
| **Gift API** | Gifts and related entities: gift splits, gift fundraisers, soft credits |
| **NXT Data Integration API** | 60+ additional endpoints added specifically to enable deeper third-party integration. **VERIFY** what it offers that the core APIs do not — it may provide materially better bulk and delta semantics, which would change our sync design |
| **Webhook API (beta)** | Event subscriptions, including constituent event types |
| **Lists API** | Access to RE NXT saved lists — potentially useful for scoping a sync to a customer-defined segment |

Investigating the NXT Data Integration API properly is the **highest-value research
task remaining** on this provider. It was purpose-built for our use case and may
remove several of the constraints below.

## Auth

OAuth 2.0 authorization-code, plus a subscription key on every call:

1. Redirect the admin to Blackbaud's authorize endpoint.
2. Exchange the code at the token endpoint, authenticating with **HTTP Basic**:
   `Authorization: Basic base64(client_id:client_secret)`.
3. Every subsequent API call sends **both**:
   - `Authorization: Bearer <access_token>`
   - `Bb-Api-Subscription-Key: <our application subscription key>`

Notes:

- Access tokens last **~60 minutes**; refresh tokens rotate. **VERIFY** refresh-token
  lifetime and whether issuing a new one invalidates the old. If it does, concurrent
  refresh will lock the customer out — refresh must be serialised behind a
  per-connection lock ([04](../04-authentication-and-tenancy.md)).
- The subscription key belongs to **our developer account**, not the customer. It
  identifies the application and is what the rate limit counts against.
- The customer's RE NXT subscription must include the relevant SKY API products. A
  403/401 on a constituent read may mean "not subscribed" rather than "bad token";
  `describeInstance` should distinguish these and report the difference, because the
  remedies are completely different (call the vendor vs. reconnect).

## Rate limits — the defining constraint

> Default **~5,000 requests per hour, per application**. Exceeding it returns
> **429** with a `Retry-After` header in seconds.
> **VERIFY** the current figure, whether it is hourly or per-minute-smoothed, and the
> process for requesting an increase — Blackbaud documents one, and initiating it
> should be an early business action, not an engineering afterthought.

Per-application, not per-customer. With one application serving all tenants:

| Blackbaud tenants | Requests/hour each |
|---|---|
| 5 | 1,000 |
| 25 | 200 |
| 100 | 50 |

Fifty requests per hour cannot sync a donor database. This drives:

1. A **global distributed token bucket** shared by every process
   ([05](../05-sync-engine.md#rate-limit-governance)).
2. **Fair-share scheduling** so a backfill cannot starve incremental syncs.
3. **Aggressive caching** of reference data (funds, campaigns, appeals, codes) —
   these change rarely and would otherwise dominate call volume.
4. **Batch-shaped reads**: always prefer list endpoints with filters over per-record
   gets. Fetching 500 constituents in one call versus 500 calls is the difference
   between viable and not.
5. A **standing business question**: if the limit cannot be raised enough, we may
   need per-tenant application registration, which changes onboarding substantially.
   Resolve before scale, not after — Q6 in [09](../09-open-questions.md).

Also **VERIFY** whether daily quotas exist in addition to the hourly limit; vendor
material mentions exceeding daily quotas in the context of sync resumption, which
implies they do.

## Change detection — and the trap

Two mechanisms:

### `last_modified` / `sort_token` list polling

`GET` list endpoints accept a `last_modified` parameter to return records changed
since a point in time. Better, they can return a **`sort_token`**: when sorting by
system date fields the result is **guaranteed stably sorted**, so the token can be
stored and reused indefinitely — after a failure, a restart, or a quota exhaustion —
without missing or duplicating records.

**Always prefer `sort_token` over a timestamp watermark.** A timestamp loses records
written in the same second as the cut-off and cannot safely resume mid-page. Persist
the token per `(connection, entity)` and never interpret it.

### ⚠ The `date_modified` trap

> **Child entity changes do not propagate to the parent.** Adding a gift, or changing
> a constituent code, does **not** update the constituent's `date_modified`. Only a
> change to the entity itself updates that entity's timestamp.

A sync that polls constituents and expects to catch their new gifts will **silently
miss every gift**. This is the single most dangerous behaviour documented for this
provider, because it fails quietly and looks healthy.

Required handling:

- Poll **every** entity type on its own cursor: constituents, gifts, gift splits,
  addresses, phones, emails, notes, constituent codes, relationships.
- Never treat a constituent's freshness as covering its children.
- Capability matrix: `changeDetection.childChangesPropagate: false`.
- Reconciliation exists partly to catch what this model still misses.

### Webhook API (beta)

Event subscriptions exist, with documented constituent event types. Treat as a
**latency optimisation, not a source of truth**: it is beta, and webhook delivery is
never guaranteed. Subscribe where available to reduce polling volume (valuable given
the rate limit), coalesce bursts, and keep `sort_token` polling running underneath at
a longer interval.

**VERIFY** which entity types have webhook coverage, and specifically whether **gift**
events are available — gift events would be the highest-value subscription given the
`date_modified` trap.

## Pagination

Default page size 100, maximum 500 (**VERIFY** per endpoint; it likely varies). Always
request the maximum for sync work — page size directly divides rate-limit consumption.

## Mapping notes

Raiser's Edge maps well onto our canonical model:

| Canonical | RE NXT |
|---|---|
| Constituent | constituent (individual/organisation), with address/phone/email children |
| Household | constituent relationships + spouse/household handling (**VERIFY** how RE NXT represents household giving units) |
| Gift | gift |
| Gift splits | gift splits — native, with per-split amount and designation |
| Soft credits | soft credits — native |
| Tribute | tribute records (**VERIFY** endpoint and shape) |
| Designation | fund / campaign / appeal / package — a four-level hierarchy that maps onto our recursive Designation `level` |
| Commitment | recurring gifts and pledges (**VERIFY** how each is represented; mapping fidelity is the main open risk) |
| Activity | actions / interactions (**VERIFY** naming in the API) |
| Constituent codes | `constituent_codes` |

Hazards:

- No upsert-by-external-id and no transactional batch. Idempotency is entirely ours
  to enforce ([05](../05-sync-engine.md#writes-outbox-and-idempotency)).
- Writing a gift with splits and soft credits requires multiple calls. Document the
  compensation path for a partial failure, and report partial failure explicitly
  rather than reporting success.
- **VERIFY** whether deleted records are discoverable. If not, deletes are only
  detectable by reconciliation, and we mark rather than delete.
- Custom fields appear as attributes/custom fields; map into canonical `custom`.

## Sandbox testing

Blackbaud does not offer a free self-service developer sandbox in the way Salesforce
does. **Obtaining a Raiser's Edge NXT test environment is a prerequisite that should
be started immediately** — it has vendor lead time, and without it none of the
**VERIFY** items above can be closed. Until it is available, the adapter should be
developed against recorded fixtures with the explicit understanding that they encode
assumptions, not confirmed behaviour.
