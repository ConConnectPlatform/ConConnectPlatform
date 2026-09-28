# 11 — Partnership & Program Status

Context that changes the engineering plan, gathered from correspondence in the
ConConnect/Untapped Solutions mailbox (September 2026). Individual contact details
stay in email rather than in this repository; roles are named here so the document
is useful without being a personal-data store.

## Blackbaud: we are already an ISV partner

**ConConnect Holdings Corporation has been enrolled in Blackbaud's ISV Program
since March 2026.** This is the single most important fact for the Blackbaud
integration, and it resolves questions the earlier drafts left open.

What it changes:

- **Q10 (do we need partner status?) is answered — we have it.** We are not
  approaching Blackbaud cold.
- **Q6 (can the rate limit be raised?) has a named channel.** The
  per-application limit problem in [05](05-sync-engine.md#rate-limit-governance)
  goes to the partner program, not to a generic support queue.
- **App registration and SKY API subscription keys run through the ISV path**, which
  is the documented route for creating an application.
- We also sit in the **Social Good Startup Program (SGSP)** 2026 cohort, with an
  assigned partner-program contact, program managers, and direct relationships into
  Blackbaud's fundraising and donor-management leadership. Sandbox access and
  technical contacts are a conversation, not a cold request — which removes the
  longest-lead blocker identified in [10](10-roadmap.md).

### ⚠ Two outstanding program obligations

Both arrived by email and neither appears to have been answered. These are
commercial/compliance actions, not engineering tasks, but engineering depends on
them.

**1. ISV Program status enquiry — overdue, and carries a deboarding risk.**

In June 2026 Blackbaud's partner program asked whether ConConnect intends to
*create an application* as part of its ISV engagement, and stated that if building
an application is not in our plans they would **proceed with deboarding us from the
ISV Program**, offering the Referral Program as an alternative.

That enquiry appears unanswered. Given that this repository is the design for
exactly such an application, the answer is plainly yes — and answering it is what
protects the ISV status the Blackbaud integration depends on. The reply should say
we are building a CRM integration application, and ask for app registration, SKY
API subscription keys, sandbox access, and the rate-limit discussion in the same
message.

**2. Updated ISV Additional Terms — two disclosure deadlines.**

Blackbaud updated the ISV Program Additional Terms in July 2026 and requested:

| Due | Deliverable |
|---|---|
| Within 30 days of the July notification (**now overdue**) | A list of any third-party vendor solutions or applications connected to Blackbaud through our platform, with a brief description of each connection. Blackbaud then returns an **Exhibit A – Approved Integrations** document recording the approvals. |
| By **31 December 2026** | A description of all existing integrations connected to Blackbaud APIs: integration name, purpose, and how it uses Blackbaud APIs. |

The notification flagged material changes in the "Other Restrictions", "Blackbaud
AI" technical terms, and "Software Partner Program AI Terms" sections. **The AI
terms matter to us specifically**, because our product applies AI to constituent
and case data. Someone needs to read those sections against what we actually do
with customer data before we ship an integration, not after.

Note the sequencing benefit: the December disclosure asks exactly what
[03](03-provider-adapter-contract.md) and [07](07-field-mapping.md) already
document — integration name, purpose, and which APIs it uses. Written properly,
this repository *is* that disclosure.

## Salesforce and Bonterra: no partnership established

No equivalent programme relationship is evident for either.

- **Salesforce**: self-service developer access is sufficient to start. A formal
  ISV/AppExchange path matters only if we want marketplace listing. Nonprofit
  customers may hold donated licences under Salesforce's nonprofit programme, which
  can affect their API allocation — see
  [`providers/salesforce.md`](providers/salesforce.md#limits).
- **Bonterra**: no relationship, and Apricot's API is licence-gated to Enterprise and
  Pro tiers. A partner conversation is likely faster than documentation archaeology,
  for the reasons in [`providers/bonterra.md`](providers/bonterra.md).

## Recommended actions, in order

1. **Reply to the ISV status enquiry.** Confirm we are building an application;
   request app registration, subscription keys, sandbox access, and the rate-limit
   discussion together. Protects ISV status and unblocks Blackbaud engineering in
   one message.
2. **Send the overdue 30-day integration list.** It is short, and it is late.
3. **Read the AI terms** against our actual data handling. Route to counsel.
4. **Open the sandbox and rate-limit conversations** through the partner contacts we
   already have.
5. **Draft the December API-usage disclosure** from this repository's own documents.
