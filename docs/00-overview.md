# 00 — Overview & Goals

## The problem

ConConnect needs to exchange donor and constituent data with whatever CRM each
nonprofit customer already runs. Those CRMs do not agree on:

- **What a donor is.** A Salesforce NPSP org models a household as an `Account`
  with `Contact` children. Raiser's Edge NXT has first-class constituents with
  relationships. EveryAction has people with a committee/campaign scoping model.
- **What a donation is.** NPSP overloads `Opportunity`. Nonprofit Cloud has
  `GiftTransaction`. Raiser's Edge has gifts with splits, soft credits, and
  tribute records.
- **How you authenticate.** OAuth 2.0 authorization-code, OAuth 2.0 JWT bearer,
  HTTP Basic with a support-issued API key, and a username/password SOAP-ish
  login all appear across the three vendors.
- **How you learn something changed.** Salesforce has Pub/Sub + Change Data
  Capture. Blackbaud has `last_modified`/`sort_token` list polling plus a beta
  webhook API. EveryAction has an asynchronous changed-entity export job.

Writing this integration once per customer, or once per CRM, inside product code
means every new feature multiplies across providers. The point of this service is
to pay that cost once, in one place, behind one contract.

## Goals

1. **One contract.** Product code calls ConConnect endpoints with canonical
   shapes. It never imports a Salesforce SDK or learns what `npe01__OppPayment__c`
   is.
2. **Honest capabilities.** Where a provider genuinely cannot represent something,
   the API says so explicitly (machine-readable) rather than silently discarding
   data. Silent data loss in a donor database is the worst failure mode available
   to us.
3. **Additive provider support.** Adding CRM #4 is a new adapter plus mapping
   config, not a change to the core or to product code.
4. **Auditable.** For any canonical record we can answer: which provider did this
   come from, which provider record, when did we last see it, and what did we
   last write.
5. **Safe under provider limits.** No customer's sync can exhaust a shared API
   budget and starve every other customer.

## Non-goals (for the first release)

- **Not a CRM.** We do not become the system of record. The nonprofit's CRM stays
  authoritative for the data it owns.
- **Not a reporting warehouse.** We store enough to sync, map identities, and
  serve reads. Analytics is a downstream concern.
- **Not a marketing/email engine.** No campaign sending, list building, or
  segmentation beyond what is needed to represent CRM records.
- **Not a data-quality product.** We will not build deduplication, address
  standardisation, or NCOA. We will pass through and respect the provider's own
  matching where it exists (EveryAction's person-match, for example).
- **No custom-object support in v1.** Every one of these CRMs allows arbitrary
  custom fields and objects. v1 covers the standard model plus a typed
  passthrough bag; full custom-schema discovery is phase 3.

## Glossary

| Term | Meaning here |
|---|---|
| **Tenant** | A ConConnect customer (one nonprofit organisation). |
| **Connection** | One authenticated link from one tenant to one CRM instance. A tenant may have several (e.g. Raiser's Edge for gifts, EveryAction for advocacy). |
| **Provider** | A CRM platform we support (`salesforce`, `blackbaud_renxt`, `bonterra_everyaction`, …). |
| **Adapter** | The code implementing our interface for one provider. |
| **Canonical** | Our provider-neutral representation. See [02](02-canonical-data-model.md). |
| **Constituent** | A person known to the nonprofit: donor, member, volunteer, client, prospect. |
| **Gift** | Money actually received. |
| **Commitment** | A promise of future money: pledge, recurring gift, awarded grant. |
| **Designation** | Where a gift is directed: fund, campaign, appeal, package. |
| **Soft credit** | Recognition of a gift for someone who is not the legal donor. Nonprofit-specific and routinely lost by naive CRM mappings. |
| **Capability** | A declared yes/no/partial answer about whether a provider supports an operation or field. |

## Why nonprofit-specific matters

A generic "CRM integration" built around contacts/companies/deals will lose:

- **Soft credits and tribute gifts** — a gift can be legally from a donor-advised
  fund while being credited to the advisor, and given *in honour of* a third
  person. Three constituents, one gift.
- **Household vs. individual giving** — reporting and acknowledgement differ, and
  each CRM models the household differently.
- **Split gifts** — one payment divided across several designations, each with its
  own amount.
- **Pledges vs. payments** — a $12,000 pledge paid monthly is one commitment and
  twelve gifts; counting it as thirteen donations misstates revenue.
- **Matching gifts** — an employer match is a second gift linked to the first.
- **Anonymity and do-not-contact flags** — legally and ethically load-bearing;
  these must never be dropped on a write.

The canonical model in [02](02-canonical-data-model.md) carries all of these as
first-class concepts precisely because the generic model cannot.
