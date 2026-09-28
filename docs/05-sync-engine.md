# 05 — Sync Engine

## Change detection: three mechanisms, one pipeline

Whatever the provider offers, it normalises into a `ChangeBatch` and flows through
one processing path. Adapters differ; the engine does not.

| Provider | Mechanism | Cursor we persist |
|---|---|---|
| Salesforce | Pub/Sub API + Change Data Capture; `SystemModstamp` polling as fallback | replay id, or modstamp watermark |
| Blackbaud RE NXT | `GET` list endpoints with `last_modified` / `sort_token`; Webhook API (beta) for push | `sort_token` (opaque, resumable) |
| Bonterra EveryAction | asynchronous changed-entity export job | export job id + `dateChangedTo` |
| any (fallback) | full scan | page offset + scan start time |

The cursor is **opaque to the core**. It is persisted verbatim per
`(connection, entity)` and handed back to the adapter. This is what lets three
incompatible mechanisms share one scheduler.

### Blackbaud: the `date_modified` trap

The most consequential single fact uncovered in research:

> Child entity changes do not propagate to the parent. Adding a gift or changing a
> constituent code does **not** update the constituent's `date_modified`.

A sync that polls constituents by `last_modified` and expects to catch new gifts
will silently miss every gift. Consequences, all mandatory:

1. **Poll every entity type independently** — constituents, gifts, addresses,
   phones, emails, notes, constituent codes — each with its own cursor. There is no
   "sync the constituent and its children" shortcut.
2. **Use `sort_token`, not a timestamp**, where offered. Sorting by system date
   fields is documented as stably sorted, and the token can be stored indefinitely
   and reused after a failure or a quota exhaustion without missing records. A
   timestamp watermark, by contrast, loses records written during the same second as
   the cut-off and cannot safely resume mid-page.
3. **Never derive "nothing changed" for a constituent from its own timestamp.**
   Freshness is per entity type.

**VERIFY** which RE NXT list endpoints expose `sort_token` versus `last_modified`
only, and whether the NXT Data Integration API's ~60 additional endpoints offer
better delta semantics for gifts specifically. This is the first thing to check in
a sandbox.

### Salesforce

Pub/Sub API with Change Data Capture is the right default for new work. Practical
notes:

- CDC must be **enabled per object** in the org's setup. `describeInstance` should
  check and report which objects are covered; a connection where CDC is off must
  fall back to modstamp polling rather than appearing healthy and syncing nothing.
- Replay ids expire (a retention window measured in days). On expiry, fall back to
  a modstamp catch-up scan, then resume streaming.
- Event delivery has **daily allocations** separate from API request limits. High-churn
  orgs can exhaust them; monitor and degrade to polling rather than dropping changes.
- CDC does not fire for every kind of change (some bulk and metadata operations are
  excluded). A low-frequency reconciliation scan is not optional; see below.

### EveryAction

Delta is a **job, not a query**: `POST /changedEntityExportJobs` with a
`dateChangedFrom`, poll `GET /changedEntityExportJobs/{id}` until complete, then
download one or more **files**. Implications:

- The worker must handle file download and parsing, not JSON page iteration.
- `dateChangedTo` is chosen by the server at execution. Persist the server's value
  as the next `dateChangedFrom` — never a locally computed clock value, or clock
  skew will open a gap.
- Job latency means this is a batch mechanism. Near-real-time is not available;
  set customer expectations accordingly (**VERIFY** typical completion times and
  any per-day job limits).
- Raw export files go to object storage for replay and debugging.

### Deletes

Hard deletes are the most commonly missed change type. Per provider:

- **Salesforce**: CDC emits delete events; also `queryAll` / the Recycle Bin can be
  consulted for a reconciliation pass.
- **Blackbaud**: **VERIFY** whether list endpoints expose deleted records or a
  deleted-since query. If not, deletes are only detectable by reconciliation.
- **EveryAction**: **VERIFY** whether changed-entity exports include deletions.

Where deletes are not detectable, we do **not** silently keep phantom records. We
mark them `stale_unconfirmed` after a reconciliation pass fails to see them, and
surface that state rather than deleting on a guess — deleting a donor record because
of a sync inference is unacceptable.

### Reconciliation

A scheduled full (or partitioned) scan per connection, independent of delta sync:
weekly by default, off-peak, rate-limit-aware. It exists because every delta
mechanism above has a documented or suspected hole. It reports drift as a metric
(`sync.drift.records`) — sustained non-zero drift is a bug, and without this scan we
would never know.

## Identity and linking

```
identity_map
  id, tenant_id, connection_id, provider, provider_object, provider_id,
  canonical_entity, canonical_id,
  provider_version,          -- etag / SystemModstamp / row version for concurrency
  first_seen_at, last_synced_at,
  link_method,               -- provider_id | external_id | deterministic_match | manual
  UNIQUE (connection_id, provider_object, provider_id)
  UNIQUE (connection_id, canonical_entity, canonical_id)
```

Linking rules, deliberately conservative:

1. **Within one connection**, `provider_id` is authoritative. No matching heuristics
   needed or wanted.
2. **Across connections** (same person in Raiser's Edge and EveryAction), automatic
   linking happens **only** on a strong deterministic signal: an exact match on a
   verified email address, or a shared external id that we ourselves wrote. Name and
   address similarity is **not** sufficient and is not used for automatic linking.
3. Everything weaker becomes a **suggested link** surfaced through the API for a
   human to confirm. Wrongly merging two donors' giving histories is materially
   worse than leaving two records unlinked, and is painful to unwind.
4. Merges performed *inside* a provider (both Salesforce and Raiser's Edge support
   constituent merges) arrive as changes we must honour: the losing provider id
   becomes an alias pointing at the surviving canonical record. Never delete the
   alias — inbound references and webhooks will keep using the old id.

## Writes: outbox and idempotency

```
outbox
  id, tenant_id, connection_id, idempotency_key UNIQUE,
  operation,                 -- create | update | delete
  canonical_entity, canonical_id,
  payload jsonb,
  status,                    -- pending | in_flight | succeeded | failed | dead_letter
  attempts, next_attempt_at,
  provider_request_id, provider_response jsonb,
  error_code, error_detail,
  created_at, completed_at
```

- Every write is enqueued with an **idempotency key** — caller-supplied via
  `Idempotency-Key`, or derived deterministically from the canonical payload. A
  replay returns the original result rather than creating a second gift. Double-posting
  a donation is among the worst bugs this system could have.
- **Where the provider supports upsert by external id (Salesforce), use it.** That
  makes idempotency the provider's problem too, which is strictly safer than ours
  alone.
- Where it does not (Blackbaud, EveryAction), we guard with the idempotency key plus
  a pre-write existence check keyed on our own stored reference. Document the
  residual race honestly: a crash between provider-write and our-commit can produce
  a duplicate, which reconciliation detects and reports for human resolution rather
  than auto-deleting.
- **Retries**: exponential backoff with jitter, honouring `Retry-After`. Retry only
  idempotent-safe failures — timeouts, 429, 5xx, and connection errors. Never retry a
  validation failure (4xx other than 429), which will fail identically forever.
- **Dead letter after N attempts** (default 8, ~6 hours). Dead-lettered writes alert
  and are visible via the API. They are never discarded.

### Ordering

Writes for the same canonical record must apply in order. Partition the outbox drain
by `(connection_id, canonical_id)` — parallel across records, strictly serial within
one. Out-of-order application produces a record that reflects an older state and then
stays wrong until the next inbound sync overwrites it.

## Conflict resolution

The same record changed on both sides between syncs.

**ASSUMPTION** (needs confirmation — Q5 in [09](09-open-questions.md)): default
policy is **provider-wins, field-level**.

Rationale: the CRM is the nonprofit's system of record and the place their staff
work. A ConConnect write that silently overwrites what a gift officer typed this
morning destroys trust faster than any other failure mode.

- **Field-level, not record-level.** Two sides editing different fields of the same
  constituent is a merge, not a conflict. Record-level last-writer-wins would discard
  a valid concurrent edit.
- Where the provider supports optimistic concurrency (`If-Match`, version fields),
  use it: a conflicting write returns 409 and is re-queued after re-reading, rather
  than clobbering.
- Configurable per connection and per field group, because the right answer differs:
  consent flags should arguably be **most-restrictive-wins** rather than
  provider-wins — if either side says do-not-contact, the answer is do-not-contact,
  regardless of which write is newer. This is the one place we deliberately deviate
  from "provider-wins" and it should be the default for `consent.*`.
- Every resolution is recorded in the audit log with both values. "Why did this field
  change?" must always be answerable.

## Rate limit governance

The Blackbaud finding, restated because it drives real architecture:

> The SKY API default rate limit is approximately **5,000 requests per hour per
> application** — not per customer. **VERIFY** the current figure and whether it can
> be raised; Blackbaud documents a process for requesting an increase, and doing so
> should be an early business action.

With one application key serving every tenant:

| Tenants | Requests/hour each (5,000 shared) |
|---|---|
| 10 | 500 |
| 50 | 100 |
| 200 | 25 |

Twenty-five requests per hour will not complete an initial import, let alone keep a
donor database in sync. Therefore:

1. **The throttle is global per provider application**, implemented as a distributed
   token bucket in Redis. Every worker and every API process draws from the same
   bucket. A per-process or per-tenant limiter does not solve this problem.
2. **Fair-share scheduling between tenants.** Weighted round-robin so one tenant's
   50,000-record initial import cannot starve everyone else's incremental sync. A
   large backfill is explicitly a low-priority, long-running, interruptible job.
3. **Priority classes**, highest first: interactive/live passthrough reads →
   outbound writes → webhook-triggered pulls → scheduled delta → backfill →
   reconciliation. Under pressure, the bottom of that list waits.
4. **Provider quota is a first-class capacity metric**, alarmed on, and shown on the
   operational dashboard. Approaching the ceiling is a scaling event that requires a
   business action (request a limit increase, or move to per-tenant applications),
   not an engineering one.
5. **Sizing the customer base against the provider ceiling is a commercial
   constraint.** If Blackbaud's limit cannot be raised, per-tenant application
   registration may become necessary — a very different onboarding story, and one
   worth establishing the answer to before we have 50 Blackbaud customers rather than
   after. See Q6 in [09](09-open-questions.md).

Salesforce differs in kind: its limits are **per customer org** (a base daily
allocation plus a per-licence increment, pooled across REST, Bulk and Connect APIs).
So Salesforce needs per-connection accounting and a courtesy reserve — we must not
consume the org's entire daily API budget and break the nonprofit's other
integrations. Reserve a configurable share (default: use no more than 25% of the
org's daily allocation) and alert the customer before we approach it.

## Scheduling defaults

| Job | Default cadence | Notes |
|---|---|---|
| Delta pull (per entity) | 15 min | Floor set by `capabilities.changeDetection.minPollIntervalSeconds` |
| Webhook-triggered pull | immediate | Coalesced: many events in a burst → one pull |
| Outbox drain | continuous | Backoff on provider errors |
| Token refresh | 75% of lifetime | Serialised per connection |
| Health check | hourly | Transitions `needs_reconnect`, notifies |
| Reconciliation | weekly, off-peak | Partitioned; interruptible |
| Initial backfill | on connect | Lowest priority, resumable, progress-reported |

Bursty webhook traffic is coalesced rather than processed one-for-one: a bulk update
in the CRM can emit thousands of events, and one pull per event would exhaust the
rate limit immediately.

## Observability

Per connection: records synced by entity, delta lag (now − provider watermark),
cursor position, error rate by canonical error code, provider quota consumed,
outbox depth and age of oldest pending, dead-letter count, drift from
reconciliation, conflict resolutions applied.

**Alert on:** `needs_reconnect`, dead-letter count > 0, delta lag exceeding 3× the
poll interval, provider quota above 80%, drift sustained above zero across two
reconciliation runs, and any outbox item older than one hour.

Delta lag deserves emphasis: a connection can be "healthy" by every other measure
while quietly falling a week behind because its cursor stopped advancing. Lag is the
metric that catches it.
