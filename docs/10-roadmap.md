# 10 — Roadmap

Sequencing, with the dependencies that actually govern it. No dates — those need the
answers from [09](09-open-questions.md) and a team size.

## Phase 0 — Unblock (do this first, mostly not engineering)

The long-lead items. Several have vendor turnaround measured in weeks, so they gate
everything and should start immediately and in parallel.

- [ ] Answer **Q1** (which Bonterra products), **Q2** (sync direction), **Q3** (may we
      store data), **Q4** (stack)
- [ ] **Request a Blackbaud RE NXT sandbox** — long lead time, blocks all Blackbaud
      verification
- [ ] **Request Bonterra/EveryAction API access** — support-issued, days to weeks
- [ ] Provision Salesforce Developer orgs: NPSP, Nonprofit Cloud, and one with both
- [ ] **Open the Blackbaud rate-limit conversation** (Q6) — commercial input to capacity
      planning
- [ ] Read the primary developer portals that were unreachable during authoring:
      `developer.blackbaud.com`, `docs.everyaction.com`, `developer.bonterra.network`
- [ ] Initiate partner/developer-relations contact with Blackbaud and Bonterra
- [ ] Legal: DPA template, retention policy, erasure position (Q7)

Nothing in phase 1 is blocked by phase 0 *finishing* — but adapter verification is, so
starting these late is the most likely cause of schedule slip on this project.

## Phase 1 — Foundation + first provider (Salesforce)

Salesforce first because it is the only provider we can fully verify immediately:
self-service sandboxes, both data models available, best tooling.

- [ ] Repository scaffold, CI, lint, test harness, migrations
- [ ] Canonical model as code, with validation of the invariants in [02](02-canonical-data-model.md)
- [ ] `CrmAdapter` interface and capability framework
- [ ] Provider gateway: **global distributed throttle**, backoff, circuit breaker,
      credential redaction at the transport layer
- [ ] Connections: OAuth flow, secrets-manager storage, serialised refresh, health checks
- [ ] Identity map
- [ ] Public API: connections + constituents + gifts, cursor pagination, canonical error
      model, idempotency
- [ ] Salesforce adapter with **NPSP/NPC detection** and both mapping branches
- [ ] Sync engine: delta pull, outbox, per-record ordering, conflict policy
- [ ] Shared contract test suite + recorded fixtures
- [ ] Observability: delta lag, quota consumption, outbox depth, drift
- [ ] Runbook

**Exit criteria**: a Salesforce NPSP org and a Nonprofit Cloud org both sync
bi-directionally, gifts with splits and soft credits round-trip, the contract suite
passes, and the capability matrix is generated from verified behaviour rather than from
documentation.

## Phase 2 — Blackbaud Raiser's Edge NXT

The interesting one, and the phase that validates the architecture — it is the provider
whose constraints the design was actually shaped around.

- [ ] Adapter: OAuth + subscription key, serialised refresh
- [ ] Per-entity cursors using **`sort_token`** — the `date_modified` trap
      ([`providers/blackbaud-renxt.md`](providers/blackbaud-renxt.md)) is the correctness
      risk of this phase
- [ ] Fair-share scheduling proven under the shared per-application limit
- [ ] Reference-data caching (funds/campaigns/appeals/packages)
- [ ] Four-level designation hierarchy mapping
- [ ] Gift splits, soft credits, tributes
- [ ] Webhook API (beta) as a latency optimisation, polling retained underneath
- [ ] **Investigate the NXT Data Integration API** — may materially improve the sync
      design
- [ ] Close out every **VERIFY** in the Blackbaud docs against the sandbox

**Exit criteria**: two Blackbaud connections sync concurrently without either starving
the other, gifts are reliably detected despite the `date_modified` behaviour, and
measured drift is zero across two reconciliation cycles.

## Phase 3 — Bonterra (scope set by Q1)

Assuming EveryAction:

- [ ] Adapter with Basic auth and the mode indicator
- [ ] **`pending` connection state** for support-issued keys, with the key reference
      recorded
- [ ] Export-job change detection: job submission, polling, file download and parsing,
      server-provided watermark persistence
- [ ] Person-matching via the provider's own matching API
- [ ] Customer-facing onboarding docs that set honest lead-time expectations

**Exit criteria**: a real EveryAction database syncs end to end, and onboarding
documentation has been walked through by someone who was not involved in building it.

## Phase 4 — Hardening and scale

- [ ] Reconciliation at production scale
- [ ] Backfill: resumable, interruptible, progress-reported, using Bulk API where available
- [ ] Batch endpoint with atomic mode where the provider supports it
- [ ] Outbound webhooks to customers
- [ ] Erasure implementation ([08](08-security-and-compliance.md#right-to-erasure))
- [ ] Custom-field mapping configuration (Q9)
- [ ] Published capability matrix for customers
- [ ] Penetration test; security review of all write paths
- [ ] Load test against realistic gift volumes, including a year-end spike

## Phase 5 — Additional providers

The real test of the abstraction. Candidates, pending Q10: HubSpot, Microsoft Dynamics,
DonorPerfect, Bloomerang, Neon CRM, Virtuous, Little Green Light.

**Success measure**: a new provider is a new adapter plus mapping configuration, with no
changes to the core, the canonical model, or the public API. If provider #4 requires core
changes, the abstraction is drawn in the wrong place and fixing that is more valuable
than shipping the connector.

## Cross-cutting, every phase

- Capability matrices generated from verified behaviour, never from documentation
- Every **VERIFY** marker either closed or explicitly carried forward with an owner
- Contract test suite green for every adapter
- Runbook updated as failure modes are discovered
- No provider-specific logic outside its adapter
