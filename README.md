# ConConnect CRM Integration API

One API for your product to read and write donor data, regardless of which CRM
the nonprofit on the other end happens to run.

Confirmed target platforms:

| Platform | Product | Domain |
|---|---|---|
| **Salesforce** | NPSP and Nonprofit Cloud (detect at runtime — they are incompatible) | Fundraising |
| **Blackbaud** | Raiser's Edge NXT, via the SKY API | Fundraising |
| **Bonterra** | Apricot (Bonterra Case Management / Impact Management) | **Case management** |

The architecture assumes more will follow.

## Status

**Design phase — documentation only.** No implementation has started. The
purpose of this branch is to settle the contract, the data model, and the
per-provider realities on paper before any code is written.

Start here: [`docs/README.md`](docs/README.md)

## The one-paragraph version

Each CRM gets an **adapter** that translates between its native shape and a
**canonical nonprofit data model** (constituents, gifts, commitments,
designations, activities). Your product code only ever speaks canonical. A
**capability matrix** makes the differences between providers explicit and
machine-readable, so the API can answer "this CRM cannot do that" honestly
instead of silently dropping data. A **sync engine** handles delta polling,
webhooks, idempotent writes, and identity mapping.

## Why this is not just "call the CRM APIs"

Three findings from the research pass that drive most of the design:

1. **Blackbaud's rate limit is per *application*, not per customer.** A default
   of ~5,000 requests/hour is shared across every nonprofit we connect. With 50
   customers that is 100 requests/hour each. This forces a global fair-share
   scheduler, not per-tenant rate limiting. See
   [`docs/providers/blackbaud-renxt.md`](docs/providers/blackbaud-renxt.md).

2. **Apricot is case management, not fundraising.** It has no native gift object, so
   the canonical model needs client / programme / enrolment / service / assessment /
   outcome entities alongside the donor entities. Its API is also licence-gated to
   Apricot Enterprise and Pro. And because it holds service-delivery records about
   justice-involved people, 42 CFR Part 2 and HIPAA are realistically in play — a
   stricter regime than donor data, and the governing constraint on that provider
   rather than a caveat. See [`docs/providers/bonterra.md`](docs/providers/bonterra.md)
   and [`docs/08-security-and-compliance.md`](docs/08-security-and-compliance.md).

3. **Salesforce nonprofits run one of two incompatible data models.** NPSP bends
   `Opportunity` into a donation; Nonprofit Cloud ships purpose-built
   `GiftTransaction` / `GiftCommitment` objects. The adapter must detect which
   and branch. See [`docs/providers/salesforce.md`](docs/providers/salesforce.md).

## We are already a Blackbaud ISV partner

ConConnect Holdings Corporation has been enrolled in Blackbaud's ISV Program since
March 2026, and sits in the Social Good Startup Program cohort. That gives us a named
channel for app registration, SKY API subscription keys, sandbox access and the
rate-limit conversation above.

It also carries **two outstanding obligations — one overdue, one with a deboarding
risk attached.** See [`docs/11-partnership-status.md`](docs/11-partnership-status.md)
before doing anything else on the Blackbaud integration.

## Open questions blocking implementation

See [`docs/09-open-questions.md`](docs/09-open-questions.md). Q1 (which Bonterra
product) is resolved. The ones still changing the build materially are Q2 (sync
direction), Q3 (may we store CRM data), and Q4 (language/runtime).

## Vendor documentation

[`docs/reference/api-docs-index.md`](docs/reference/api-docs-index.md) indexes what to
read for each platform, in what order. It is an index rather than a local mirror
because this environment's network policy blocks all three vendor developer portals.
