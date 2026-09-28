# 09 — Open Questions

Ordered by how much the answer changes the build. Q1–Q4 should be answered before
implementation starts; the rest can be resolved during phase 1.

---

## Q1 — Which Bonterra products do our customers actually use? ⛔ blocking

**Why it matters more than any other question here.** Bonterra is a portfolio, not a
product ([`providers/bonterra.md`](providers/bonterra.md)). The candidates are
different projects, not different configurations:

- **EveryAction / NGP VAN** — fundraising and advocacy. Documented REST API. Maps well
  onto our canonical model. Support-issued keys, so slow onboarding.
- **Apricot / Impact Management** — case management. No gift object. Licence-gated API.
  Different compliance regime. Would require extending the canonical model
  substantially.
- **ETO** — case management, password-based auth, same considerations as Apricot.
- **CyberGrants / Network for Good / Salsa** — unassessed.

Building the wrong one is weeks of wasted work, and "Bonterra support" is a claim we
cannot honestly make to customers without naming products.

**Needed**: the actual product names from the customers or prospects driving this
requirement.

---

## Q2 — Which direction does data move, and is ConConnect ever the source of truth? ⛔ blocking

Three materially different products:

| Option | Build | Risk |
|---|---|---|
| **Read-only ingestion** | Smallest. No outbox, no conflict policy, no write capabilities. Roughly 40% of the design here. | Least useful to customers who want ConConnect activity reflected in their CRM. |
| **Bi-directional** | Everything documented here. | Most work; conflict and duplicate risk are real and need care. |
| **Write-only push** | We push our events into the CRM and never read. | Cannot enrich our product with CRM data. |

The docs currently assume **bi-directional**, which is the superset. If the real answer
is read-only, a great deal drops out and phase 1 lands much sooner.

Related: **what actually needs to sync?** Constituents and gifts only, or the full model
including activities and commitments? Narrowing this is the cheapest available scope
reduction.

---

## Q3 — May we store a copy of CRM data? ⛔ blocking

The architecture ([01](01-architecture.md)) assumes yes, and uses a local store for
fast, quota-free reads. If some customers contractually forbid it:

- cached reads become unavailable for them,
- passthrough-only becomes a supported mode,
- and the provider rate limits in [05](05-sync-engine.md#rate-limit-governance) become
  the binding constraint on every single page view — which, for Blackbaud's shared
  5,000/hour, is likely unworkable.

This is a contractual and sometimes regulatory question, not only technical. It needs a
real answer before we commit to the caching design.

---

## Q4 — Language, runtime, and deployment target ⛔ blocking

The repository is empty, so nothing constrains this. Recommendation: **TypeScript with
NestJS**, because per-CRM adapters map cleanly onto DI modules, OpenAPI generation is
first-class, Salesforce and HubSpot have maintained SDKs, and the same language can be
shared with front-end work.

Alternatives worth considering: **Python/FastAPI** if the team is Python-first or data
work is planned alongside; **C#/.NET** if the existing platform is .NET or we are
deploying to Azure — note Blackbaud publishes a .NET sample sync application, which is
a modest argument in that direction.

Also needed: cloud provider, whether Postgres is managed, and whether the deployment is
containers on Kubernetes, ECS, or a PaaS. The three-deployable shape in
[01](01-architecture.md) holds either way.

---

## Q5 — Conflict resolution default

Currently **ASSUMPTION: provider-wins, field-level**, with `consent.*` on
most-restrictive-wins ([05](05-sync-engine.md#conflict-resolution)).

Confirm this matches how customers expect the integration to behave. The alternative
(ConConnect-wins) risks overwriting what a gift officer typed this morning, which is
the fastest way to lose a customer's trust in a sync.

---

## Q6 — Blackbaud rate limits: can the ceiling be raised, and what is the scaling plan?

The shared per-application limit is the hardest technical constraint in the design.
Needed:

1. Confirmation of the current limit and whether it is hourly or smoothed.
2. Whether daily quotas also apply.
3. What an increase request requires, and what ceiling is achievable.
4. Whether per-tenant application registration is possible — and if so, what it does to
   onboarding.

**This is a business action with lead time, and it should start now.** The answer
determines how many Blackbaud customers we can serve, which is a commercial planning
input, not just an engineering detail.

---

## Q7 — Retention and erasure policy

- How long do we retain synced records after a connection is deleted?
- Audit-log retention — the docs assume 7 years; confirm with counsel.
- Do we support GDPR erasure in v1, or design-for-it-and-defer
  ([08](08-security-and-compliance.md#right-to-erasure))?
- Are EU/UK donors in scope at all? That answer changes the compliance surface
  substantially.

---

## Q8 — Sandbox access ⏱ long lead time

Every **VERIFY** marker in these docs closes with a sandbox. Needed:

- **Salesforce**: easy — Developer Edition orgs. Need one NPSP, one Nonprofit Cloud,
  and one with both installed to exercise ambiguous detection.
- **Blackbaud RE NXT**: no self-service developer sandbox. **Start this request
  immediately** — it has vendor lead time and blocks closing out the Blackbaud
  questions.
- **Bonterra**: depends on Q1, and EveryAction keys come through support with days-to-weeks
  of lead time. **Also start immediately.**

These are the longest-lead items on the project and they are not parallel to
implementation — they gate it.

---

## Q9 — Custom fields and objects

Every one of these CRMs supports arbitrary customisation, and real nonprofit orgs use it
heavily. v1 offers a typed `custom` passthrough. Do we need per-connection custom-field
mapping configuration in v1, or can it wait for phase 3?

Customer-configurable mapping is a significant sub-project (schema discovery, mapping
UI, validation, migration when the customer changes their schema) and should not be
absorbed into phase 1 silently.

---

## Q10 — Commercial and partnership questions

- Do we need formal ISV/partner status with Salesforce, Blackbaud, or Bonterra? This can
  affect rate limits, support quality, marketplace listing, and access to sandboxes.
  Blackbaud and Bonterra both look like cases where a partnership conversation moves
  faster than documentation archaeology.
- Is a customer-facing "which CRMs do you support, and how completely" matrix a
  deliverable? The capability matrix in [03](03-provider-adapter-contract.md) can
  generate one, which is a genuinely honest and differentiating thing to publish.
- Which CRMs beyond these three are on the roadmap? HubSpot, Microsoft Dynamics, DonorPerfect,
  Bloomerang, Neon CRM, Little Green Light and Virtuous all come up in this market. Knowing
  the likely next two would validate whether the adapter abstraction is drawn in the right
  place.
