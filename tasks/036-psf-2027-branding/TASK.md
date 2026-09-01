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

- **In:** `-frontend` palette + header logo.
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
