# Provider: Salesforce

**Provider ids**: `salesforce_npc` (Nonprofit Cloud), `salesforce_npsp` (Nonprofit
Success Pack). One adapter, two data-model branches selected by `describeInstance`.

## The central problem: two incompatible nonprofit data models

A nonprofit Salesforce org is running one of two fundamentally different schemas,
and they are not variations on a theme.

### NPSP (Nonprofit Success Pack) — the installed-package legacy

A managed package layered on standard sales objects:

| Concept | Object |
|---|---|
| Donation | `Opportunity` (with record types) |
| Payment / installment | `npe01__OppPayment__c` |
| Recurring donation | `npe03__Recurring_Donation__c` |
| Household | `Account` with `npe01__SYSTEM_AccountType__c` = Household |
| Relationship | `npe4__Relationship__c` |
| Soft credit | `OpportunityContactRole` (+ NPSP partial-credit fields) |
| Rollups | `npo02__*` fields on Contact/Account, maintained by package automation |

NPSP has had **no new features since roughly March 2023** and new nonprofit orgs are
routed to Nonprofit Cloud by default (**VERIFY** current vendor position). It remains
very widely deployed, so supporting it is not optional.

**The critical constraint: NPSP is automation, not just schema.** Rollups,
household naming, payment scheduling and recurring-gift generation are implemented
as package triggers and scheduled jobs. Writing directly to `Opportunity` or
`npe01__OppPayment__c` without respecting that automation produces records that look
right and report wrong — mismatched rollups, orphaned payments, households that never
get renamed.

Consequences for the adapter:

- **Never write NPSP rollup fields** (`npo02__*`). They are derived. Mark them
  `derivedFields` in the capability matrix.
- Create donations as `Opportunity` and let NPSP generate the payment, rather than
  creating both. **VERIFY** exact behaviour against the package version in a sandbox
  — this differs by NPSP settings.
- A pledge is an `Opportunity` with a future close date and multiple
  `npe01__OppPayment__c` children. Our Commitment maps to that pattern, not to a
  single object — a lossy mapping in both directions that must be documented for
  customers.
- Detect the installed package **version**; behaviour differs across versions.

### Nonprofit Cloud (NPC) — the current purpose-built model

Standard objects designed for fundraising, 30+ of them, with the key ones:

| Concept | Object |
|---|---|
| Gift received | `GiftTransaction` |
| Pledge / recurring / grant commitment | `GiftCommitment` |
| Designation | `GiftDesignation` (**VERIFY** exact name) |
| Gift allocation / split | `GiftTransactionDesignation` (**VERIFY**) |
| Soft credit | `GiftSoftCredit` (**VERIFY**) |
| Person | `Contact` or Person Account depending on org config |

This model maps onto our canonical model **almost directly**, which is not a
coincidence — our Gift/Commitment split was chosen partly because NPC validates that
separation as the correct nonprofit modelling. `GiftCommitment` status moves through
Draft → Active → Paused → Failing → Lapsed → Closed (**VERIFY** exact values), which
is why our Commitment carries a `failing` status.

`Opportunity` still exists in NPC but is used for major-gift and grant *pipeline*
tracking rather than as the donation record itself. Mapping Opportunity → our Gift on
an NPC org would double-count revenue.

### Detection

`describeInstance` must determine which model applies, and it must be robust — the
cost of getting it wrong is writing donations into the wrong schema:

1. Call `/services/data/vXX.0/sobjects/` and look for `GiftTransaction` (NPC) and
   `npe01__OppPayment__c` (NPSP).
2. Check the installed-packages list for the NPSP package and its version.
3. **Handle both present.** Mid-migration orgs exist. Do not guess: set the
   connection to a `needs_configuration` state and require an explicit choice of
   which model is authoritative. Silently picking one would write gifts into a
   schema the customer's reports do not read.
4. Detect Person Accounts, which changes Contact handling throughout.
5. Cache the result on the connection; re-run on a schedule, since orgs migrate.

## API surface

- **Version**: v66.0 as of Spring '26 (**VERIFY** at implementation time; Salesforce
  ships three releases a year). Pin a version explicitly — never call an unversioned
  endpoint — and schedule an upgrade review each release.
- **REST** `/services/data/vXX.0/sobjects/{Object}/{id}` for CRUD.
- **SOQL** via `/query` for reads, with `queryMore` for pagination.
- **Composite** `/composite` for transactional multi-object writes, up to 25
  subrequests. This is how a Gift with splits and soft credits is written atomically —
  the only provider where we can offer that.
- **sObject Collections** for up to 200 records per call; the efficient path for
  medium batches.
- **Bulk API 2.0** for initial backfill and large loads. Async, job-based, and far
  cheaper in API-call terms than iterating REST.
- **Pub/Sub API** (gRPC) with Change Data Capture for change events — the
  recommended path for new event-driven integrations.
- **`external_id` upsert**: `PATCH /sobjects/{Object}/{ExternalIdField}/{value}`.
  This is a significant advantage — it makes writes genuinely idempotent at the
  provider, which no other provider here offers. Create a ConConnect external-id
  field on every synced object at connect time and use it for every write.

## Auth

OAuth 2.0: web-server flow with PKCE for admin-initiated connect; JWT bearer for
unattended sync. Store the returned `instance_url` per connection — hardcoding
`login.salesforce.com` breaks sandboxes and My Domain orgs. See
[04](../04-authentication-and-tenancy.md).

## Limits

- **Per-org daily API request allocation**: a base allowance (on the order of 100,000
  calls/24h) plus a per-licence increment (Enterprise +1,000/user; Unlimited
  +5,000/user), **pooled across REST, SOAP, Bulk and Connect APIs**. **VERIFY**
  current figures and nonprofit-licence specifics — donated Salesforce licences under
  the Power of Us programme may allocate differently.
- Because the budget is the **customer's**, and shared with their other integrations,
  we must not consume it all. Reserve a configurable share (default: ≤25% of daily
  allocation), track consumption per connection, and alert the customer as we
  approach it.
- Platform-event delivery has its own daily allocation, separate from API calls.
- Concurrent long-running request limits apply; keep SOQL selective and avoid
  unbounded queries.

## Change detection

Pub/Sub + CDC preferred. Notes in [05](../05-sync-engine.md#salesforce): CDC is
enabled **per object** in org setup, replay ids expire, and CDC does not cover every
change type — so modstamp polling is the fallback and reconciliation is still
required.

`SystemModstamp` (not `LastModifiedDate`) is the correct watermark field: it updates
on system-level changes that `LastModifiedDate` misses.

## Mapping hazards

| Hazard | Handling |
|---|---|
| NPSP rollup fields are derived | Never write; expose read-only |
| NPSP household naming is automated | Never write household `Name` directly |
| Record types gate `Opportunity` semantics | Read record-type metadata per org; do not assume |
| Currency: multi-currency orgs carry `CurrencyIsoCode` and dated conversion rates | Always read/write explicit currency; never assume org default |
| Person Accounts change Contact/Account handling fundamentally | Detect and branch |
| Field-level security can hide fields from our integration user | `describeInstance` must check field accessibility and degrade capabilities rather than failing writes at runtime |
| Required custom fields and validation rules vary per org | Surface provider validation errors verbatim in `provider_error`; do not attempt to guess around them |
| Duplicate rules and matching rules may silently block or merge writes | Detect and report; do not retry blindly |

Field-level security deserves emphasis: a capability matrix computed from the schema
alone will claim we can write a field that our integration user cannot see. It must
be computed from *effective* permissions for the connected user.

## Sandbox testing

Salesforce is the easiest of the three to test properly — free Developer Edition orgs
and the Power of Us programme give access to both NPSP and NPC configurations.
Establish, before writing adapter code: one NPSP sandbox, one NPC sandbox, and one
org with both installed to exercise the ambiguous-detection path.
