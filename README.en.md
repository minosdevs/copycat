# copycat

[Français](./README.md) · **English**

A Claude Code skill that takes a URL and gives you back **the same site**. Not "in the spirit of":
the same. Same fonts, same colours, same spacing, same responsive behaviour, same hovers, same
animations, clean console.

```
/copycat https://example.com
```

---

## Why

Ask an AI to "rebuild this site" and it looks at one screenshot, guesses a font, rounds the spacing,
picks a "close enough" grey, forgets the hover, and hands you something that's 70 % there. The
remaining 30 % is everything that makes a site look professional.

copycat replaces guessing with **measuring**:

| Without copycat | With copycat |
|---|---|
| One above-the-fold screenshot | Full-page + fold by fold on desktop, tablet, mobile + one crop per section + hover states |
| "Looks like Inter" | `document.fonts` + every `@font-face`, `.woff2` files downloaded, weights actually in use |
| "A light grey" | Area-weighted computed palette, `:root` CSS variables, exact gradients and shadows |
| "About 40px" | Computed typography per tag (size, weight, line-height, tracking, colour), containers, breakpoints, spacing scale |
| Hover and motion forgotten | Measured hover states (before → after), keyframes, transitions, sticky elements, reveal-on-scroll |
| "I think it's right" | Pixel diff original vs clone, worst zones mapped to sections, clone console and network at 0 errors |

---

## What it does

### 1. `capture.mjs` — photograph and dissect the original

```bash
node skills/copycat/scripts/capture.mjs https://example.com
```

Writes `copycat/example.com/`:

```
REPORT.md                    ← the readable summary: stack, fonts, palette, tokens, typography, layout, sections, CTAs, hover, motion, console
manifest.json                ← everything, as JSON
screens/
  desktop-full.png           ← full page (native, or stitched folds when the site reveals on scroll)
  desktop/desktop-fold-01.png…  ← a real viewport at each scroll position
  tablet-full.png, mobile-full.png, tablet/, mobile/
  sections/01-header.png…    ← one crop per detected section
  hover/07-start-for-free-hover.png…
css/                         ← every stylesheet as served (+ inline <style> tags)
assets/images|fonts|icons|svg  ← everything downloaded, named after the source (/_next/image proxies resolved)
content/text.md              ← all copy in reading order, headings as markdown, CTAs tagged
content/dom-outline.txt      ← DOM tree with sizes and flex/grid hints
content/page.html            ← rendered DOM after hydration
pages/<slug>/                ← with --depth 1: the other pages linked from the nav
```

It handles what breaks naive captures: full scroll to trigger lazy-loading and IntersectionObservers,
cookie banners hidden via CSS (**never** clicking "accept"), pages too tall for Chromium,
`@font-face` URLs relative to cross-origin CSS, inline SVG logos, sites whose layout breaks after a
native full-page capture.

It also reads the original's console: errors, failed requests, HTTP ≥ 400, third-party hosts,
detected frameworks and libraries (Next, Nuxt, Webflow, Framer, Tailwind, GSAP, Lenis, Swiper…),
analytics.

### 2. The skill — rebuild in the right order

`SKILL.md` enforces the order that works: read the folds one by one → fonts + tokens + body +
container + header + meta → sections top to bottom with responsive written alongside → verify. With a
40-item checklist of what gives a clone away and ready-made recipes (sticky header, reveal-on-scroll,
marquee, FAQ, buttons with measured hover, `@font-face` from the manifest).

Output stack: static HTML/CSS/JS by default (most faithful), Next.js when you're in a Next.js repo,
whatever you ask for otherwise.

### 3. `compare.mjs` — measure instead of believing

```bash
node skills/copycat/scripts/compare.mjs --original copycat/example.com --clone http://localhost:4173
```

Loads the clone exactly like the original (same viewports, same scroll, fonts ready) and writes
`compare/REPORT.md`:

- **Grade A/B/C/D** and % of differing pixels per viewport (A = < 2 % with a clean console)
- **Worst zones**: the most different bands, mapped to sections → you know what to fix first
- **Design diff**: missing fonts, missing colours, h1/h2/h3/p/a/button typography, body, missing
  headings and CTAs, meta, motion counters, broken images
- **Clone console**: errors, exceptions, 404s — target 0
- `desktop-side-by-side.png` (original | clone | diff) and the clone's folds

Iterate to A. Usually three passes.

---

## Install

### Option 1, plugin (recommended)

In Claude Code:

```bash
/plugin marketplace add minosdevs/copycat
```

```bash
/plugin install copycat@minosdevs-copycat
```

### Option 2, manual

```bash
git clone https://github.com/minosdevs/copycat.git
cp -r copycat/skills/copycat ~/.claude/skills/
cp copycat/commands/copycat.md ~/.claude/commands/
```

### Either way: script dependencies (once)

```bash
cd ~/.claude/skills/copycat/scripts && npm install && npx playwright install chromium
```

(or the plugin's `scripts/` folder). Node ≥ 18. Installs Playwright, pixelmatch, pngjs and a
headless Chromium (~150 MB).

---

## Usage

```bash
/copycat https://example.com
```

Claude captures, reads the report and screens, builds, serves the clone, compares, fixes, and
delivers: where the clone is, the grade per viewport, the console, and an honest list of what isn't
reproduced.

Variants:

```bash
/copycat https://example.com --depth 1
```
also captures the pages linked from the nav.

```bash
/copycat https://example.com --stack next
```
forces Next.js components as output.

You can also run the scripts directly (options: `--viewports desktop,mobile`, `--scale 2`, `--dark`,
`--locale fr-FR`, `--wait 3000`, `--videos`, `--headed`, `--no-hover`, `--no-assets`).

### Without Playwright: the DevTools console

`skills/copycat/scripts/extract.browser.js` pastes as-is into any browser console, then:

```js
copy(JSON.stringify(__copycatExtract(), null, 2))
```

You get the same design-token JSON as `manifest.json → design`. Handy for sites behind a login or
bot protection.

---

## What it doesn't do

- No backend: forms, auth, payments, dynamic content, third-party widgets (chat, analytics) are not
  reproduced. The skill says so in the deliverable.
- No bypassing of bot protection or logins: it offers a manual capture path instead.
- Never clicks "accept" on a cookie banner: it hides it with CSS.
- No licence to reuse someone else's logo, photos, copy or brand. Cloning to learn, prototype or
  rebuild *your own* site: yes. Publishing someone else's site with their assets: no — the skill
  swaps brand assets for placeholders when the clone is for anything other than your own use or study.

---

## Layout

```
copycat/
├── .claude-plugin/          plugin.json, marketplace.json
├── commands/copycat.md      the /copycat command
├── skills/copycat/
│   ├── SKILL.md             the workflow (capture → read → foundations → sections → verify → deliver)
│   ├── scripts/
│   │   ├── capture.mjs      full Playwright capture
│   │   ├── compare.mjs      pixel diff + design diff + console
│   │   ├── extract.browser.js  the in-page extractor (also pasteable in DevTools)
│   │   ├── lib.mjs          shared helpers (scroll, folds, stitching, consent, PNG)
│   │   └── package.json
│   └── references/
│       ├── fidelity-checklist.md   the 40 details that look "off"
│       ├── rebuild-recipes.md      recipes: fonts, tokens, header, reveal, marquee, FAQ, buttons…
│       └── troubleshooting.md      blocked site, empty capture, 404 fonts, inner scroll, high diff
└── README.md / README.en.md / LICENSE
```

---

## Tested on

- Next.js + Tailwind v4 landing with reveal-on-scroll, fixed header, animated hero, auto carousel,
  delayed cookie banner, self-hosted fonts and proxied images (`/_next/image`): 3-viewport capture in
  ~90 s, 11 sections, 74 assets, 0 warnings; self-comparison of the site against itself at 1.6 %
  (infinite animations), grade A.

---

## Licence

MIT — [Minos](https://github.com/minosdevs).
