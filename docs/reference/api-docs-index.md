# API Documentation Index

Where the authoritative documentation lives for each platform, and what to read in
what order.

## ⚠ Why these are links and not mirrored copies

Every one of these domains is **blocked by this environment's outbound network
policy** — `developer.salesforce.com`, `developer.blackbaud.com`,
`docs.everyaction.com`, `developer.bonterra.network`, `partnerportal.blackbaud.com`
and `help.blackbaud.com` all fail to connect from here. So this is a curated index
of what to read, not a local copy of the documentation, and URLs are given as
entry points rather than quoted content.

To mirror the docs into this repository, network access needs widening in the
environment settings (cloud environment menu in the title bar → Edit), either to a
broader access level or with these hosts added to the allowed domains. Knowing
*which* documents matter is most of the value; the pages themselves change too
often to be worth vendoring anyway.

Anything below stated as fact rather than as a link carries a **VERIFY** marker in
the provider documents. Do not let an unverified detail reach code.

---

## Salesforce

Current API version **v66.0** as of Spring '26 — Salesforce ships three releases a
year, so confirm and pin explicitly before coding.

### Read in this order

1. **REST API Developer Guide** — `developer.salesforce.com/docs/atlas.en-us.api_rest.meta/api_rest/`
   sObject CRUD, SOQL via `/query`, `queryMore` pagination, `describe` calls.
2. **Composite resources** — in the REST guide. Up to 25 subrequests, transactional.
   This is how a gift with splits and soft credits is written atomically; it is the
   only provider where we get that.
3. **sObject Collections** — up to 200 records per call. The efficient middle path
   between single REST calls and Bulk.
4. **Bulk API 2.0 Developer Guide** — `developer.salesforce.com/docs/atlas.en-us.api_asynch.meta/api_asynch/`
   Initial backfill and large loads. Job-based, far cheaper per record.
5. **Pub/Sub API** — `developer.salesforce.com/docs/platform/pub-sub-api/guide/`
   gRPC. The recommended path for change events; pair with Change Data Capture.
6. **Change Data Capture Developer Guide** — `developer.salesforce.com/docs/atlas.en-us.change_data_capture.meta/change_data_capture/`
   Note CDC is enabled **per object** in org setup, and replay ids expire.
7. **OAuth / Connected Apps** — `help.salesforce.com` and the REST guide.
   Web-server flow with PKCE for admin connect; JWT bearer for unattended sync.
8. **Limits and Allocations Quick Reference** — `developer.salesforce.com/docs/atlas.en-us.salesforce_app_limits_cheatsheet.meta/salesforce_app_limits_cheatsheet/`
   Per-org daily API allocation, pooled across REST/SOAP/Bulk/Connect. Platform-event
   allocations are separate.
9. **Upsert by external id** — `PATCH /sobjects/{Object}/{ExternalIdField}/{value}`.
   Read this carefully; it is the cheapest idempotency win available on any provider
   here.

### The nonprofit data model — read both, because customers run one or the other

- **Nonprofit Cloud Developer Guide** — `developer.salesforce.com/docs/atlas.en-us.nonprofit_cloud.meta/nonprofit_cloud/`
  The current model. Start with the Fundraising API objects, especially
  `GiftTransaction` and `GiftCommitment`.
- **NPSP documentation** — the Salesforce.org / Nonprofit Success Pack developer
  material and the package's object reference. Frozen since ~2023 but widely
  deployed. Critical reading: how NPSP **automation** (rollups, household naming,
  payment scheduling) reacts to direct writes.

See [`../providers/salesforce.md`](../providers/salesforce.md) for why detecting
which model an org runs is mandatory and what breaks if we guess.

---

## Blackbaud Raiser's Edge NXT (SKY API)

Access runs through our existing ISV Program enrolment — see
[`../11-partnership-status.md`](../11-partnership-status.md).

### Read in this order

1. **Getting started** — `developer.blackbaud.com/skyapi/docs/getting-started`
   The page Blackbaud's own partner team pointed us to for creating an application.
   Start here; it is the app-registration path.
2. **Basics** — `developer.blackbaud.com/skyapi/docs/basics`
   OAuth 2.0 authorization-code flow, Basic auth on the token endpoint, and the
   `Bb-Api-Subscription-Key` header required on every call.
3. **Throttling and retry patterns** — `developer.blackbaud.com/skyapi/docs/in-depth-topics/api-request-throttling`
   The rate-limit ceiling, 429 handling, and `Retry-After`. **Read this before
   designing anything**, because the limit is per application rather than per
   customer and that drives our whole scheduler.
4. **Synchronize data** — `developer.blackbaud.com/skyapi/docs/in-depth-topics/synchronize-data`
   `last_modified` and `sort_token` semantics. Also the source of the single most
   dangerous documented behaviour: child entity changes do not update the parent's
   `date_modified`.
5. **Constituent API** — `developer.blackbaud.com/skyapi/products/renxt/constituent`
6. **Gift API** — `developer.blackbaud.com/skyapi/products/renxt/gift`
   Gift splits, gift fundraisers, soft credits.
7. **NXT Data Integration API** — 60+ endpoints added specifically to enable deeper
   third-party integration. **Highest-value unread document on the list** — it may
   materially improve our sync design, and we have not been able to assess it.
8. **Webhook API (beta)** — `developer.blackbaud.com/skyapi/products/renxt/webhook/tutorial`
   and the constituent event types page. Check whether **gift** events exist; they
   would be the most valuable subscription we could hold.
9. **API lists** — `developer.blackbaud.com/skyapi/docs/in-depth-topics/api-lists`
   Access to RE NXT saved lists; potentially useful for scoping a sync to a
   customer-defined segment.

### Reference implementation

- `github.com/blackbaud/skyapi-headless-data-sync` — Blackbaud's own .NET sample
  sync application. Worth reading whatever language we choose: it demonstrates the
  vendor's intended delta-sync pattern, including the 1-minute polling interval and
  `sort_token` handling.

### Partner and programme material

- Partner portal — `partnerportal.blackbaud.com` (agreement versions, service
  listings, notification settings)
- ISV Program Additional Terms — linked from the July 2026 partner notification.
  Read "Other Restrictions", "Blackbaud AI" technical terms, and "Software Partner
  Program AI Terms".

---

## Bonterra Apricot (Bonterra Case Management / Impact Management)

**Confirmed as the Bonterra product in scope.** Note that this is case management,
not fundraising — see [`../providers/bonterra.md`](../providers/bonterra.md) for
what that means for the data model and the compliance posture.

### Where to look

1. **Bonterra developer portal** — `developer.bonterra.network`
   Search summaries indicate authentication, OAuth 2.0, user management and core
   platform services. **We have not been able to read it.** Whether it fronts
   Apricot data or is a separate platform-administration surface is the largest
   single unknown in this project. Check first.
2. **Account portal** — `account.bonterra.network`
3. **Apricot API integration FAQ** — Bonterra's Apricot help centre, `intercom.help/Bonterra-Apricot`.
   Covers authentication, available endpoints, permissions and troubleshooting.
   Also the source of the **licence gate**: API integration is available to Apricot
   **Enterprise and Pro** customers only.
4. **Bonterra Central Community** — `community.bonterratech.com`, particularly the
   API keys and integrations sections.
5. **ETO API** — `intercom.help/bonterra-eto`, if any customer runs ETO rather than
   Apricot. Authenticates via `POST /API/Security.svc/SSOAuthenticate/` with a user
   email and password, and requires the API feature enabled on the site.

### Also relevant if a fundraising product enters scope

- **EveryAction / NGP VAN** — `docs.everyaction.com`, base URL
  `https://api.securevan.com/v4/`. Includes `/changedEntityExportJobs` for delta
  sync. Not currently in scope, but it is the Bonterra product that maps onto the
  donor-data model in [02](../02-canonical-data-model.md), so worth knowing exists.

---

## Practical note on reading these

The three platforms use different vocabulary for the same ideas. Keep
[07](../07-field-mapping.md) open while reading, and record every field you confirm —
closing a **VERIFY** marker while you happen to have the documentation in front of
you is far cheaper than rediscovering it later.

When a document contradicts something in this repository, the document wins, and the
repository should be corrected in the same sitting.
