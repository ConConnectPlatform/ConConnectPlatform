# 01 — Architecture

## Components

```
                  ConConnect product / customer apps
                                 |
                       (REST + API key / OAuth)
                                 v
          +--------------------------------------------------+
          |                  API Gateway                     |
          |   authn/authz, tenant resolution, rate limiting,  |
          |   request validation, idempotency keys            |
          +--------------------------------------------------+
                                 |
          +--------------------------------------------------+
          |                  Core Service                    |
          |                                                  |
          |  Canonical model  ·  Capability resolver          |
          |  Identity map     ·  Field-mapping engine         |
          |  Validation       ·  Conflict policy              |
          +--------------------------------------------------+
             |                    |                    |
             v                    v                    v
    +----------------+   +----------------+   +------------------+
    | Adapter:       |   | Adapter:       |   | Adapter:         |
    | Salesforce     |   | Blackbaud      |   | Bonterra         |
    | (NPSP | NPC)   |   | RE NXT         |   | (EA | Apricot)   |
    +----------------+   +----------------+   +------------------+
             |                    |                    |
             +--------------------+--------------------+
                                  |
                    +-------------------------------+
                    |   Provider Gateway            |
                    |   global fair-share throttle, |
                    |   retry/backoff, circuit      |
                    |   breaker, response cache     |
                    +-------------------------------+
                                  |
                                  v
                        external CRM APIs

    Asynchronous side:

    +-------------+   +--------------+   +-------------------+
    | Scheduler   |-->| Sync Workers |-->| Outbox / Inbox    |
    | (delta cron)|   | (per conn.)  |   | (durable queues)  |
    +-------------+   +--------------+   +-------------------+
            ^                                     |
            |          +-----------------+        |
            +----------| Webhook Receiver|<-------+
                       | (provider push) |
                       +-----------------+
```

## Responsibilities

### API Gateway
Authenticates the caller, resolves tenant, enforces *our* rate limits (separate
from provider limits), validates request bodies against the canonical schema, and
records idempotency keys for writes.

### Core Service
The only place that knows the canonical model. It:
- resolves which connection(s) a request targets,
- asks the **capability resolver** whether the operation is possible on that
  provider and fails fast with a precise error if not,
- runs the **field-mapping engine** (declarative mappings, see [07](07-field-mapping.md)),
- consults the **identity map** to translate canonical IDs ↔ provider IDs,
- applies **conflict policy** on inbound changes.

### Adapters
One per provider. Stateless. Speak canonical on the inside, native on the outside.
They own: endpoint paths, auth header construction, provider pagination, provider
error taxonomy → canonical error taxonomy, and provider-specific quirks. They do
**not** own: retry policy, throttling, or credential storage — those are shared
infrastructure so that behaviour is uniform and testable.

### Provider Gateway
The single egress point to external APIs, and the answer to Blackbaud's
per-application rate limit. Because that budget is shared across all tenants, the
throttle must be **global per provider application**, with fair-share scheduling
between tenants. Details in [05](05-sync-engine.md#rate-limit-governance).

Also owns: exponential backoff honouring `Retry-After`, circuit breaking per
provider, structured logging of every outbound call, and a short-TTL response
cache for reference data (funds, campaigns, appeals, picklists) which changes
rarely and is requested constantly.

### Sync Engine
Scheduler triggers delta pulls per connection. Workers execute them. The
**inbox** absorbs webhook deliveries and normalises them into the same processing
path as polled changes, so there is one code path for "something changed
upstream" regardless of how we learned it. The **outbox** makes our writes durable
and idempotent.

## Two read paths, deliberately

| Path | When | Behaviour |
|---|---|---|
| **Cached/synced read** | Default for lists, search, reporting | Served from our store, populated by the sync engine. Fast, no provider quota cost, may be seconds-to-minutes stale. Every response carries `synced_at`. |
| **Live passthrough read** | `?consistency=live` on single-record reads | Calls the provider directly. Authoritative, costs provider quota, subject to provider latency and limits. |

This matters because of the rate-limit maths in
[`providers/blackbaud-renxt.md`](providers/blackbaud-renxt.md): a UI that reads
live on every page view will exhaust a shared hourly budget almost immediately.
Default to cached; make live an explicit, documented, opt-in choice.

**ASSUMPTION**: we are permitted to store a copy of CRM data. This is a contractual
and sometimes regulatory question per customer, not only a technical one — see
[08](08-security-and-compliance.md) and Q3 in [09](09-open-questions.md). If some
customers forbid it, the write-through/passthrough-only mode must be a supported
configuration, and cached reads become unavailable for them.

## Data stores

| Store | Holds | Notes |
|---|---|---|
| **PostgreSQL** | canonical records, identity map, connections, mappings, sync state, audit log, outbox | Primary. Row-level tenant scoping on every table. |
| **Redis** | throttle counters, distributed locks, reference-data cache, job coordination | Throttle counters must be shared across all workers, so this cannot be in-process. |
| **Object storage** | large export payloads (EveryAction changed-entity export files), raw webhook bodies for replay | EveryAction's delta mechanism returns files, not JSON pages. |
| **Secrets manager** | OAuth client secrets, subscription keys, per-connection tokens | See [04](04-authentication-and-tenancy.md). Never in Postgres in plaintext. |

## Deployment shape

Three deployables, so that a slow sync cannot degrade interactive API latency:

1. **api** — synchronous HTTP (gateway + core). Scales on request volume.
2. **worker** — sync jobs, outbox drain, webhook processing. Scales on queue depth.
3. **scheduler** — a single leader electing delta-pull jobs. Must not fan out
   duplicate work; use a lock in Redis or a single replica.

Webhook receiver endpoints live in **api** (they are HTTP) but do nothing except
verify the signature, persist to the inbox, and return `202`. Provider webhook
deliveries have tight timeouts and retry aggressively; processing inline is how
you get duplicate storms.

## Failure principles

- **Never lose a write.** Outbox + idempotency key. A provider being down delays a
  write; it does not drop it.
- **Never partially apply a multi-object write silently.** Where a provider offers
  transactional composite calls (Salesforce Composite), use them. Where it does
  not (Blackbaud, EveryAction), document the compensation path and surface partial
  failure explicitly in the response.
- **Prefer stale to wrong.** A cached read labelled with `synced_at` is better than
  a live read that intermittently 429s.
- **Fail loudly on capability gaps.** See [03](03-provider-adapter-contract.md).
