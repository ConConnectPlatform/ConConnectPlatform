# Brand Tokens

Extracted from Untapped Solutions' own HubSpot email templates (the customer
newsletter and the product's transactional notification template), September 2026.

⚠ **Not yet confirmed against staging or a brand guide.** The staging and production
domains are blocked by this environment's network policy, so these are the values the
brand actually uses in email, which is the best available source from here. Confirm
against the app before treating them as canonical, and correct this file where they
differ — email templates drift from the product.

## Palette

| Token | Value | Where it came from | Use |
|---|---|---|---|
| `--navy` | `#102F40` | Newsletter header band, feature panel | Top navigation, dark surfaces |
| `--brand` | `#1B7CAF` | Hero band, card headings | Primary brand blue, links, quiet accents |
| `--cyan` | `#67D5F5` | "SOLUTIONS" in the wordmark, subheads on navy | Highlight on dark only — too light for text on white |
| `--cyan-light` | `#8DE3FA` | Headline highlight on navy | Display emphasis on dark |
| `--action` | `#F26522` | Primary CTA button, "Voting is open" badge, accent rule | **Actions only.** Buttons, badges. Keep it scarce |
| `--action-light` | `#F6A06B` | Eyebrow text on navy | Small labels on dark |
| `--bg` | `#EEF2F5` | Email body background | Page ground |
| `--surface` | `#FFFFFF` | Content sections | Cards, panels |
| `--tint` | `#F2F9FD` | Three-up feature cards | Quiet fills, inset panels |
| `--line` | `#D8EBF4` | Feature card borders | Borders, dividers |
| `--ink` | `#1A1A1A` | Body copy, headings | Primary text |
| `--ink-2` | `#3C484F` | Secondary paragraphs (also `#3E4B52`, `#58666E`) | Secondary text |
| `--ink-3` | `#7A878E` | Fine print (also `#6D7A81`, `#8F999E`) | Muted text, captions |
| `--pale-1` | `#B9DCEB` | "Customer Community" label on navy | Muted text on dark |
| `--pale-2` | `#E4F0F5` / `#EAF7FC` | Body copy on navy | Body text on dark |

Deliberately excluded: `#4472C4` appears as a link color in the transactional
template, but it is the default Microsoft Office theme blue and almost certainly
arrived with a pasted template rather than by choice. `--brand` `#1B7CAF` is the real
blue.

Also excluded: `#333333` and `#f7f7f7` from the transactional template's header and
footer bands. They do not match the newsletter's navy-and-blue system, which suggests
the transactional template predates the current brand. **Worth aligning** — a customer
who gets a grey-and-black system email after seeing a navy-and-cyan product has seen
two different companies.

## How the palette works

The system is **navy and blue with orange as the single action color.** That division
is what makes it legible, and it is worth protecting:

- Blue carries identity and structure — navigation, headings, borders, quiet accents.
- Orange carries action — the button you are meant to press. In the newsletter it
  appears exactly three times: the badge, the CTA button, and a 70×5px rule. That
  restraint is what gives it force.
- Cyan is a highlight that only works on navy. At `#67D5F5` it fails contrast on
  white, so it must not be used for text on light surfaces.

The most common way to damage this palette is to spend the orange on non-actions.
Once several things on a page are orange, none of them reads as the thing to click.

## Dark theme

The brand has no published dark palette, so the tokens in
[`../../prototypes/integration-marketplace.html`](../../prototypes/integration-marketplace.html)
derive one by promoting the existing dark surfaces:

| Token | Light | Dark | Reasoning |
|---|---|---|---|
| `--bg` | `#EEF2F5` | `#0B1E29` | A step darker than the navy so surfaces can sit above it |
| `--surface` | `#FFFFFF` | `#102F40` | The brand navy becomes the card |
| `--brand` | `#1B7CAF` | `#67D5F5` | The brand blue is too dark on navy; the cyan is exactly the value the brand already uses on navy |
| `--action` | `#F26522` | `#F26522` | Unchanged — it carries on both grounds |
| `--ink` | `#1A1A1A` | `#EAF7FC` | The pale blue the newsletter already uses for body copy on navy |

This is an inference, not a brand decision. If a dark palette exists, replace it.

## Other brand facts worth recording

- **Wordmark**: "UNTAPPED" at weight 800, "SOLUTIONS" at weight 400 in `#67D5F5`,
  letter-spacing `-0.2px`. Reproduced in the marketplace prototype's top navigation.
- **Tagline**: "Serving those who serve others."
- **Logo asset**: hosted on HubSpot at `untappedsolutions.io/hubfs/` (Untapped
  Solutions Logo (Email Header).png).
- **Typography**: email templates use Arial/Helvetica, which is an email constraint
  rather than a brand choice, so it says nothing about the product typeface. The
  prototype uses Archivo for display — its heavy weights and tight tracking match the
  brand's 800-weight, negative-tracking headline style — with IBM Plex Sans for body.
  Replace both if the product has its own typeface.
- **Product vocabulary**, taken from the transactional email and form notifications:
  *client*, *intake*, *program*, *enrolled*, *form*. The marketplace navigation uses
  these words rather than invented ones.
- **Current shell**: the transactional email refers customers to "the support section
  at the bottom of the side bar menu", which suggests the app currently uses a
  **sidebar**, not a top navigation. The prototype builds the top nav as requested —
  worth confirming which shell the marketplace is actually landing in.

## Using these

Copy the `:root` block from the marketplace prototype. It defines the full light
palette on bare `:root`, redefines only the tokens under
`@media (prefers-color-scheme: dark)` guarded with `:root:not([data-theme="light"])`,
and again under `:root[data-theme="dark"]`, so an explicit theme choice wins in both
directions.
