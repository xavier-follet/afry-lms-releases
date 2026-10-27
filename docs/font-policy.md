# Font Policy

_Applies to the AFRY LMS **application GUI** (the admin, editor and common/auth SPAs).
Established by issue #66._

## The rule: two fonts, plus mono for code

| Role | Font | Token |
|------|------|-------|
| **Headings** (`h1`–`h6`, MUI heading variants) | **Plus Jakarta Sans** | `--font-heading` |
| **Body & labels** (everything else: text, buttons, inputs, captions, table cells) | **Roboto** | `--font-body` |
| **Code / technical values** (SCORM ids, hex colours, URLs, source view) | system monospace | `--font-mono` |
| **Icons** | Material Icons / Symbols | — |

Do **not** introduce any other font family into the GUI.

## Where the tokens live

Single source of truth: `packages/frontend/src/editor/styles/typography.css` (`:root`).
The MUI theme (`packages/frontend/src/editor/theme.js`) mirrors the same values so MUI
components and raw HTML stay in sync.

```css
:root {
  --font-heading: 'Plus Jakarta Sans', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  --font-body:    'Roboto', -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif;
  --font-mono:    ui-monospace, 'Cascadia Code', 'Segoe UI Mono', Consolas, monospace;
}
```

## Where the files live

Both families are **served from our own origin**. `typography.css` imports
`packages/frontend/src/common/styles/fonts/plus-jakarta-sans.css` and `roboto.css`, which
declare `@font-face` over the `woff2` files in
`packages/frontend/src/common/assets/fonts/`. `unicode-range` is preserved per subset, so a
browser fetches only the scripts a page actually renders. **No monospace webfont is
loaded** — mono uses whatever the OS provides.

They used to come from Bunny Fonts via two `@import url(https://…)` lines. #255 replaced
that, and the reason is worth keeping: **a CSS `@import` cannot be made conditional**. It
fires when the stylesheet parses — before React mounts, before any `/api/` call, before
login — so no setting, flag or preference could ever have suppressed it. Customers who are
air-gapped, or whose policy forbids third-party egress, need those requests not to exist.
Since these two families are fixed, shipping them is simply the correct build decision;
it also takes a render-blocking cross-origin stylesheet off the critical path for
everyone else.

To change a family or an axis, edit the manifest in `scripts/vendor-fonts.mjs` and re-run
it — it downloads the `woff2` files and rewrites the CSS. Do not hand-edit either. The
fonts are OFL / Apache-2.0, which permit redistribution.

### GUI chrome rendered inside the player

The player does not load `typography.css`: it is in `player.css`, which every learner fetches,
and the learner path must not carry an external font request (offline SCORM packages included).
Author-only surfaces mounted inside the player — the preview tools panel (#209) — therefore call
`ensureGuiFont()` from `common/utils/guiFont.js` on mount. It injects the same vendored
`roboto.css` (imported `?inline`, so the rules stay out of `player.css`) and publishes
`--font-body`, so the CSS still names no family. A learner downloads no `woff2` from it:
`@font-face` only fetches a face that something on the page renders in. `guiFont.js` and
`typography.css` are the only two places that spell the stack out; change one and change
the other.

## How to use it

- **CSS**: use `font-family: var(--font-heading | --font-body | --font-mono)`. Never hard-code a
  family name. Body is inherited from `<body>`, so you rarely need to set it explicitly.
- **MUI**: use the theme. Headings come from the `h1`–`h6` variants; everything else already
  defaults to Roboto. Don't pass `fontFamily` in `sx` except to reach for `var(--font-mono)`.
- **Adding a new component**: it should need **zero** font declarations. If you find yourself
  typing a font name, stop and use a token.

## Exceptions (deliberately NOT governed by this policy)

These render user/organisation content and are configurable by design — they are features, not
GUI chrome:

1. **Organisation portal** (`packages/frontend/src/organization/**`) — each portal template ships
   its own display/body font pairing, and org admins can override them
   (`portal_font_display` / `portal_font_body`). Driven by the `--sp-font-*` variables. The
   nine **template** families are vendored like the GUI ones, each imported by the template
   stylesheet that uses it. `organization/main.jsx` loads all five template stylesheets, so
   all nine sets of `@font-face` rules ship in `organization.css` (~4 KB gzipped) — what is
   per-template is the *downloading*: a browser fetches a woff2 only for a face something on
   the page renders in, so a portal pulls its own two families and nothing else. An
   **override** is not vendored, because it can be any family.
2. **Player content** (`packages/frontend/src/player/**`) — training themes expose per-element
   `--*-font-family` variables; fonts are chosen by authors and loaded on demand. See the theme
   editor and `common/themes/*.json`.

## The `EXTERNAL_FONTS` flag

The two exceptions above are open-ended: we cannot ship a font before knowing which one an
author will pick. That single case is governed by a deployment flag (#255):

| `EXTERNAL_FONTS` | Effect |
|---|---|
| `bunny` (default) | An author-chosen family is fetched from `fonts.bunny.net` on demand |
| `off` | Nothing is fetched. The pickers offer web-safe families only, and any family already saved falls back to its declared stack |

It is a property of the *installation* — does this box have egress, does this customer's
policy permit it — so it lives in `.env` and `docker-compose.yml` next to `PORTAL_BASE_URL`,
**not** on a settings page. A settings page could not have worked: a database read happens
long after the stylesheets have parsed, and a SCORM package has no backend to ask at all.

The frontend learns it from `GET /api/version` (unauthenticated), read through the shared
axios client so `VITE_API_URL` applies — a bare `fetch('/api/version')` would hit the
frontend's own Nginx on an installation whose API is not same-origin, get the SPA fallback
HTML, and silently conclude `off`. Consult the flag in exactly one place: `ensureWebFont()`
in `common/utils/webFont.js`. A call made before the answer arrives chains on the shared
promise rather than firing — never let one through early, or an air-gapped install emits the
request the flag exists to prevent. Two failures are retried before concluding, because a
backend that is still starting looks exactly like one that does not exist; only then is
silence read as `off`.

Two things follow, and both are deliberate:

- `common/components/FontPicker.jsx` is the **only** font picker — the theme editor and the
  portal design editor both use it. It lists web-safe families first, always, and web fonts
  in a second group only when the flag allows them.
- The curated font pairings in the portal design editor are hidden entirely when the flag is
  off, rather than offered and quietly broken. Most of them name families we do not vendor;
  one (`Corporate`, Barlow Condensed + Barlow) happens to be a template's own pairing and
  would in fact render — the block is hidden wholesale because `isLocalFamily()` knows only
  the web-safe list, not what has been vendored. See the note below.

### Known limits

- **`isLocalFamily()` means "web-safe", not "on our own disk".** The nine portal families are
  vendored and their `@font-face` rules already ship in `organization.css`, yet an org admin
  who picks one still triggers a `fonts.bunny.net` request, and with the flag off they cannot
  be offered at all. Teaching `ensureWebFont` about the vendored set — and injecting the local
  stylesheet instead of a CDN link — would give an air-gapped install 20 fonts instead of 9
  and stop a default install fetching fonts it already ships.
- **Imported SCORM packages are not governed by the flag.** They bring their own CSS, and many
  open with an `@import` against Google Fonts. Nothing in this codebase can rewrite someone
  else's course; enforcing it needs a `font-src 'self'` CSP on the `/files/` location. See
  `docs/deployment/production-deployment.md`.
- **Vite must not inline a woff2.** `assetsInlineLimit` in `vite.config.js` excludes `.woff2`
  precisely because a base64 `data:` URI is fetched *with* the stylesheet, which defeats
  `unicode-range` and puts subsets nobody renders on the critical path.

## History

- **#66** — Consolidated the GUI to this 2-font policy. Previously the chrome used Plus Jakarta
  Sans for everything and declared monospace four inconsistent ways; a dead `editor/index.css`
  also set a conflicting system-font stack (removed).
- **#255** — Vendored the GUI and portal-template families, so a stock install makes no
  external font request. Added `EXTERNAL_FONTS` for the author-chosen case, merged the two
  font pickers into one, and put the portal templates back on their `--sp-font-*` tokens —
  several rules had hard-coded the family, so an org admin's override never reached them.
