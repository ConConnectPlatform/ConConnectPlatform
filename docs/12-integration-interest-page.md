# 12 — Integration Interest Page

A page inside the ConConnect CRM that advertises the three planned integrations and
measures which one customers actually ask for. The point is to sequence the build
against real demand instead of a guess.

**Prototype**: [`../prototypes/integration-interest-page.html`](../prototypes/integration-interest-page.html)
(published at `https://claude.ai/artifact/FaXKqBka61UCJxAjsqHNGb` — private until
shared from the page's Share menu).

## Where the measurement lives

Inside our own product, not in the prototype. The prototype is the design and the
copy; it records clicks only in the viewer's own browser, which is fine for showing
the page and useless for counting.

The real page ships in the ConConnect CRM, where we already know who the viewer is
and which organization they belong to. That matters more than the raw count: **"three
clicks" is noise, but "three clicks from two Apricot customers and one prospect in
the pilot" is a decision.** Anonymous aggregate counts would throw away the only part
worth having.

## What to instrument

One event per meaningful action, sent to Mixpanel, with the organization attached so
results can be read per customer rather than per click.

| Event | When |
|---|---|
| `integration_page_viewed` | Page renders. Gives the denominator — a request rate is meaningful, a raw count is not. |
| `integration_interest_clicked` | A "Request early access" button is pressed. |
| `integration_other_submitted` | Someone names a system we do not list. |

Properties on every event:

```jsonc
{
  "org_id":        "org_…",       // the nonprofit, not the user
  "org_name":      "…",
  "user_role":     "admin",       // admin | staff | viewer — who asks matters
  "plan_tier":     "…",
  "provider":      "blackbaud_renxt",  // salesforce | bonterra_apricot | other
  "provider_other":"Bloomerang",  // only on the other-submitted event
  "card_position": 1,             // guards against position bias — see below
  "surface":       "settings_integrations"
}
```

Also write the result back to the CRM record, because a click is a sales signal and it
should not live only in analytics:

- Set a contact/company property in HubSpot (`requested_integration`, multi-value) so
  it reaches whoever follows up.
- Store the request against the organization in our own database so the page can show
  returning users what they already asked for, and so the pilot list builds itself.

## Reading the result honestly

Three things will make this measurement lie if they are not handled.

**1. The sample is small.** With a customer base in the tens, no difference between
two providers will be statistically significant. Treat the numbers as a prompt for a
conversation, not as a verdict — the follow-up call with the three organizations that
clicked Apricot will teach us more than the count did. Do not run a significance test
on twelve clicks and report a winner.

**2. Card order biases clicks.** The first card gets more attention regardless of
content. Randomize card order per viewer, hold it stable for that viewer, and record
`card_position` so the bias can be measured rather than assumed away. Without this,
the page measures its own layout.

**3. Existing customers are not the whole market.** This page only reaches
organizations that already use ConConnect. It answers "what do our customers want
next", not "what would win us new customers". Those are different questions and the
second one needs a different instrument — the conference conversation, the sales
pipeline, the prospect list.

Related: the copy itself is a variable. "Building" versus "In design" on a status pill
changes click rates. Keep the status labels truthful and identical in tone across
cards, or the page measures its own wording.

## Decision rule — set before looking

Write down what the result will change, now, before any data arrives. Otherwise the
numbers get read to confirm whatever we already intended to build.

Proposed:

- **Blackbaud first regardless**, because the ISV partnership is live and it is the
  only provider where we have a named channel for sandbox access and rate limits
  ([11](11-partnership-status.md)). Demand data would have to be strongly against it
  to change that.
- **Salesforce second unless Apricot clearly leads**, since Salesforce is the only
  provider we can fully verify today with self-service sandboxes.
- **Apricot's position is set by its licence gate, not only by clicks.** A customer on
  a lower Apricot tier cannot be integrated at all, so the useful question the page can
  answer is *which organizations are on Enterprise or Pro* — worth asking directly in
  the follow-up rather than inferring from a button press.
- **A strong showing for "something else"** is the most valuable outcome available
  here, because it is the only one we cannot predict. Treat any provider named three
  or more times as a candidate for the roadmap.

## Page content

The copy in the prototype is drawn from the provider research, so each card states
something concrete and true rather than generic marketing:

- **Raiser's Edge NXT** — split gifts, soft credits and tributes kept intact; the full
  fund/campaign/appeal/package hierarchy. These are real Raiser's Edge capabilities
  that naive integrations lose, so saying it signals we understand the product.
- **Salesforce** — works with either data model and detects which one the org runs;
  leaves existing rollups and automations alone. This addresses the actual fear a
  Salesforce admin has about a third-party integration.
- **Apricot** — participants, programmes, enrolments, services and outcomes, ready for
  funder reporting; the customer's own forms rather than a fixed template. The licence
  requirement is stated on the card on purpose: it pre-qualifies, and it is better
  discovered here than on an implementation call.

Note the Apricot card's wording avoids promising anything about case-note content.
What is synced there is an open compliance question
([08](08-security-and-compliance.md)), and the page should not commit ahead of that
answer.

## Build notes

- The page is a settings surface, not a marketing page. It lives where an admin
  configures their account.
- Cards must be equal height with the buttons aligned. Unequal cards give one option
  more visual weight and bias the result.
- A requested state persists per organization, so a returning admin sees what their
  colleague already asked for rather than requesting twice.
- Works at phone width; some of these admins will open it on a phone.
- Once an integration ships, its card changes from a request button to a real connect
  flow ([06](06-public-api.md#connections)) — including the pending state that
  human-in-the-loop credential issuance needs.
