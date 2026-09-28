# ConConnect CRM Integration API — Design Documentation

Read in order. Each document states its own decisions and open questions.

| # | Document | What it settles |
|---|---|---|
| 00 | [Overview & goals](00-overview.md) | What we are building, what we are explicitly not, glossary |
| 01 | [Architecture](01-architecture.md) | Components, request flow, deployment shape |
| 02 | [Canonical data model](02-canonical-data-model.md) | The entities and fields every provider maps onto |
| 03 | [Adapter contract](03-provider-adapter-contract.md) | The interface each CRM implements; capability matrix |
| 04 | [Authentication & tenancy](04-authentication-and-tenancy.md) | Connections, credential storage, multi-tenant isolation |
| 05 | [Sync engine](05-sync-engine.md) | Delta polling, webhooks, idempotency, conflicts, merges |
| 06 | [Public API contract](06-public-api.md) | Endpoints, pagination, error model, versioning |
| 07 | [Field mapping](07-field-mapping.md) | Canonical ↔ provider field tables |
| 08 | [Security & compliance](08-security-and-compliance.md) | PII, encryption, audit, case-management data |
| 09 | [Open questions](09-open-questions.md) | Decisions needed before/during build |
| 10 | [Roadmap](10-roadmap.md) | Phasing and rough sequencing |
| 11 | [Partnership status](11-partnership-status.md) | Blackbaud ISV standing, outstanding obligations, programme contacts |

Provider-specific research and integration notes:

- [Salesforce (NPSP + Nonprofit Cloud)](providers/salesforce.md)
- [Blackbaud Raiser's Edge NXT (SKY API)](providers/blackbaud-renxt.md)
- [Bonterra Apricot](providers/bonterra.md)
- [Adding a new provider](providers/adding-a-provider.md)

Machine-readable contract sketch: [`api/openapi.yaml`](api/openapi.yaml)

Where the vendor documentation lives and what to read first:
[`reference/api-docs-index.md`](reference/api-docs-index.md)

## How to read the confidence markers

These docs were assembled from vendor documentation, vendor community posts, and
third-party integration guides. Network access from the authoring environment was
restricted, so some primary sources could not be read directly.

- Unmarked statements are ones we are confident about.
- **VERIFY** marks a specific claim that should be confirmed against primary
  vendor documentation or a sandbox before code depends on it.
- **ASSUMPTION** marks a design choice we made in the absence of a decision from
  the ConConnect team. Any of these can be overturned cheaply now and expensively
  later.

Sources that could not be reached from the authoring environment and should be
read before implementation begins:

- `developer.blackbaud.com/skyapi` — SKY API reference, throttling, sync guides
- `docs.everyaction.com` — EveryAction / NGP VAN API reference
- `developer.bonterra.network` — Bonterra platform API portal
