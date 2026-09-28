# 08 — Security & Compliance

This service holds donor records for multiple nonprofits: names, addresses, giving
history, employment, and sometimes wealth-research notes. A breach is an existential
event for a nonprofit's relationship with its donors, and for ours with our customers.

## Data classification

| Class | Examples | Handling |
|---|---|---|
| **Restricted** | OAuth tokens, API keys, subscription keys, full tax IDs, payment instruments | Secrets manager only. Never in Postgres, logs, error messages, or analytics. Never returned by the API. |
| **Sensitive PII** | Name, address, phone, email, birth date, employment, giving history, wealth notes | Encrypted at rest, tenant-scoped, access-logged. |
| **Special category** | Case-management data (Apricot/ETO): case notes, assessments, service records | **Out of scope for v1.** See below. |
| **Operational** | Sync cursors, job state, metrics, capability matrices | Standard handling. No PII permitted in metric labels or job names. |

### Fields we deliberately do not store

- **Full tax identifiers (EIN/SSN).** The canonical model carries `tax_id_last4` only.
  There is no use case in this service that requires the full value, and holding it
  materially raises breach impact.
- **Payment instruments.** Commitments carry `last4`, expiry, and a processor
  reference — never a card number, never a bank account number. We are not in PCI
  scope and must stay out of it.
- **Donor document bodies.** Attachments are metadata plus a fetch URL. Proxying and
  storing wealth-research documents and gift agreements would expand our exposure for
  little benefit in v1.

## Encryption and access

- **In transit**: TLS 1.2+ everywhere, including to provider APIs. Certificate
  validation is never disabled — if a proxy makes TLS fail, fix the trust store, do
  not bypass verification.
- **At rest**: database and object-storage encryption, plus **application-level
  envelope encryption for sensitive PII columns** with a per-tenant data key. A
  stolen database file should not be a readable donor list.
- **Credentials**: secrets manager, per-tenant paths, rotation supported. The database
  stores a reference, never the secret ([04](04-authentication-and-tenancy.md)).
- **Tenant isolation**: `tenant_id` on every table, PostgreSQL row-level security in
  addition to application-level filters, so a forgotten `WHERE` clause cannot leak
  across tenants.
- **Least privilege for provider credentials**: request the narrowest scopes that work,
  and document why each is needed. A connector that asks for full admin access is both
  a security liability and a sales objection.

## Logging

The rule: **logs are not a PII store.**

- Structured logs carry identifiers (`tenant_id`, `connection_id`, `canonical_id`,
  `provider_id`, `request_id`) — never names, addresses, emails, phone numbers, or gift
  amounts.
- **Redaction at the transport layer**, not the call site: the HTTP client strips
  `Authorization`, `Bb-Api-Subscription-Key`, cookies, and known key prefixes before
  anything reaches a log. Relying on every call site to remember is how credentials end
  up in log aggregation.
- Provider request/response bodies are logged only when explicitly enabled for
  debugging, for a bounded window, with PII scrubbing applied, and with that access
  itself audited.
- Error messages returned to API callers never echo raw provider payloads, which
  routinely contain donor PII.

## Audit log

Append-only, tenant-scoped, and queryable, recording: who or what acted, the action,
the entity, before/after values for changed fields, the connection involved, the
resolved conflict policy where one applied, and the correlation id.

This exists for three reasons, in order of how often they will matter:

1. **"Why did this donor's record change?"** — the single most common support question
   in CRM integration work, and unanswerable without this.
2. Demonstrating to a customer's auditor that access is controlled and traceable.
3. Incident forensics.

Retention: **ASSUMPTION** 7 years for audit entries, aligned with typical nonprofit
financial-record retention. Confirm with counsel (Q7 in [09](09-open-questions.md)).

## Regulatory considerations

**None of this is legal advice; it is a list of things to route to counsel.**

- **GDPR / UK GDPR** — if any customer has EU/UK donors. Brings data-subject access,
  erasure, and portability rights. Erasure is architecturally significant: a request
  must propagate through our store, our audit log (where retention obligations may
  conflict), and be reflected in the provider. Design for it now; retrofitting erasure
  into a sync engine is genuinely hard.
- **CCPA/CPRA** — California donors. Similar shape.
- **PCI DSS** — avoided by never touching card data. Keep it that way; the canonical
  model is designed to make it easy.
- **HIPAA / 42 CFR Part 2 / FERPA** — potentially engaged by case-management data. See
  below.
- **State charitable-solicitation rules** — donor privacy and do-not-solicit handling.
  Another reason `consent.*` uses most-restrictive-wins
  ([05](05-sync-engine.md#conflict-resolution)).
- **Our contractual position** — we are a processor/service provider acting on the
  nonprofit's behalf. That requires a DPA with each customer and sub-processor
  disclosure. Worth having ready before enterprise sales conversations, not during.

### Case-management data: a different regime

Bonterra Apricot and ETO hold **service-delivery records about vulnerable people** —
case notes, assessments, programme participation. Depending on the customer and
programme this can attract HIPAA, 42 CFR Part 2 (substance-use treatment records,
which are stricter than HIPAA), FERPA, or state confidentiality law.

**This is not "another CRM connector."** It requires its own data model, its own
compliance review, likely a BAA, and possibly separate infrastructure. The
recommendation in [`providers/bonterra.md`](providers/bonterra.md) stands: out of scope
for v1, treated as a separate initiative with legal involvement from the start.

Building it as an afterthought inside a donor-data service would be the most serious
mistake available on this project.

## Right to erasure

Because it is architecturally load-bearing rather than a feature to add later:

```
DELETE /v1/constituents/{id}?mode=erase
```

Must handle: our canonical record, the identity map (retaining a tombstone so sync
does not resurrect the record — the most common erasure bug in sync systems), cached
reads, the outbox, raw webhook and export payloads in object storage, and the audit
log (where legal retention may require keeping an entry that records the erasure
itself without the erased content). It must also decide whether to propagate the
deletion to the provider, which is the customer's call and must be configurable.

Every one of those touchpoints is a place a naive implementation leaks the data it was
asked to erase. Design now, implement in phase 2, but do not let phase 1 make it
impossible.

## Security practices

- Dependency scanning and automated updates; a lockfile committed.
- Secret scanning in CI, pre-commit hooks. A leaked provider credential affects every
  tenant on that application.
- Provider webhook signature verification is mandatory and fails closed. An
  unverified webhook is an unauthenticated write path into donor data.
- Rate limiting and abuse detection on our own API.
- Security review before enabling any provider write path for a customer.
- An incident-response runbook that includes **provider credential revocation**, since
  our subscription key or application credential compromises every tenant at once.
- Penetration test before general availability.

## Open questions

Carried to [09](09-open-questions.md): whether we may store provider data at all (Q3),
audit retention (Q7), and whether case-management products are in scope (Q1).
