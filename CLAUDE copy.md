# CLAUDE.md — Signal Cleveland scrollytelling stories

Guidance for building pinned-photo "scrolly" stories for Signal Cleveland (signalcleveland.org, a Newspack/WordPress site). This repo holds the first one, the John Williams billboard piece, and is the reference implementation for future stories.

## What's here

No build step, no package manager, no tests. Each story is two hand-edited static HTML files:

| File | Role |
|---|---|
| `john-williams-scrolly.html` | **Child page.** The whole story: markup, CSS, and one inline `<script>`. Loaded in an iframe via pym.js. Also runs standalone for previewing. |
| `wordpress-embed-snippet.html` | **Parent snippet.** Pasted into a WordPress "Custom HTML" block. Creates the tall scroll track, pins the iframe, relays scroll progress into it. |

(`index.html` was renamed to `john-williams-scrolly.html`. If the old GitHub Pages preview URL is still in use, it needs `/john-williams-scrolly.html` appended now.)

For a new story, copy both files and rename them with a new story-specific class prefix (here everything is `jw-`; parent ids are `jw-embed-wrap` / `jw-billboard-hook`). Change every occurrence in both files so stories can't collide if two are ever on one page.

## How it works

1. The parent's `#jw-embed-wrap` is a very tall invisible track. `#jw-billboard-hook` is `position:sticky; top:0; height:100vh` inside it, holding the pym iframe.
2. On scroll the parent computes progress 0→1 through the track and sends it to the child with `pymParent.sendMessage('progress', …)`.
3. The child never scrolls when embedded. It converts progress to `v` (vh scrolled, 0→`TOTAL_VH`) and calls `render(v)`, which sets `transform`/`opacity` on every element. Embedded, it eases toward the target on its own `requestAnimationFrame` loop (factor 0.22) so bursty postMessage delivery doesn't look janky.
4. Standalone (not in an iframe), the child gives itself `height: (TOTAL_VH+100)vh` and drives `render()` from its own scroll. **This is the fast preview loop.**

### Parent ⇄ child messages (pym.js v1)

- child → parent `trackHeight` (px): the child measures its real content height and reports it, so the parent's `--jw-scroll-length` is always exact. Never hand-maintain a track height. The `734vh` fallback in the snippet CSS is only the pre-first-message placeholder.
- parent → child `progress` (0–1): sent on scroll (rAF-throttled) and after each `trackHeight`.
- parent → child `postMeta` (JSON): real author, avatar and date scraped from the post's own `.entry-header`. It's sent from inside the `trackHeight` handler so the child's listener is guaranteed to exist. The child's hardcoded byline fades in only after this arrives (or a 3s timeout), to avoid flashing the wrong byline.
- child → parent `height` (built-in `child.sendHeight()`): used only in the reduced-motion static layout.

### The child's layout model

- **Everything is absolutely positioned and JS-driven.** Each `.jw-item` is `position:absolute; inset:0` and flex-aligned; `render()` moves it with `translateY((item.center − v)vh)`. Cards are fully opaque and move 1:1 with scroll, like real page content, not fade-ins.
- **Cards self-measure.** `layoutStack()` reads each card's `offsetHeight`, lays cards out top-to-bottom with `STACK_BUFFER` (80vh) between, and returns the running end. `TOTAL_VH` is derived from that. **Adding a card needs no JS edits and no timeline edits**; the track grows automatically.
- **Order on the timeline:** headline + caption (intro stack) → floating Reddit/quote comments → item cards. Comments start at the intro's end and are fixed-length pulses (`T.commentsStagger`, `T.commentsWidth`).
- **Hand-tuned constants (absolute vh) in `T`:** `pullbackEnd`, `billboardIn`, `closeupOut` control the opening zoom-out and closeup→billboard crossfade. They are *not* derived from the intro's length, so if the headline/caption cards change height a lot, re-check that this sequence still lands where intended.
- **Photo frame:** `.jw-zoom` holds `.jw-closeup` and `.jw-billboard` (declared in `:root` as `--img-closeup` / `--img-billboard`), scaled 1.4→1.0 (mobile 1.08→1.0). Then a filmstrip of `data-bg` layers.

### Adding a section (the common task)

Copy an existing `.jw-item` block. Alignment classes: default = left, `.jw-item-right`, `.jw-item-center`. Add `jw-mobile-center` so it re-centers on phones (≤768px). Attributes on the card (or the `.jw-item-stack` wrapping it):

- `data-bg="<image url>"` — swaps the pinned photo when scroll reaches this card. Default is an opaque slide-up (new photo travels `translateY(100vh)→0` while the old one exits the top, rigidly 100vh apart so they never gap). Add `data-bg-transition="fade"` on that element to crossfade in place instead.
- `data-photo-exit="center"` — the photo locks to its card and they scroll away together once the card's bottom edge clears the screen (+`CARD_STICK_MARGIN_VH`). Without it, the photo waits until its card has fully scrolled off. It's read from the card, so it also works for the base billboard photo, which has no `data-bg`.
- Group cards that must move as one rigid unit (e.g. a card plus a note callout) inside one `.jw-item-stack`. The stack is measured as a whole.
- Card variants that exist: `.jw-item-card` (body copy), `p.jw-item-sub` (condensed bold subhead), `p.jw-pivot-lead` (bigger, with a red `<span>`), `.jw-note-card` (cream callout with yellow rule and eyebrow label), `.jw-item-caption`, `.jw-caption-card`, `.jw-headline-card`, and the `.jw-cmt` + `.jw-rd-card` Reddit/news quote cards.
- Comments: slots are named `.jw-cmtN` with fixed left/right/top anchors (two columns). `cmt4` intentionally doesn't exist. On mobile, `cmt2` and `cmt5` are hidden and the rest are re-slotted. New comment ⇒ add its slot in both the desktop and the ≤768px CSS and check it doesn't overlap neighbors.

## Gotchas learned the hard way

Read the git log (`git log`) for the full story. The commit bodies explain each of these.

- **Reduced motion is a separate layout.** `render()` returns immediately, so nothing can depend on JS positioning. The `prefers-reduced-motion` CSS block un-pins everything and turns the piece into a static top-to-bottom article; the parent has a matching override. **Any new element type needs a rule in that block** or it will stack invisibly on top of the others. Each `data-bg` card also gets a static `.jw-static-photo` twin injected by JS.
- **Never use `vh` sizing in the embedded reduced-motion layout.** The iframe's `vh` is the iframe's own height, which the parent sets from the child's content height, giving a runaway feedback loop. Use fixed px under `.jw-embedded` (see existing rules).
- **Per-photo mobile crops are keyed on the filename.** Rules like `.jw-layer[data-bg*="Billboards-2-scaled"]` shift `background-position` so the subject stays in frame on narrow screens (`cover` on a 100vh box crops hard). When adding a photo, check it at ~375px and add a rule if the subject is cut off. Filenames in CSS must match the `data-bg` URLs.
- **Specificity traps:** `.jw-headline-card p` and `.jw-caption-card p` set large type, so smaller variants need the parent prefix (`.jw-headline-card p.jw-photo-credit`, `.jw-caption-card p.jw-cap-body`). A bare class loses.
- **Flash-of-stacked-cards:** `.jw-sticky` starts `opacity:0` and is revealed by `.jw-ready` after the first `render()`. Don't remove that.
- **`isMobile` is evaluated once at load** (768px breakpoint). Fine for phones; a resized desktop window won't switch modes until reload.
- **Fonts shift line wraps**, so the child re-runs `buildStack()` on `document.fonts.ready` and re-sends `trackHeight`. Keep that path intact if you touch `recompute()`.
- **Parent CSS needs `!important` almost everywhere.** It's fighting the Newspack theme. See the numbered comments at the top of the snippet for each override and why (full-bleed `alignfull` + JS margin correction for sidebar layouts, `.site-content{overflow:visible}` on Single Wide/Feature templates because any non-visible `overflow` on an ancestor kills `position:sticky`, masthead un-stick, visually hiding `.entry-header` while keeping it in the DOM for `postMeta`, hiding AddToAny). Don't use the `margin-left:-50vw` trick and don't set `margin` shorthand on the wrapper (both were bugs; JS owns `margin-left`).
- **Signal's host defers external scripts.** The parent's inline script waits for `DOMContentLoaded` before `new pym.Parent(...)` for that reason. Keep that wrapper.
- iOS Safari fires `resize` during scroll (address bar), so parent resize handling is rAF-throttled and computes the margin correction without a reset-then-remeasure flash. Preserve that.

## Design conventions (Signal brand)

- Fonts: **Inter** for body, **Roboto Condensed Bold** for headlines/subheads (`--font-body`, `--font-heading`). Headline sizes mirror Newspack's `.entry-title` scale via `--newspack-theme-font-size-*`. Not all-caps except note-card eyebrow labels.
- Palette used so far: text/background slate `#404f54`, teal `#23685b` / light `#51a89a`, accent yellow `#f4c913` (highlights, note rule), accent red `#d64d4d` (emphasis), gray `#879599` (meta text), cream `#f4f0e6` (note card), border `#ccd8db`. Reddit avatar gradients vary per commenter.
- Byline markup copies the live site's `.entry-subhead` / `.entry-meta` classes so it matches theme styles. Photo credit sits directly under it.
- Cards: white, `box-shadow: 0 20px 50px rgba(0,0,0,.4)`, body copy `clamp(18px,1.9vw,21px)` / 1.75. Desktop cards are capped near `46vw` so the photo stays visible; mobile widens to 82–90vw.
- Accessibility: photo layers use `role="img"` + `aria-label`; static-fallback photos are `aria-hidden`; the real title stays in the DOM (visually hidden) for screen readers.

## Workflow

- **Preview:** open the child HTML directly in a browser (standalone mode), or `python3 -m http.server` in this folder. Check embedded behavior only in a WordPress draft (or a test page carrying the snippet), because that's where the sticky/full-bleed/theme issues show up.
- **Before calling something done, check:** ~375px (phone), just above/below 768px, ~1440px desktop, and reduced motion (DevTools → Rendering → emulate `prefers-reduced-motion`). Confirm standalone and embedded scroll feel the same speed.
- **Deploying:** upload the child HTML to the WP Media Library and set `CHILD_URL` in the snippet to its direct URL (currently `https://signalcleveland.org/wp-content/uploads/2026/09/john-williams-scrolly.html`). The GitHub Pages URL in the snippet comments is for iterating only. Photos are also Media Library URLs; make sure no `REPLACE_…` placeholders remain in `:root` or `data-bg`.
- **Article copy is reported journalism.** Don't rewrite, trim, or "tighten" text, and keep quotes verbatim (curly quotes, existing punctuation). Structural changes are fine; wording changes need the reporter's say-so.
- **Comments in this code are long and explain *why*** (what broke, how it was confirmed, why the fix works). Match that style when adding non-obvious code.
- **Commits:** short imperative subject, body explaining the problem and reasoning. Only commit when asked.
