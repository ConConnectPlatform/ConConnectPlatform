# Provider: Bonterra

**Provider ids**: `bonterra_everyaction`, `bonterra_apricot`, `bonterra_eto`
(more may be needed — see below)

## Read this first: Bonterra is not a CRM

Bonterra is a **portfolio company**, assembled from acquisitions. "Integrate with
Bonterra" is not a single piece of work, and we should not describe it as one to
customers. The products have different APIs, different authentication, different data
models, and different commercial gates.

Known products with relevance to us:

| Product | Also known as | Domain | API status |
|---|---|---|---|
| **EveryAction** | NGP VAN, EA8 | Fundraising, advocacy, organising, supporter engagement | Documented REST API at `api.securevan.com/v4` |
| **Apricot** | Bonterra Case Management, Impact Management | Case management, service delivery, outcomes | API exists; **licence-gated to Enterprise and Pro** |
| **ETO** | Bonterra ETO | Case management (enterprise//government) | REST API with a security/login endpoint |
| **Others** | CyberGrants, Network for Good, Salsa, EveryAction Advocacy | Corporate giving, small-nonprofit fundraising, advocacy | Unassessed |

**Q1 in [09](../09-open-questions.md) is the blocking question: which Bonterra
products do our customers actually use?** Everything below is preparation; the
sequencing depends on that answer. Building the wrong one is weeks of wasted work.

⚠ **Documentation gap.** `developer.bonterra.network`, `docs.everyaction.com` and the
Bonterra community sites were **unreachable from the authoring environment** (network
egress policy). The material below is assembled from search summaries and third-party
integration guides. Treat **every** specific here as **VERIFY**, and read the primary
portals before writing code.

---

## Bonterra EveryAction / NGP VAN

The most likely first target: it is the fundraising product, it has a real public API
reference, and it is what "Bonterra" most often means for a nonprofit doing donor
management.

### API surface

- Base URL: `https://api.securevan.com/v4/` (**VERIFY**)
- REST/JSON. Resource areas include people, contributions, recurring commitments,
  disbursements, events, survey questions, activist codes, and canvass responses.
- People matching is a first-class API concept (`people/find`, `people/findOrCreate`
  — **VERIFY** exact paths). This is valuable: it means we can defer to the provider's
  own matching rather than inventing our own, which is both safer and cheaper.

### Auth — and the onboarding consequence

- **HTTP Basic**, where the username is the **application name** and the password is
  the **API key**, conventionally suffixed with a mode indicator (`|0` / `|1`) that
  selects the voter-file context versus the campaign/CRM context. **VERIFY** the exact
  format and which mode corresponds to donor data — getting this wrong points the
  integration at the wrong database.
- **Keys are issued by vendor support**, not by an OAuth flow: the customer files a
  support request in the EveryAction UI naming the application, and the key is
  delivered to a designated contact via a **one-time link** that expires, alongside a
  **four-digit key reference** used to identify it later.

This is the most important product fact on this page. There is **no self-service
connect**, no OAuth redirect, and no programmatic issuance. Every EveryAction customer
onboarding involves a human, a support ticket, and vendor lead time measured in days
to weeks.

Design consequences:

- The connect flow must support a **`pending` connection state** where the customer has
  filed the request and is waiting, with the four-digit key reference recorded so
  support conversations can be matched up.
- Onboarding documentation must tell customers to start the key request **first**, in
  parallel with everything else.
- The API surfaces the expected lead time in the connect response
  ([06](../06-public-api.md#connections)) so the UI can set honest expectations.
- Keys are long-lived and do not rotate automatically: compromise is more damaging and
  rotation is manual. Provide an explicit "replace key" flow and alert on auth failures.

### Change detection — `changedEntityExportJobs`

Delta is an **asynchronous export job**, not a query:

1. `POST /changedEntityExportJobs` with job parameters including `dateChangedFrom`.
2. Poll `GET /changedEntityExportJobs/{exportJobId}` for status and content.
3. On completion, download **one or more files** containing every record changed
   between `dateChangedFrom` and a **server-generated `dateChangedTo`**.

Implications:

- Persist the **server's** `dateChangedTo` as the next `dateChangedFrom`. Using a
  locally computed timestamp opens a gap the width of your clock skew.
- The worker downloads and parses files; this is not JSON page iteration. Raw files go
  to object storage for replay and debugging.
- This is inherently **batch**. Near-real-time sync is not available through this
  mechanism. **VERIFY** typical job completion time and any per-day job limits, then
  set the poll cadence and customer expectations from real numbers.
- **VERIFY** whether exports include **deletions**, and which entity types are
  supported.

### Mapping notes

| Canonical | EveryAction (**all VERIFY**) |
|---|---|
| Constituent | person |
| Gift | contribution |
| Commitment | recurring commitment |
| Designation | designation / fund / appeal (hierarchy depth unclear) |
| Activity | canvass response, activist code application, event signup |
| Tags | activist codes |

Open mapping questions: split-gift support, soft-credit fidelity, and write coverage
(the capability matrix currently guesses `partial` for writes — that guess must be
replaced with tested fact).

Note also that EveryAction carries **political/electoral** data structures (voter
file, committees, modes). Our canonical model is deliberately nonprofit-fundraising
shaped and does not attempt to represent the voter file. If a customer needs electoral
data, that is a separate scoping conversation, not a mapping exercise.

---

## Bonterra Apricot / Impact Management

### ⚠ Commercial gate

**API access requires an Apricot Enterprise or Pro licence.** A customer on a lower
tier cannot be integrated regardless of engineering effort. The connect flow must
detect this and say so plainly; discovering it during an implementation call is a bad
experience for everyone.

Third-party integrations in this ecosystem commonly go through Power Automate,
Workato or Zapier rather than direct API work, which suggests the direct API surface
may be narrower than EveryAction's. **VERIFY**.

### ⚠ Domain mismatch

Apricot is **case management**, not fundraising. It models clients, programmes,
services, forms and outcomes. There is **no native gift object** to map to our Gift
entity.

This means "supporting Apricot" is not the same project as supporting a donor CRM. It
would require extending the canonical model with client/programme/service/outcome
entities — explicitly out of scope in [02](../02-canonical-data-model.md) and
[00](../00-overview.md).

### ⚠ Compliance posture

Apricot data is **service-delivery data about vulnerable people**: case notes,
assessments, and programme participation. Depending on the customer this can attract
HIPAA, 42 CFR Part 2, FERPA, or state-level confidentiality obligations — a materially
different regime from donor data.

**Do not begin Apricot integration work as though it were another CRM connector.** It
needs its own data model, its own compliance review, and probably its own contractual
terms. See [08](../08-security-and-compliance.md).

Recommendation: unless a specific customer commitment requires it, Apricot should be
**out of scope for v1** and treated as a separate initiative.

---

## Bonterra ETO

- Authentication via a REST security endpoint: `POST /API/Security.svc/SSOAuthenticate/`
  taking a **user email and password**, with the API feature enabled on the site.
  **VERIFY** current mechanism.
- Password-based service-account authentication is a security posture worth pushing
  back on. If this is still the only option, the credential requires the strictest
  handling we have ([08](../08-security-and-compliance.md)) and a dedicated,
  least-privilege service account — never a staff member's own login.
- Same case-management domain and compliance considerations as Apricot.

---

## The Bonterra platform API portal

`developer.bonterra.network` documents what appears to be a newer **platform-level**
API — search summaries mention authentication, OAuth 2.0, user management and core
platform services. There is also an account portal at `account.bonterra.network`.

Whether this portal fronts EveryAction/Apricot data behind a unified modern API, or is
a separate surface for platform administration, **could not be determined** from here.

**If it is a unified data API with OAuth 2.0, it changes the plan substantially** — it
would replace the support-issued-key onboarding that is otherwise EveryAction's worst
property. This is the **first thing to check** once network access permits, and it is
worth a direct conversation with Bonterra's partner/developer relations team rather
than reverse-engineering from public docs.

---

## Recommended sequencing

1. **Answer Q1**: which Bonterra products do our customers run? Nothing else here is
   worth doing before this.
2. **Read the primary portals**, especially `developer.bonterra.network`.
3. **Talk to Bonterra's partner team.** Given the portfolio complexity, the
   support-issued keys, and the licence gates, a partnership conversation will be
   faster and more reliable than documentation archaeology.
4. **Build EveryAction first** if fundraising is the use case — it is the best-documented
   and best-fitting.
5. **Treat Apricot/ETO as a separate initiative** with its own data model and compliance
   review.
