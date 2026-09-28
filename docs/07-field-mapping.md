# 07 — Field Mapping

Canonical ([02](02-canonical-data-model.md)) ↔ provider fields.

**Status: draft.** Provider columns are populated from documentation review, not from
sandbox verification. Cells marked **?** are unknown; cells marked **VERIFY** are
best-guess. This document is only trustworthy once each column has been confirmed
against a live instance, and it must be kept in sync with the capability matrix in
[03](03-provider-adapter-contract.md) — where they disagree, the capability matrix is
what the code enforces, so a mismatch is a bug.

Legend: `↔` read+write · `→` we can write · `←` read only · `✗` unsupported

---

## Constituent (individual)

| Canonical | Salesforce NPC | Salesforce NPSP | Blackbaud RE NXT | Bonterra EveryAction |
|---|---|---|---|---|
| `name.prefix` | `Contact.Salutation` ↔ | `Contact.Salutation` ↔ | `title` ↔ | `salutation` VERIFY |
| `name.first` | `Contact.FirstName` ↔ | `Contact.FirstName` ↔ | `first` ↔ | `firstName` ↔ |
| `name.middle` | `Contact.MiddleName` ↔ | `Contact.MiddleName` ↔ | `middle` ↔ | `middleName` VERIFY |
| `name.last` | `Contact.LastName` ↔ | `Contact.LastName` ↔ | `last` ↔ | `lastName` ↔ |
| `name.suffix` | `Contact.Suffix` ↔ | `Contact.Suffix` ↔ | `suffix` ↔ | `suffix` VERIFY |
| `name.formal_salutation` | VERIFY | `npe01__Formal_Salutation__c`-family VERIFY | `preferred_name`/addressee VERIFY | ? |
| `name.informal_salutation` | VERIFY | NPSP salutation fields VERIFY | salutation VERIFY | ? |
| `name.sort_name` | ← derived | ← derived | ← derived | ? |
| `emails[]` | `ContactPointEmail` VERIFY | `Contact.Email` + `npe01__*` ↔ | `emailaddresses` ↔ | `emails[]` ↔ |
| `phones[]` | `ContactPointPhone` VERIFY | `Contact.Phone`/`MobilePhone` ↔ | `phones` ↔ | `phones[]` ↔ |
| `addresses[]` | `ContactPointAddress` VERIFY | `npsp__Address__c` ↔ | `addresses` ↔ | `addresses[]` ↔ |
| `addresses[].seasonal_*` | ? | `npsp__Seasonal_*__c` ↔ | seasonal address fields ↔ | ? |
| `birth_date` | `Contact.Birthdate` ↔ | `Contact.Birthdate` ↔ | `birthdate` ↔ | `dateOfBirth` VERIFY |
| `deceased` | VERIFY | `npsp__Deceased__c` ↔ | `deceased` ↔ | `isDeceased` VERIFY |
| `employment.employer_name` | `Contact.AccountId`→Account | `npe01__WorkEmail__c`/Account ↔ | relationship of type employer | `employer` VERIFY |
| `employment.job_title` | `Contact.Title` ↔ | `Contact.Title` ↔ | `position` VERIFY | `occupation` VERIFY |
| `primary_household_id` | Person Account / household VERIFY | `Account` (Household record type) ↔ | spouse/household relationship VERIFY | ? |
| `lifecycle_stage` | VERIFY | derived from rollups ← | `constituent` type/codes VERIFY | ? |
| `constituent_codes[]` | VERIFY | `npsp__*` / topics VERIFY | `constituentcodes` ↔ | activist codes ↔ |
| `consent.do_not_contact` | VERIFY | `npsp__Do_Not_Contact__c` ↔ | `inactive`/solicit codes VERIFY | VERIFY |
| `consent.do_not_solicit` | VERIFY | `npo02__*`/solicit-code VERIFY | solicit codes ↔ | VERIFY |
| `consent.anonymous_giving` | VERIFY | `npsp__*` VERIFY | `anonymous` VERIFY | VERIFY |
| `consent.email_opt_in_status` | VERIFY | `HasOptedOutOfEmail` ↔ | email opt-out VERIFY | subscription status ↔ |
| `giving_summary.*` | ← computed by us | ← `npo02__*` rollups (read-only) | ← computed by us | ← computed by us |

⚠ **The consent row is the most important and least verified block in this table.**
`do_not_contact`, `do_not_solicit`, `anonymous_giving` and email opt-out are legally
and ethically load-bearing. Every one of these cells must be confirmed against a live
instance before any write path is enabled, and the conflict policy for them is
most-restrictive-wins, not provider-wins
([05](05-sync-engine.md#conflict-resolution)).

## Gift

| Canonical | Salesforce NPC | Salesforce NPSP | Blackbaud RE NXT | Bonterra EveryAction |
|---|---|---|---|---|
| (object) | `GiftTransaction` | `Opportunity` (+`npe01__OppPayment__c`) | gift | contribution |
| `constituent_id` | `GiftTransaction.DonorId` VERIFY | `Opportunity.npsp__Primary_Contact__c` ↔ | `constituent_id` ↔ | `vanId` ↔ |
| `amount` | `Amount` ↔ | `Opportunity.Amount` ↔ | `amount` ↔ | `amount` ↔ |
| `gift_date` | `TransactionDate` VERIFY | `Opportunity.CloseDate` ↔ | `date` ↔ | `dateReceived` VERIFY |
| `received_date` | VERIFY | `npe01__Payment_Date__c` ↔ | `date_added`/post date VERIFY | VERIFY |
| `gift_type` | `PaymentMethod` VERIFY | `npe01__Payment_Method__c` ↔ | `type` ↔ | `paymentType` VERIFY |
| `status` | `Status` VERIFY | `Opportunity.StageName` ↔ | gift status VERIFY | `status` VERIFY |
| `splits[]` | `GiftTransactionDesignation` VERIFY | ✗ (single designation; multi needs custom) | gift splits ↔ **native** | ? VERIFY |
| `soft_credits[]` | `GiftSoftCredit` VERIFY | `OpportunityContactRole` + partial credit ↔ | soft credits ↔ **native** | VERIFY |
| `tribute` | VERIFY | `npsp__Tribute__c`-family VERIFY | tribute records VERIFY | ? |
| `matching_gift` | VERIFY | `npo02__*` matching fields VERIFY | matching gift fields VERIFY | ? |
| `acknowledgement.*` | VERIFY | `npsp__Acknowledgment_Status__c` ↔ | acknowledgement fields ↔ | VERIFY |
| `receipt.*` | VERIFY | `npe01__*` receipt fields VERIFY | receipt fields ↔ | VERIFY |
| `anonymous` | VERIFY | `npsp__*` VERIFY | `is_anonymous` VERIFY | VERIFY |
| `payment_reference` | VERIFY | `npe01__*Transaction_ID__c` VERIFY | reference field VERIFY | `onlineReferenceNumber` VERIFY |

⚠ **NPSP split gifts**: NPSP's `Opportunity` carries a single designation natively.
Multi-designation gifts require either allocation objects (`npsp__Allocation__c` —
**VERIFY**) or org-specific customisation. A gift with `splits.length > 1` must not be
silently collapsed onto one designation; capability should be `partial` with a clear
error when allocations are unavailable.

⚠ **NPSP write path**: prefer creating the `Opportunity` and allowing NPSP automation
to generate the payment, rather than writing both. Writing both risks duplicate
payments and broken rollups. **VERIFY** in a sandbox against the installed package
version.

## Commitment

| Canonical | Salesforce NPC | Salesforce NPSP | Blackbaud RE NXT | Bonterra EveryAction |
|---|---|---|---|---|
| (object) | `GiftCommitment` | `npe03__Recurring_Donation__c` / pledge `Opportunity` | recurring gift / pledge | recurring commitment |
| `commitment_type` | VERIFY | object choice implies type | object/type distinction VERIFY | VERIFY |
| `status` | `Status` (Draft/Active/Paused/Failing/Lapsed/Closed) VERIFY | `npe03__Open_Ended_Status__c` VERIFY | status VERIFY | `status` VERIFY |
| `total_amount` | VERIFY | `npe03__Amount__c`/pledge amount VERIFY | pledge amount VERIFY | VERIFY |
| `installment_amount` | VERIFY | `npe03__Installment_Amount__c` VERIFY | installment amount VERIFY | VERIFY |
| `frequency` | VERIFY | `npe03__Installment_Period__c` ↔ | frequency VERIFY | VERIFY |
| `next_expected_date` | VERIFY | `npe03__Next_Payment_Date__c` ← | next transaction date VERIFY | VERIFY |
| `payment_method.*` | VERIFY | `npe03__*` card fields VERIFY | VERIFY | VERIFY |
| `balance.*` | ← derived | ← `npe03__*` rollups | ← derived | ← derived |

Our `status: "failing"` maps cleanly to NPC's `Failing` (**VERIFY**) but has **no clean
equivalent in NPSP**, where card-failure state is typically tracked by customisation or
by the payment processor. Expect `partial` capability and document that recurring-gift
recovery workflows may not round-trip on NPSP.

## Designation

| Canonical | Salesforce NPC | Salesforce NPSP | Blackbaud RE NXT | Bonterra EveryAction |
|---|---|---|---|---|
| `level: fund` | `GiftDesignation` VERIFY | `npsp__General_Accounting_Unit__c` VERIFY | fund ↔ | designation VERIFY |
| `level: campaign` | `Campaign` VERIFY | `Campaign` ↔ | campaign ↔ | VERIFY |
| `level: appeal` | VERIFY | `Campaign` (nested) VERIFY | appeal ↔ | VERIFY |
| `level: package` | ✗ VERIFY | ✗ | package ↔ | ✗ VERIFY |
| `restricted` | VERIFY | VERIFY | fund restriction VERIFY | ? |
| `goal` | VERIFY | `Campaign.ExpectedRevenue` ↔ | goal fields ↔ | ? |

Blackbaud's four-level fund/campaign/appeal/package hierarchy is the richest. Mapping
it into Salesforce's flatter `Campaign` model is **lossy in that direction** — flag it
explicitly for customers syncing Blackbaud → Salesforce rather than discovering it in
a reconciliation report.

## Transform rules

Applied by the mapping engine, declaratively, so they are testable in isolation:

| Transform | Rule |
|---|---|
| Money | Provider decimal ↔ canonical minor units. Always carry currency explicitly; never assume the org default. Round only at the provider boundary, never internally. |
| Dates | Calendar dates (`gift_date`, `birth_date`) never pass through a timezone conversion. Timestamps are normalised to UTC. Getting this wrong moves gifts between fiscal years. |
| Enums | Provider value → canonical enum via a per-provider lookup table, with the original preserved in `*_raw`. An unmapped value maps to `other` and emits a metric — it never fails the sync. |
| Phones | Stored E.164 where the country can be determined; original preserved. Never discard an unparseable number. |
| Countries | ISO 3166-1 alpha-2 canonically; per-provider lookup for their own code sets. |
| Names | Never reconstruct a display name from parts when the provider supplies one. Nonprofits curate these deliberately. |
| Empty vs absent | Writing `null` clears a provider field; omitting leaves it unchanged. `PATCH` semantics must distinguish these, or a partial update will wipe fields the caller never mentioned. |

That last rule is the most common source of data-loss bugs in integration work and
should have explicit test coverage for every writable field.

## How to close out this document

For each provider, in a sandbox:

1. Create one record of each entity with **every** canonical field populated.
2. Read it back; record what survived, what was transformed, and what was dropped.
3. Update this table with the confirmed field, and remove the **VERIFY** marker.
4. Where a field could not round-trip, add it to `unsupportedFields` or
   `readOnlyFields` in the capability matrix.
5. Re-run the shared contract suite; it should now pass for every capability claimed.

Until a column has been through that exercise, that provider is not ready to enable
for a customer.
