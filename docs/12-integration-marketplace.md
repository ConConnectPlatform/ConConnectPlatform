# 12 — Integration Marketplace

A marketplace page inside the ConConnect CRM listing the available integrations.
A customer picks one and **books a setup call**; our team configures it with them on
that call.

**Prototype**: [`../prototypes/integration-interest-page.html`](../prototypes/integration-interest-page.html)
— published at `https://claude.ai/artifact/FaXKqBka61UCJxAjsqHNGb` (private until
shared from its Share menu).

## Why the booking model changes the plan

Concierge setup — a human configuring each integration on a call — decouples selling
from building.

**We can list an integration before its adapter is finished.** The first few customers
get set up by hand, using whatever combination of direct API calls, exports and
scripts the situation needs. What the customer experiences is a working integration;
what we get is the thing no amount of design produces — real field mappings, real
data volumes, real edge cases, from a real donor database.

That inverts the usual order for the better. [10](10-roadmap.md) sequences the build
against provider research; the marketplace lets a handful of hand-run integrations
inform that build before it hardens. The first Raiser's Edge setup call will teach us
more about the `date_modified` behaviour in [05](05-sync-engine.md) than any amount of
documentation reading.

**It also puts a ceiling on itself, deliberately.** Hand-configuring integrations does
not scale past roughly ten customers, and every one we run by hand is engineering time
not spent on the adapter. The point is to learn, then automate — so keep count, and
treat rising manual setup load as the signal to stop selling ahead of the build.

**And a booking is a far better signal than a click.** Somebody who gives up thirty
minutes of their week wants the thing. A click costs nothing and measures curiosity.
Count bookings, not clicks — see below.

## What to instrument

The click still matters as a funnel step, but the booking is the outcome.

| Event | When | What it tells us |
|---|---|---|
| `marketplace_viewed` | Page renders | The denominator |
| `integration_booking_clicked` | A "Book setup call" button is pressed | Interest |
| `integration_booking_completed` | The calendar confirms a booking | **Intent — the number that decides the roadmap** |
| `integration_setup_completed` | The integration goes live for that org | Conversion, and the real cost per setup |
| `marketplace_other_submitted` | Someone names a system we do not list | Demand we cannot predict |

Properties on each:

```jsonc
{
  "org_id":        "org_…",            // the nonprofit, not the user
  "org_name":      "…",
  "user_role":     "admin",
  "provider":      "blackbaud_renxt",  // salesforce | bonterra_apricot | other
  "provider_other":"Bloomerang",       // other-submitted only
  "card_position": 1,                  // guards against position bias
  "surface":       "marketplace"
}
```

Wire the booking calendar so the provider travels with the booking. The prototype
appends `?integration=<key>` to the booking URL; HubSpot meetings can carry that into
a custom property on the created contact or meeting, which is what closes the loop
between the click and the booking without manual reconciliation.

Then write the outcome back to the CRM record — the request is a sales signal and
should not live only in analytics:

- A HubSpot property (`requested_integration`, multi-value) so whoever runs the call
  sees it.
- The org's own record in our database, so the marketplace can show a returning admin
  that a colleague already booked, rather than letting them book twice.

## The gap between clicked and completed is the useful number

A card that gets clicked often but booked rarely is telling us something specific: the
offer is attractive and the commitment is not. Thirty minutes may be too much, the
times may not suit, or the card may promise more than the booking page delivers.

Watch that ratio per provider. It is more actionable than the raw ranking, because it
separates "they don't want this" from "they want this but not enough to meet about it"
— and those have different fixes.

## Reading the result honestly

**The sample is small.** With customers in the tens, no difference between two
providers will be statistically significant. Treat the ranking as a prompt for the
conversation you are now having with those customers anyway — the setup call itself is
the research. Do not run a significance test on a handful of bookings and report a
winner.

**Card order biases clicks.** The first position draws attention regardless of
content. Randomize order per viewer, hold it stable for that viewer, and record
`card_position`, so the bias can be measured rather than assumed away. Without this
the page partly measures its own layout.

**This only reaches existing customers.** It answers what our customers want next, not
what would win new ones. The second question needs the sales pipeline and conference
conversations, not this page.

**The copy is a variable.** Card wording and category labels move click rates. Keep
them parallel in tone and specificity across cards, or the page measures its own
writing.

## Decision rule — set before the data arrives

Written down now, so the numbers are not read to confirm what we already intended.

- **Blackbaud goes first regardless.** The ISV partnership is live and it is the only
  provider with a named channel for sandbox access and rate limits
  ([11](11-partnership-status.md)). Demand would have to be strongly against it to
  change that.
- **Salesforce second unless Apricot clearly leads on completed bookings** — Salesforce
  is the only provider we can fully verify today with self-service sandboxes.
- **Apricot's position turns on its licence gate**, not only on bookings. Customers
  below Enterprise or Pro cannot be integrated at all, so the question worth asking on
  the call is which tier they hold.
- **Three or more mentions of the same unlisted system** makes it a roadmap candidate.
  This is the most valuable thing the page can surface, because it is the only outcome
  we cannot predict.
- **More than ten hand-run setups, or manual setup crowding out adapter work**, means
  stop listing ahead of the build and finish the automation.

## Page content

Card copy states something concrete and true per provider rather than generic
marketing, drawn from the research in [`providers/`](providers/):

- **Raiser's Edge NXT** — split gifts, soft credits and tributes kept intact; the full
  fund/campaign/appeal/package hierarchy. These are exactly what naive integrations
  lose, so naming them signals we know the product.
- **Salesforce** — works with either data model and detects which one the org runs;
  leaves existing rollups and automations alone. This answers the actual fear a
  Salesforce admin has about a third-party integration.
- **Apricot** — participants, programmes, enrolments, services and outcomes, ready for
  funder reporting; the customer's own forms rather than a fixed template. The licence
  requirement is on the card deliberately: it pre-qualifies, and it is far better
  discovered here than on the call.

The Apricot card says nothing about case-note content. What is synced there is an open
compliance question ([08](08-security-and-compliance.md)) and the page must not commit
ahead of that answer.

Category pills say what each integration **is** (Fundraising, Case management) rather
than how far along we are. A marketplace tells a customer what a thing does; build
status is our concern, and "In design" on a card you can book a call for reads as a
contradiction.

## Build notes

- Set `BOOKING_URL` in the page script to the real HubSpot meetings link. Individual
  cards can override it with `data-booking` if each integration needs its own meeting
  type — worth doing if different people run different setups.
- The marketplace is a settings surface, not a marketing page; it lives where an admin
  configures their account.
- Cards are equal height with the buttons aligned. Unequal cards give one option more
  visual weight and bias the result.
- Show a returning admin that their organization already has a booking or a live
  integration, instead of offering the same call again.
- Works at phone width.
- When an integration's adapter ships, its card swaps the booking link for the real
  connect flow ([06](06-public-api.md#connections)) — including the pending state that
  human-in-the-loop credential issuance needs.
