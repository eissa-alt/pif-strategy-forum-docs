# Task 036 — PIF / PSF26 branding on the registration site

- **Status:** `in-progress`
- **Opened:** 2026-09-02
- **Owner:** Imtinan Almutairi
- **Sub-app(s):** frontend
- **Branch(es):** `main`

## Goal

Put the public registration site on the real PIF brand: the palette from the PIF Brand
Manual and the PSF26 lockup, replacing the basecode's placeholder colours and a logo left
over from a different event.

## Scope

- **In:** `-frontend` palette, header logo, page background, favicon, SEO share card.
- **Out (owner decision):** `-admin`. It is an internal tool, still on Inter / NotoKufiArabic
  and its own theme; the brand manual governs public-facing material.
- **Out:** the Fund webfont — **already done**. All four weights were installed at
  `public/fonts/Fund/` and wired in `css/fonts.css` at 300/400/500/600, byte-identical to the
  supplied `Fund v2/` files and matching the manual (p29–31) exactly. Nothing to change.

## Sources

- `Brand Guidelines/PIF - Brand Manual - 2024 1119.pdf` — palette p33–34, typography p29–31.
- `Brand Guidelines/PIF PSF26 _ Logo Variations.pdf` — three lockups (dark green / gold / cream).
- `Fund v2/` — the four TTFs.

> `Brand Guidelines/` and `Brand Guidelines 2/` are **byte-identical duplicates** (same MD5 on
> both files). One of them can be deleted; kept both for now since they are client-supplied
> and `docs/ai/AI_RULES.md` rule 8 says not to touch client asset folders.

## What changed

### Palette — `tailwind.config.js`

| Token | Was (basecode) | Now | Source |
| --- | --- | --- | --- |
| `primary` | `#FF0A5A` pink | **`#004D43`** PIF Green | manual p33 |
| `secondary` | `#1EC8A0` teal | **`#C4984F`** PIF Gold | manual p33 |
| `accent` | `#FB9239` orange | **`#02CD9A`** bright green | manual p34 |
| `brand.*` | `mint: #6BCCDE` | `green` / `gold` / `green-dark #00342B` / `mint #8AFFE2` / `cream #FFFCF2` | manual p33–34 |
| `error` | — | **unchanged** | semantic, not brand |

`500`/DEFAULT is the documented brand value. Lighter steps are tints blended toward white —
the only tinting the manual permits ("blend a color with white… restricted to 100%
saturation"). Darker steps blend toward black so hover/active states are actually
distinguishable; `primary-700` is pinned to the manual's own dark green `#00342B`.

> Blending the primary's dark end toward `#00342B` instead of black was tried first and
> **rejected**: `#004D43` and `#00342B` are close enough that 600–950 all collapsed to within a
> few units of 500, which would have made `hover:bg-primary-600` invisible.

### Logo — `public/images/logo-psf26-on-dark.{svg,png}`

Extracted from page 1 of the logo PDF (top / dark-ground variation) as **vector**, its
background rect removed so the asset is transparent and sits on any dark ground. The header
sits on a black `body`, so this is the correct variation.

- The PDF renders gold colour-shifted (`#BF933E`); normalised to the brand `#C4984F`.
- A **PNG is what the header loads**, not the SVG: `next.config.js` does not set
  `dangerouslyAllowSVG`, so `next/image` would refuse the SVG. The SVG ships beside it as the
  vector source.
- Header box resized `h-[50px] w-[80px]` → `h-[38px] w-[166px] sm:h-[46px] sm:w-[201px]`. The
  lockup is **4.364:1**; in the old near-square box `objectFit="contain"` would have
  letterboxed it to about a third of its usable size.

### Background — the site was wearing Tourise's

`public/images/pattern.jpg`, `pattern-page-2.png` and `pattern-page.png` were
**byte-identical** to `Tourise/tourise-frontend`'s tiled geometric motif, and
`layout/index.tsx` tiled one of them behind every page.

Replaced with the PEP v2 setup, which is the reference PIF build: `pif-pattern.png`
(the PIF "Ripple Effect", manual p19-21), `body` ground `#00231c`, and the ripple
restored in the footer bled off the right edge (`bg-no-repeat` is load-bearing —
repeated it becomes a tiled band). The four Tourise files are `git rm`'d.

### Accent — the mint form controls

`accent` was `#02CD9A`, the manual's bright mint. It is a legitimate PIF accent for
graphics, but on radio dots, checkbox fills and focus rings it read as another
client's palette. Now `#BD9B60`, taken from PEP v2, so all three tokens match it
exactly: `primary #004D43`, `secondary #C4984F`, `accent #BD9B60`.

### SEO share card — the worst one

`public/seo/seo-v3.jpg` was a **full TOURISE 2027 poster**, byte-identical to
Tourise's, and `_app.tsx` served it as the `og:image` — so every shared link
previewed the wrong event. Replaced with `psf-2027-og.jpg`, composed from the real
assets (PSF26 lockup on `#00231c` with the ripple, 1200x630, the OG spec).
`seo.jpg` was identical to Tourise's too; both removed.

`translations/*/meta.json` still said "Registration Portal" — the text that sits
next to that card in a preview. Now the PSF 2027 name, EN and AR.

### Favicon — it was another event's

`favicon.ico`, `favicon.svg`, `favicon-96x96.png`, `apple-touch-icon.png` and both
`web-app-manifest-*.png` were an **"E|G"** mark shared with Tourise and
static-basecode, so every browser tab showed the wrong logo. Rebuilt from the PIF
emblem inside the PSF26 lockup on the PIF green ground, rendered at 16/32/48/64
(ico), 96, 180 and 192/512.

`favicon.tsx` also hardcoded `theme-color: #2D2E80` (a basecode blue) and
`msapplication-TileColor: #da532c` (an orange), and an `"app-name"` placeholder —
all three corrected.

### Em dashes removed (user request)

32 translation strings across both apps, EN and AR, with per-sentence
replacements rather than a blind swap — an em dash in "JPG or PNG - max 5 MB"
wants a comma, "optional - you can set them later" wants a full stop. Plus 49
`'-'` placeholder literals in 16 admin files normalised from the em dash to the
hyphen the other 271 usages already used, and the category names. Code comments
were deliberately left alone: not user-facing, and a large pointless diff.

## Verification

Contrast measured (WCAG 2.1), not eyeballed:

| Pair | Before | After |
| --- | --- | --- |
| primary text on a white card | 3.86:1 (AA-large) | **9.80:1 (AAA)** |
| white text on a primary button | — | **9.80:1 (AAA)** |
| secondary on the black body | 9.82:1 | **7.95:1 (AAA)** |
| logo on the black body | — | **21:1 / 7.95:1 (AAA)** |

The swap inverts primary from a bright pink to a very dark green, so every `text-primary` was
checked for a dark ground. All 67 sit inside `bg-white` cards — contrast **improves**. The two
that looked risky were cleared: `components/layout/styles/header.module.css` is entirely
commented out and imported nowhere, and `registration-closed-sections.tsx` is on a white card.

`accent` on white is only 2.06:1, but none of its 7 usages is text on white — they are
`form-checkbox` / `form-radio` control fills and the step menu, which sits on the dark ground.

Gate: `yarn type-check` + `yarn build` green.

## Known gaps

1. **Not verified in a real browser.** The logo was rendered and inspected against `#004D43`
   and `#000000`, and contrast is computed, but no page has been loaded.
2. **`public/images/logo-white.png` and `eg-logo.png` are now unreferenced** — leftovers from
   another event. Left in place rather than deleted; safe to remove once someone confirms no
   email template or static export points at them.
3. **Only the dark-ground lockup exists as an asset.** If a light/cream surface ever needs the
   logo, extract the cream variation from page 1 the same way.
4. The Arabic and English wordmarks are baked into one lockup, as the client supplied it —
   there is no separate RTL asset, and none appears to be needed.

## Definition of Done

- [x] Palette from the brand manual, with sourced comments
- [x] PSF26 lockup installed and wired into the header at the correct aspect ratio
- [x] Contrast checked across the inverted primary
- [x] `yarn type-check` + `yarn build` green
- [ ] Browser check of the header, homepage and a registration form
- [ ] Owner sign-off that the dark-ground lockup is the intended default
- [ ] Decide whether to delete the stale `logo-white.png` / `eg-logo.png`
