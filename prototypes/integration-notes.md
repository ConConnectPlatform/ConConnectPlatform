# Dropping the marketplace into the CRM

Two files here:

| File | What it is |
|---|---|
| `integration-marketplace.html` | Standalone page, including a mock top navigation. For showing people. |
| `integration-marketplace.partial.html` | **The one to ship.** Page content only, no nav, styles scoped under `.ccn-mkt`. |

## The partial

Paste the whole file into a page template. It carries its own `<style>` and
`<script>`, and everything is scoped under `.ccn-mkt` — tokens are declared on that
class rather than `:root` — so it cannot collide with the app's existing styles and
the app's styles cannot leak into it.

It renders inside whatever shell the app already has. It does **not** include the top
navigation; that belongs to the shell.

### Three things to set before it ships

1. **`BOOKING_URL`** in the script. Currently `https://meetings.hubspot.com/REPLACE-ME`.
   A card can override it with `data-booking` on its button if different integrations
   need different meeting types.
2. **The palette.** The tokens at the top were read off the Untapped email templates,
   not off staging — see [`../docs/reference/brand-tokens.md`](../docs/reference/brand-tokens.md).
   If the app has its own tokens, delete the block and point these at them.
3. **The fonts.** The partial asks for Archivo and IBM Plex Sans and falls back to
   system fonts. If the app already loads a typeface, swap `--mkt-display` and
   `--mkt-body` to it and drop the Google Fonts link from the standalone page.

### Tracking

`track()` in the script calls `window.mixpanel.track` when it is present and does
nothing otherwise. Point it at whatever the app actually uses. The events and their
properties are specified in
[`../docs/12-integration-marketplace.md`](../docs/12-integration-marketplace.md) —
the important one is `integration_booking_completed`, which fires from the booking
calendar rather than from this page.

`card_position` is recorded on every click because card order biases clicks toward
the first position. **Randomize the card order per viewer and hold it stable for that
viewer**, or the page partly measures its own layout rather than demand.

## The navigation entry

One item, pointing at wherever this page is routed:

```
Integrations   →   /integrations
```

Mark it active when the route matches. In the standalone page that is
`aria-current="page"`, which is what a screen reader announces as the current
location — worth keeping whatever markup the shell uses.

**Where it goes is an open question.** The mock puts it last in the top nav, after
Reports, on the reasoning that it is a configuration destination rather than daily
work. Two things to check against the real shell:

- **The nav items in the mock are inferred**, not read from the app: Dashboard,
  Clients, Programs, Forms, Reports. They come from the product's own vocabulary in
  its notification emails ("New client enrolled", "intake due", "Fall 2026 Program
  Interest Form"). The real labels and order should win.
- **The app may not have a top nav at all.** Its transactional email tells customers
  to use "the support section at the bottom of the side bar menu", which suggests a
  sidebar. If so, the partial still drops in unchanged — only the nav entry moves.

## What could not be checked

Staging was not reachable from the environment this was built in: no browser tool, and
the network policy blocks every outbound host including `untappedsolutions.io` and
`staging.untappedsolutions.io`. The CRM codebase was not reachable either — the only
repository visible to the session is the empty `ConConnectPlatform/ConConnectPlatform`
profile repo.

So the shell, the real nav, the app's own tokens and its fonts are all **inferred**.
The partial is built to survive being wrong about them: scoped styles, tokens in one
block, no assumptions about the surrounding markup.

A screenshot of the staging top navigation would settle the nav question in one step.
