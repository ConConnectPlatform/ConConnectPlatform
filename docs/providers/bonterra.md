# Provider: Bonterra Apricot

**Provider id**: `bonterra_apricot`
**Status**: confirmed as the Bonterra product in scope (September 2026), superseding
the earlier open question about which Bonterra product our customers run.

## Read this first: Apricot is case management, not fundraising

Bonterra is a portfolio company assembled from acquisitions — EveryAction/NGP VAN,
Apricot, ETO, CyberGrants, Network for Good, Salsa — with different APIs, different
authentication and different commercial terms. The product we are integrating is
**Apricot**, also sold as **Bonterra Case Management** and **Bonterra Impact
Management**.

Apricot models **clients, programmes, services, forms, assessments and outcomes**.
It does not model donations. There is **no native gift object** to map to the Gift
entity in [02](../02-canonical-data-model.md).

This fits ConConnect's customers, who run reentry and second-chance programmes and
whose core work is service delivery rather than fundraising. It is the right product
to integrate. But it means two things have to change, and neither is a detail:

### 1. The canonical data model needs case-management entities

[02](../02-canonical-data-model.md) was designed around donor data — constituents,
gifts, commitments, designations. Apricot needs entities that document does not yet
have:

| Needed entity | Represents |
|---|---|
| **Client** | A person receiving services. Maps loosely onto Constituent, but the fields that matter are different — intake status, eligibility, case assignment, demographics collected for grant reporting. |
| **Programme** | A service offering the organisation runs. |
| **Enrolment** | A client's participation in a programme, with start/exit dates and exit reason. |
| **Service / Encounter** | A delivered unit of service: a session, a placement, a referral. This is what gets counted for funder reporting. |
| **Assessment** | A structured, form-based evaluation with scored responses, administered repeatedly over time. |
| **Outcome** | A measured result tied to a programme goal — the thing funders actually pay for. |
| **Form / Record definition** | Apricot is form-driven, so its schema is substantially customer-defined. This may be closer to schema discovery than to fixed mapping. |

The Gift/Commitment/Designation half of the canonical model stays for Salesforce and
Blackbaud. Apricot uses the constituent half plus these new entities. That is a
genuine extension of scope, and it should be planned as one rather than absorbed
quietly.

**This reverses the recommendation in the earlier draft**, which proposed keeping
Apricot out of v1 on the grounds that it was a different domain. It *is* a different
domain — but it is the customers' domain, so it belongs in scope. What does not
change is that it needs its own data model and its own compliance review.

### 2. The compliance posture is materially stricter

Apricot holds **service-delivery records about vulnerable people**: case notes,
assessments, programme participation, and demographics.

For ConConnect's customer base — reentry and justice-involved populations — the
realistic exposure includes **42 CFR Part 2** (substance-use disorder treatment
records, stricter than HIPAA and requiring specific consent handling), **HIPAA**
where health services are delivered, and state-level confidentiality and criminal-
justice-record rules. Some of this data is more sensitive than anything in a donor
database, and a breach affects people whose housing, employment and liberty may
depend on that confidentiality.

Concretely, before a single Apricot field is written:

- Counsel reviews which regimes apply across the customer base, and whether a BAA is
  required.
- Decide explicitly whether case-note and assessment **content** is synced at all, or
  only metadata and structured fields. Defaulting to "sync everything" is the wrong
  default here.
- Consent and release-of-information handling must be modelled, not inferred — under
  42 CFR Part 2, redisclosure without specific consent is the violation.
- Data residency, retention, minimum necessary, and audit requirements all get
  stricter than [08](../08-security-and-compliance.md) currently assumes.

See [08](../08-security-and-compliance.md#case-management-data-a-different-regime).
That section was written when Apricot was out of scope and now needs to be the
governing document for this provider rather than a caveat.

---

## ⚠ Commercial gate

**Apricot API access requires an Enterprise or Pro licence.** A customer on a lower
tier cannot be integrated regardless of engineering effort.

The connect flow must detect and explain this plainly — discovering it during an
implementation call is a bad experience for the customer and for us. It is also a
qualification question for sales: knowing a prospect's Apricot tier before promising
an integration avoids a commitment we cannot honour.

Third-party integrations in this ecosystem commonly go through Power Automate,
Workato or Zapier rather than direct API work, which may indicate the direct API
surface is narrower than a modern REST CRM's. **VERIFY.**

## ⚠ Documentation gap

`developer.bonterra.network`, `account.bonterra.network`, the Apricot help centre and
the Bonterra community sites are **all unreachable from this environment's network
policy**. Everything specific below is assembled from search summaries and
third-party integration guides, so treat all of it as **VERIFY** and read the primary
sources before writing code. See
[`../reference/api-docs-index.md`](../reference/api-docs-index.md).

The largest single unknown: `developer.bonterra.network` documents what appears to be
a newer **platform-level API** with OAuth 2.0, user management and core platform
services. Whether it fronts Apricot data or is a separate administration surface
**could not be determined**. If it is a unified modern data API, it changes the
integration plan substantially. **Check this first.**

## What we believe about the API

All **VERIFY**:

- REST-based, with API authentication and endpoint requests — consistent with the
  Zapier connector's description of connecting directly to a Bonterra Impact
  Management site.
- Endpoints exist for client management, case tracking and social-services data.
- Apricot is **form-driven**: records are instances of customer-defined forms. So
  "the schema" is substantially per-tenant, which pushes us toward runtime schema
  discovery (`describeInstance` doing real work) rather than a static field map. This
  is a bigger deal for Apricot than for either other provider and should be resolved
  early — it determines whether Q9 (custom-field mapping) is a v1 requirement here
  rather than a phase-3 nicety.
- Change detection mechanism is **unknown**. If there is no delta query and no
  webhook, sync falls back to full scans, and the scan cost scales with the
  customer's record count. Establish this before promising sync latency.

## Open questions specific to Apricot

1. Does `developer.bonterra.network` expose Apricot data, and under OAuth 2.0?
2. What is the change-detection mechanism, and are deletes detectable?
3. How is the customer-defined form schema exposed — is there a metadata/describe API?
4. What are the rate limits, and are they scoped per application or per tenant?
   (The Blackbaud lesson: ask this early, because the answer is architectural.
   See [05](../05-sync-engine.md#rate-limit-governance).)
5. Which of our customers are on Enterprise or Pro, and which are not?
6. Is write access needed, or is read-only ingestion sufficient for the product? For
   case-management data specifically, read-only is a much easier compliance story and
   worth considering on those grounds alone.

## Recommended sequencing

1. **Establish a Bonterra partner or developer-relations contact.** Given the
   portfolio complexity, the licence gates and the unreadable portals, a
   conversation will be faster and more reliable than documentation archaeology —
   the same approach that turns out to be available to us with Blackbaud
   ([`../11-partnership-status.md`](../11-partnership-status.md)).
2. **Read `developer.bonterra.network`** and close the questions above.
3. **Start the compliance review in parallel**, not afterwards. It has legal lead
   time and it can invalidate design choices.
4. **Design the case-management entity extension** to the canonical model.
5. **Build read-only first.** It delivers value, and it defers the hardest consent
   and redisclosure questions until we understand them properly.
