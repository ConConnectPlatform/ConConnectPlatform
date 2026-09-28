# 02 — Canonical Data Model

Provider-neutral. Every adapter maps to and from these shapes. Designed for
nonprofit fundraising and constituent management, not adapted from a sales CRM.

## Conventions

- **IDs**: `ccn_<entity>_<ulid>`, e.g. `ccn_con_01HQ8…`. ULIDs sort by creation
  time, which makes cursor pagination cheap.
- **Money**: always `{ "amount_minor": 2500, "currency": "USD" }`. Integer minor
  units. Never floats — a float cent error in a donor's giving total is a support
  ticket and a trust problem.
- **Dates**: `date` for calendar dates that must not shift (gift date, birth date)
  as `YYYY-MM-DD`; `timestamp` for instants as RFC 3339 UTC. Conflating these is
  how a gift moves to the previous fiscal year for donors in UTC−5.
- **Enums**: closed canonical sets, with `*_raw` alongside preserving the
  provider's original value. Never fail a sync because a provider invented a new
  status.
- **Nullability**: `null` means "not known / not set". Absent means "this provider
  cannot express this field". They are different and both meaningful.
- **`custom`**: a typed passthrough map for provider fields with no canonical home.
  Round-trips unchanged.

Every entity carries:

```jsonc
{
  "id": "ccn_con_01HQ8…",
  "tenant_id": "ten_01HQ…",
  "created_at": "2026-01-14T09:22:11Z",
  "updated_at": "2026-03-02T17:04:50Z",
  "sources": [{
    "connection_id": "conn_01HQ…",
    "provider": "blackbaud_renxt",
    "provider_object": "constituent",
    "provider_id": "280549",
    "synced_at": "2026-03-02T17:04:50Z",
    "provider_updated_at": "2026-03-02T16:58:02Z",
    "version": "W/\"…\""
  }],
  "custom": {}
}
```

`sources` is an array because the same human can exist in two connected systems
(Raiser's Edge *and* EveryAction). See [05](05-sync-engine.md#identity-and-linking).

---

## Constituent

A person. Donor, member, volunteer, prospect, client.

```jsonc
{
  "type": "individual",              // individual | organization | household
  "name": {
    "prefix": "Dr",
    "first": "Amara",
    "middle": "K",
    "last": "Osei",
    "suffix": "PhD",
    "formal_salutation": "Dr Osei",     // how to address in a letter
    "informal_salutation": "Amara",
    "sort_name": "Osei, Amara",
    "display_name": "Dr Amara K Osei"
  },
  "emails":    [{ "id": "…", "address": "a@x.org", "type": "personal", "primary": true, "do_not_email": false, "bounced": false }],
  "phones":    [{ "id": "…", "number": "+15125550101", "type": "mobile", "primary": true, "do_not_call": false, "sms_opt_in": true }],
  "addresses": [{ "id": "…", "type": "home", "primary": true,
                  "line1": "…", "line2": null, "city": "Austin",
                  "state_region": "TX", "postal_code": "78701",
                  "country": "US", "do_not_mail": false,
                  "seasonal_from": null, "seasonal_to": null }],
  "birth_date": "1974-06-02",
  "deceased": false,
  "deceased_date": null,
  "employment": { "employer_name": "Acme", "job_title": "CTO",
                  "employer_constituent_id": "ccn_con_01HR…" },
  "primary_household_id": "ccn_hh_01HQ…",
  "lifecycle_stage": "donor",        // prospect | donor | lapsed | member | volunteer | client
  "constituent_codes": ["Board", "Alumni"],
  "tags": ["gala-2026"],
  "consent": {
    "do_not_contact": false,
    "do_not_solicit": false,
    "anonymous_giving": false,
    "email_opt_in_status": "opted_in",      // opted_in | opted_out | unknown | pending
    "consent_source": "web_form",
    "consent_updated_at": "2026-01-14T09:22:11Z"
  },
  "giving_summary": {                  // derived, read-only, never written back
    "first_gift_date": "2019-11-02",
    "last_gift_date": "2026-02-14",
    "last_gift_amount": { "amount_minor": 25000, "currency": "USD" },
    "lifetime_total":   { "amount_minor": 1450000, "currency": "USD" },
    "largest_gift":     { "amount_minor": 500000, "currency": "USD" },
    "gift_count": 34
  }
}
```

Notes:

- **Salutations are not decoration.** Raiser's Edge and NPSP both carry multiple
  addressee/salutation forms, and nonprofits care intensely about them. Dropping
  them produces letters addressed "Dear Dr Amara K Osei PhD".
- **`giving_summary` is derived.** Providers compute these differently (does a
  pledge count? a soft credit?). We expose ours as read-only and document the
  formula rather than pretending it matches the CRM's own rollup field.
- **`deceased`** gates all outbound contact. It must survive every mapping.

## Organization

Same envelope, `type: "organization"`. Companies, foundations, DAF sponsors,
corporate matching partners.

```jsonc
{
  "type": "organization",
  "legal_name": "Acme Foundation",
  "organization_kind": "foundation",   // company | foundation | government | daf_sponsor | other
  "tax_id_last4": "4821",              // never store the full EIN — see doc 08
  "website": "https://…",
  "is_matching_gift_company": true,
  "contacts": [{ "constituent_id": "ccn_con_…", "role": "program_officer", "primary": true }]
}
```

## Household

A giving unit. Modelled explicitly because every provider models it differently
and reporting depends on it.

```jsonc
{
  "type": "household",
  "name": "The Osei Family",
  "members": [
    { "constituent_id": "ccn_con_01HQ…", "role": "head", "gives_jointly": true },
    { "constituent_id": "ccn_con_01HR…", "role": "spouse", "gives_jointly": true }
  ],
  "primary_address_id": "…"
}
```

## Relationship

Directed, typed edge between two constituents.

```jsonc
{
  "from_constituent_id": "ccn_con_01HQ…",
  "to_constituent_id": "ccn_con_01HR…",
  "relationship_type": "spouse",     // spouse | parent | child | employer | employee |
                                     // board_member | advisor | daf_sponsor | other
  "reciprocal_type": "spouse",
  "is_primary": true,
  "start_date": null,
  "end_date": null
}
```

## Designation

Where money is directed. Providers split this into two, three, or four levels
(fund / campaign / appeal / package). We model one recursive entity with a declared
level, which flattens or nests as the provider requires.

```jsonc
{
  "level": "appeal",                 // fund | campaign | appeal | package
  "code": "AG26-EM3",
  "name": "Annual Giving 2026 — Email 3",
  "parent_id": "ccn_desg_01HQ…",
  "active": true,
  "fiscal_year": "FY2026",
  "start_date": "2025-10-01",
  "end_date": "2026-09-30",
  "goal": { "amount_minor": 50000000, "currency": "USD" },
  "restricted": true,
  "restriction_note": "Scholarships only"
}
```

## Gift

**Money actually received.** Not a pledge, not a promise.

```jsonc
{
  "constituent_id": "ccn_con_01HQ…",       // the legal donor
  "household_id": "ccn_hh_01HQ…",
  "commitment_id": "ccn_cmt_01HQ…",        // set when this is a pledge/recurring installment
  "amount": { "amount_minor": 25000, "currency": "USD" },
  "gift_date": "2026-02-14",               // date of the gift for receipting/fiscal purposes
  "received_date": "2026-02-16",           // date the org actually got it
  "posted_date": null,                     // date it hit the ledger, if tracked
  "gift_type": "cash",                     // cash | check | credit_card | ach | stock |
                                           // in_kind | crypto | daf | grant | other
  "status": "received",                    // received | pending | refunded | written_off | voided
  "channel": "online",                     // online | mail | phone | event | in_person | wire | other
  "splits": [                              // MUST sum to amount
    { "designation_id": "ccn_desg_01HQ…", "amount": { "amount_minor": 20000, "currency": "USD" } },
    { "designation_id": "ccn_desg_01HR…", "amount": { "amount_minor": 5000,  "currency": "USD" } }
  ],
  "soft_credits": [
    { "constituent_id": "ccn_con_01HS…", "amount": { "amount_minor": 25000, "currency": "USD" },
      "credit_type": "daf_advisor" }      // daf_advisor | solicitor | household_member |
                                          // matching_company | other
  ],
  "tribute": {
    "tribute_type": "in_memory_of",        // in_honor_of | in_memory_of
    "honoree_name": "Kwame Osei",
    "honoree_constituent_id": null,
    "notify_constituent_id": null
  },
  "matching_gift": {
    "is_match": false,
    "matched_gift_id": null,
    "matching_company_id": "ccn_con_01HT…",
    "expected_amount": { "amount_minor": 25000, "currency": "USD" }
  },
  "acknowledgement": {
    "status": "not_acknowledged",         // not_acknowledged | queued | acknowledged | do_not_acknowledge
    "acknowledged_date": null,
    "letter_code": "AG-TY-1"
  },
  "receipt": { "status": "receipted", "receipt_number": "R-2026-00841", "receipt_date": "2026-02-17",
               "tax_deductible_amount": { "amount_minor": 25000, "currency": "USD" } },
  "anonymous": false,
  "is_recurring_installment": false,
  "payment_reference": "ch_3Ox…",         // processor/transaction reference for reconciliation
  "batch_reference": "BATCH-0219",
  "note": null
}
```

Invariants the core enforces before any write:

1. `splits[].amount` sums exactly to `amount`. Reject otherwise — a split that does
   not balance corrupts fund accounting.
2. `soft_credits[].amount` may each be up to `amount`; they are recognition, not
   allocation, so they do **not** sum to `amount`.
3. `commitment_id` set ⟹ `is_recurring_installment` or pledge-payment semantics
   apply; the commitment's balance must be recalculated.
4. `tax_deductible_amount` ≤ `amount`.
5. A gift with `anonymous: true` must never be written to a provider field that
   surfaces publicly, and the flag must round-trip.

## Commitment

**A promise of future money.** Pledge, recurring gift, or awarded grant. Modelled
separately from Gift so that "expected" never contaminates "received" — the
mistake NPSP's single-`Opportunity` model makes and Nonprofit Cloud's
`GiftCommitment` fixes.

```jsonc
{
  "constituent_id": "ccn_con_01HQ…",
  "commitment_type": "recurring",       // pledge | recurring | grant | planned_gift
  "status": "active",                   // draft | active | paused | failing | fulfilled | lapsed | cancelled
  "total_amount": { "amount_minor": 1200000, "currency": "USD" },   // null for open-ended recurring
  "installment_amount": { "amount_minor": 10000, "currency": "USD" },
  "frequency": "monthly",               // one_time | weekly | monthly | quarterly | semiannual | annual | irregular
  "start_date": "2026-01-01",
  "end_date": null,
  "next_expected_date": "2026-04-01",
  "designation_splits": [{ "designation_id": "ccn_desg_01HQ…", "percent": 100 }],
  "payment_method": { "kind": "credit_card", "last4": "4242", "expires": "2029-01",
                      "processor": "stripe", "processor_ref": "pm_1Ox…" },
  "balance": {                          // derived from linked gifts
    "committed":  { "amount_minor": 1200000, "currency": "USD" },
    "received":   { "amount_minor": 300000,  "currency": "USD" },
    "outstanding":{ "amount_minor": 900000,  "currency": "USD" },
    "installments_paid": 3,
    "installments_expected": 12
  },
  "write_off": null
}
```

`status: "failing"` exists because recurring-gift card failures are an operational
category of their own — a donor whose card expired is not a lapsed donor, and
nonprofits run specific recovery workflows on exactly this state. Nonprofit Cloud's
`GiftCommitment` carries the same idea (**VERIFY** exact picklist values).

## Activity

Anything that happened with a constituent that is not money.

```jsonc
{
  "constituent_id": "ccn_con_01HQ…",
  "activity_type": "call",             // call | email | meeting | note | task | event_attendance |
                                       // volunteer_shift | advocacy_action | membership | mailing_sent
  "subject": "Stewardship call",
  "body": "Discussed scholarship impact.",
  "occurred_at": "2026-03-01T15:00:00Z",
  "direction": "outbound",             // inbound | outbound | none
  "status": "completed",               // planned | completed | cancelled
  "owner_user_ref": "usr_sf_0057g…",   // provider user, opaque
  "related_gift_id": null,
  "related_designation_id": null,
  "outcome": "positive"
}
```

Deliberately broad. Each provider has its own activity/interaction/contact-report
objects with incompatible type sets; one entity with a canonical type enum plus
`*_raw` is far more maintainable than eight near-duplicate entities.

## Note & Attachment

```jsonc
{ "constituent_id": "…", "note_type": "research", "title": "…", "body": "…",
  "is_confidential": true, "author_user_ref": "…", "noted_at": "2026-02-02T…" }
```

Attachments are metadata + a fetch URL only. We do not proxy or store donor
document bodies in v1 — see [08](08-security-and-compliance.md).

---

## Entity relationship summary

```
Household 1──* Constituent *──* Constituent        (Relationship)
                   │
                   ├──* Gift *──* Designation      (splits, amounts)
                   │      ├──* soft_credits ──> Constituent
                   │      └──? tribute      ──> Constituent
                   ├──* Commitment ──* Gift        (installments)
                   ├──* Activity
                   └──* Note
Designation ──? parent Designation                 (fund > campaign > appeal > package)
```

## What we are explicitly not modelling in v1

Memberships as a first-class object (folded into Activity), events and
registrations, volunteer scheduling, grant proposal pipelines, prospect
research/wealth ratings, case-management records (Apricot/ETO program data — this
is service-delivery data about vulnerable people and carries a different
compliance posture entirely; see [08](08-security-and-compliance.md)).
