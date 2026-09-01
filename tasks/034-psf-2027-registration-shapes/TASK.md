# Task 034 — PSF 2027 registration form shapes

- **Status:** `in-progress`
- **Opened:** 2026-09-01
- **Owner:** Imtinan Almutairi
- **Sub-app(s):** backend | admin | frontend | docs
- **Branch(es):** `dev`

## Goal

Turn the client sheet `PSF 2027 Registration Sheet Requirement_13082026 - CAD comments.xlsx`
into working registration flows: two new guest form shapes (`psf-public-2027`,
`psf-private-2027`) rendered on the public site, creatable and editable in the admin,
visible in the guest "see more" modal, and carried into the guests export.

## Scope

- **In:** the sheet's `PUBLIC REGISTRATION PAGE 2027` and `PRIVATE PAGE 2027` tabs, end to end.
- **Out:** the `DEALS REGISTRATION PAGE 2027` tab — it is not a guest form (two signing
  parties, attendees, monetary values, a review workflow and its own dashboard). Specified
  separately in [`../035-psf-2027-deals/TASK.md`](../035-psf-2027-deals/TASK.md); nothing
  for it has been built.
- **Out:** the `HP` tab, which contains only the words "Home page".

## What the sheet asked for, and where it landed

### `psf-private-2027` — sheet "PRIVATE PAGE 2027" (1 step)

Rows 1–10 map onto columns that **already existed**, so this shape needed no migration.
The sheet's 2nd page (confirmation + QR) is the existing success page, not a step.

| Sheet row | Field |
|---|---|
| 1 Title | `title_id` |
| 2–3 First/Last name | `first_name`, `last_name` |
| 4 Gender | `gender` |
| 5 Company | `company` |
| 6 Position (badge free-text) | `job_title` |
| 7 Country | `country_of_residence` |
| 8 Email (work domain *preferred*) | `email` |
| 9 Phone | `phone` |
| 10 Picture | `personal_image` |

### `psf-public-2027` — sheet "PUBLIC REGISTRATION PAGE 2027" (2 steps)

Page 1 rows 1–9, page 2 rows 1–5. The sheet's 3rd page is the completion message —
the existing success page. **Six new columns** were required:

| Sheet row | New column | Type |
|---|---|---|
| p1 r5 Position (dropdown) | `position_level` | `string(30)` |
| p2 r2 Which **2** ecosystems | `ecosystems` | `json`, capped at 2 |
| p2 r3 Purpose of attendance | `attendance_purpose` | `string(50)` |
| p2 r3.1 (If other) description | `attendance_purpose_other` | `string(500)` |
| p2 r4 Accept PIF communications | `pif_communications_consent` | `string(3)` yes/no |
| p2 r5 Upload company profile | `company_profile` | `string`, private disk |

Page 1 row 5 and 5.1 are **two different fields** — a dropdown and a badge free-text.
Both are captured; `job_title` stays the badge text.

## Decisions

- **LOCKED — Deals is out of scope for this task.** It needs its own tables, review states
  and dashboard; folding it into `FORM_SHAPES_CONFIG` would have shipped a form that
  satisfies none of the sheet's rows B24–B32. → Task 035.
- **LOCKED — new shapes live under `froms/psf/`, `default/` untouched.** `FORM_RESTRUCTURE_GUIDE`
  says a clone renames `default/` to its own client name; that would have forced a data
  migration on every `categories.form_shape` already storing `default*`. A sibling `psf/`
  subtree costs nothing and leaves the three baseline shapes alone.
- **`ecosystems` is `json`, not the CSV `interests` uses.** The cap is on a real array, and
  `GuestsController::MAX_ECOSYSTEMS` is the copy that a crafted request cannot bypass (both
  form UIs also cap, for the UX).
- **"Company domain only" (p1 r7) is implemented as a free-provider denylist**
  (`frontend/utils/is-company-email.ts`). A domain cannot be *proven* to belong to a company;
  an allowlist would reject legitimate registrants. Deliberately **not** enforced in the admin
  form — staff adding a guest by hand need a way through — and **not** on the private shape,
  whose sheet asks only for a preference message.
- **PDF uploads were added narrowly.** Every upload path in the repo was jpg/png only
  (`UploadService::ALLOWED_EXTENSIONS`, `Base64Image`), so "upload your company profile"
  could not be submitted at all. New `Base64Document` rule + `UploadService::storeBase64Document()`,
  which is **private-disk-only** and always sniffs the decoded bytes. `storeBase64()` is
  byte-for-byte unchanged, so the public-disk callers that pass `imageOnly: false` keep the
  extension allowlist that is their only real control.
- **`BADGE_POSITION_MAX = 40` is ours, not the client's.** The sheet says only "very limited in
  character for badge purposes". Lives in `utils/psf-constants.ts` in both Next apps.

## Log

- 2026-09-01 — opened. Sheet parsed; the three tabs mapped against the schema. Scope and shape
  naming locked with the owner.
- 2026-09-01 — backend: migration `2026_09_01_000001_add_psf_2027_fields_to_guests_table`,
  `Guest` fillable + `ecosystems` array cast, `GuestsResources`, all three payload sites in
  `GuestsController` (public `store`, admin `storeGuest`, admin `updateGuest`),
  `GuestsExport` (**appended**, never inserted — the class formats by hardcoded column letter and
  `ExportColumnAlignmentTest` pins them), `FormShapeFieldsSeeder` catalogue, `company_profile`
  through both upload endpoints + `GuestDocumentController::TYPES`.
- 2026-09-01 — admin: 3 step components under `froms/psf/`, 3 read-only panels under
  `by-admins/psf/`, both COMPONENT_MAPs, `FORM_SHAPES_CONFIG`, `FormShapeTypeSelect`,
  ecosystem multi-select, `guests-props` (where_null/where_not_null) — `ecosystems` deliberately
  omitted there, see below.
- 2026-09-01 — frontend: 3 step components under `join/forms/psf/`, ecosystem checkbox group,
  company-email helper, `renderFormSteps` branches for the 2-step shape, `FORM_SHAPES_CONFIG`,
  COMPONENT_MAP. EN + AR landed with the code.
- 2026-09-01 — tests: `tests/Unit/PsfPublicFieldsTest.php` — 10 cases pinning the ecosystem cap
  (array + CSV input, dedupe, blanks → null), the purpose/other coupling, blank-to-null coercion,
  and that `GuestsExport::headings()` and `map()` stay the same width (the thing that would break
  the hardcoded column letters).
- 2026-09-02 — **verification pass over add / edit / export / see-more / mandatory-fields.**
  Found and fixed one real defect: **the guests listing has TWO export buttons and they hit
  different classes.** "Export all" → `/guests/export` + `/guests/export-filtered` → `GuestsExport`
  (had the PSF columns); **"Export guests" → `/guests/export-guests` + `/guests/export-filtered-guests`
  → `GuestsExportGuestView` (did NOT)**. Added the six columns there, and to `GuestsExportView`
  (`/export-view`, `/export-filtered-view` — live routes with no admin caller today, and it already
  carried `Industry`, so omitting its siblings would have been odd). Both appends;
  `GuestsExportGuestView` has its own hardcoded `COL_TITLE` D / `COL_EMAIL` I / `COL_PHONE` J /
  `DATE_COLS` M,T, `GuestsExportView` has none. The export tests are now parameterised over all
  three classes and assert both the width and the presence of every PSF heading, so a fourth export
  cannot be missed the same way.
- 2026-09-02 — the other four checked and correct, no change needed: **add** (admin create restores
  the `user_data_admin` cookie and validates `savedStep` against the shape's step list, so
  `psf-public-interests` resumes); **edit** (each step PUTs on its own; `updateGuest`'s `only()`
  carries all six keys; the purpose/other coupling preserves an existing description when step 1
  re-PUTs an unchanged `other`); **see-more** (`GuestsResources` serialises the whole `category`, so
  `form_shape` reaches `getStepsConfigForFormShape`); **mandatory fields** (no `form_shape` whitelist
  exists anywhere, so the new shapes save; the picker is fully data-driven off
  `/form-shape-fields/select` and re-fetches on `watch('form_shape')`; admin forms read
  `mandatory_fields_admin` with a fallback to `mandatory_fields`, the public forms read
  `mandatory_fields` — and all 15 `translationKey`s the two new shapes emit exist in EN and AR).
- 2026-09-01 — **full gate green.** Backend `pint --test` + `phpstan` (needs `--memory-limit=2G`;
  the 128M default crashes the worker) + `php artisan test` → **628 passed** (618 before, +10 mine).
  `ExportColumnAlignmentTest` re-run on its own and green, confirming the append kept
  `COL_TITLE`/`COL_EMAIL`/`COL_PHONE`/`DATE_COLS` pointing at the right headings. Admin and frontend:
  `yarn type-check` + `yarn build` both green; admin `yarn check:rbac` green (11 pre-existing warnings,
  no new pages added).

## Known gaps / follow-ups

1. **Nothing has been exercised in a browser or against a live API.** Everything is reasoned
   from source with the static gate green. The click-list is in "Verification still owed" below.
2. **The migration has only been exercised on sqlite.** The test suite runs `:memory:` sqlite with
   `RefreshDatabase`, so the migration ran 39 times — but sqlite ignores `->after()` and stores
   `json` as `text`. There is no MySQL on this machine, so column *placement* and the real `json`
   type are unverified against the production driver. Run `php artisan migrate` on a MySQL dev DB
   before trusting it.
3. **`ecosystems` is not offered in the `where_null` / `where_not_null` filter.** Those apply
   `whereNull($f)->orWhere($f, '')`, and comparing a MySQL `json` column to `''` raises
   "Invalid JSON text". Needs a json-aware branch in `GuestsController::index()` first.
4. **Photo background is not detected.** The sheet (p1 r9) asks for a "Simple detection solution
   to reject photos" that are not on a white background. Not built — the requirement is stated
   in the upload note and checked by a human. Needs its own decision (client-side heuristic vs.
   an admin review step).
5. **`FormShapeFieldsSeeder` must be re-run** for the two new shapes to appear in the admin's
   mandatory-fields picker: `php artisan db:seed --class=FormShapeFieldsSeeder`.
6. **The Arabic copy is a first pass** and has not been reviewed by a native speaker/the client.

## Verification still owed (browser)

- [ ] Category with `form_shape = psf-private-2027` → register end to end → guest row, success + QR.
- [ ] Category with `form_shape = psf-public-2027` → page 1 → page 2 → submit; back button keeps page 1.
- [ ] Ecosystems: third checkbox locks at 2; un-ticking one unlocks the rest.
- [ ] Purpose = Other reveals the description and requires it; switching away clears it.
- [ ] Company profile: upload a **PDF** and a PNG; both stream back through the signed URL.
- [ ] Public form rejects a gmail.com address; admin form accepts one.
- [ ] Admin: create + edit on both shapes; see-more tabs render every field.
- [ ] Export: check **both** listing buttons — "Export all" *and* "Export guests" — carry the six
      new columns, and that `Title`/`Email`/`Phone`/date columns are still formatted correctly
      (the alignment tests cover this, but confirm on a real workbook).

## Definition of Done

- [x] Code written in backend / admin / frontend
- [x] EN + AR translations in the same change
- [x] Next apps `yarn type-check` green; admin `yarn check:rbac` green
- [x] Next apps `yarn build` green
- [x] Backend `composer qa` green — pint + phpstan + 635 tests (17 new)
- [ ] `FormShapeFieldsSeeder` re-run on the target environment
- [ ] `php artisan migrate` run against **MySQL** (only sqlite so far — see gap 2)
- [ ] Browser verification (list above)
- [ ] Docs: this TASK.md set to `done`; index row updated
- [ ] Mobile contract checked — `routes/api.php` was **not** touched, but `GuestsResources`
      gained six keys; confirm the mobile consumer tolerates additive fields
      (`../../mobile/RESPONSE_SHAPE_DELTAS.md`)
