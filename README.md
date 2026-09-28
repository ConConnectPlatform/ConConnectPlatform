# ConConnect CRM Integration API

One API for your product to read and write donor data, regardless of which CRM
the nonprofit on the other end happens to run.

Today the target platforms are **Salesforce** (both NPSP and Nonprofit Cloud),
**Blackbaud Raiser's Edge NXT** (SKY API), and **Bonterra** (a product family,
not a single API — see below). The architecture assumes more will follow.

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

2. **"Bonterra" is not one CRM.** It is a portfolio — EveryAction/NGP VAN,
   Apricot (Case Management), ETO, and others — with genuinely different APIs,
   different auth, and different commercial gates (Apricot's API requires an
   Enterprise or Pro licence). "We support Bonterra" is a claim we cannot make
   without naming products. See [`docs/providers/bonterra.md`](docs/providers/bonterra.md).

3. **Salesforce nonprofits run one of two incompatible data models.** NPSP bends
   `Opportunity` into a donation; Nonprofit Cloud ships purpose-built
   `GiftTransaction` / `GiftCommitment` objects. The adapter must detect which
   and branch. See [`docs/providers/salesforce.md`](docs/providers/salesforce.md).

## Open questions blocking implementation

See [`docs/09-open-questions.md`](docs/09-open-questions.md). The ones that
change the build materially are Q1 (which Bonterra products), Q2 (sync
direction), and Q4 (language/runtime).
