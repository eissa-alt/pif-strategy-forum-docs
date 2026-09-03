# Task 039 — Mandatory-field catalogue back in the admin bundle

- **Status:** `done (code)` — manual browser QA pending
- **Opened:** 2026-09-03
- **Owner:** Eissa
- **Sub-app(s):** admin + backend
- **Branch(es):** `dev`

## Goal

Stop relying on `FormShapeFieldsSeeder`. Put the per-shape mandatory-field lists back in the
admin bundle behind `getMandatoryFieldsByFormShape`, exactly as the sibling clones do
(`123-pif-pep-v2`), and remove the seeded table, its endpoint and its model.

**This reverses [Task 033](../033-fork-port-back/TASK.md)'s `form_shape_fields` delivery**
(Imtinan Almutairi, 2026-08-28 — backend `ad96ae1`, admin `f3563cc`). Owner's call.

## Why

The catalogue being *data* meant every environment needed
`php artisan db:seed --class=FormShapeFieldsSeeder` after any list change, or its picker
silently showed a stale set. That bit us the same day: [Task 038](../038-psf-2027-gender-title/TASK.md)
added `title_id` to the public shape and the option only appeared after a re-seed. In the
bundle the list ships with the build and cannot go stale per environment.

## Scope

- **In:** new `admin/data/mandatory-fields-by-form-shape.tsx`; `mandatory-fields-select-multi.tsx`
  rewritten to read it; backend model, controller, seeder, routes and tests deleted; a migration
  dropping `form_shape_fields`.
- **Out (owner decision):** any replacement drift guard. Offered a `scripts/check-form-fields.mjs`
  mirroring `check-rbac-map.mjs`; owner chose to match pep-v2 exactly — review only.
- **Out:** backend validation of mandatory fields. Not in scope and unchanged either way — see
  "What the backend actually validates" below.

## Decisions

- **Parity was proved before anything was deleted.** The bundled list was diffed against the live
  seeded catalogue per shape: `default` 9/9, `default-one-step-rsvp` 9/9 (aliases default, as the
  seeder did), `default-four-steps` 30/30, `psf-public-2027` 16/16, `psf-private-2027` 10/10 —
  **identical, order included**. `full_name_on_document` (the one key PSF had that pep-v2 never
  did) and `title_id` from Task 038 both survive.
- **Full removal, not a dead table.** Model, controller, seeder, `DatabaseSeeder` entry, the two
  routes and `FormShapeFieldsTest` are gone, and `2026_09_03_000001` drops the table. Leaving it
  in place would have meant two sources for one list — the state that caused this whole
  investigation. `down()` recreates the schema faithfully but **cannot repopulate**: the seeder
  that knew the rows goes with it.
- **The seeder's docblock knowledge was carried across, not deleted.** Chiefly the RSVP caveat:
  `default-one-step-rsvp` offers a copy of the default nine, which is wrong — that form registers
  only `will_attend` and `dietary_requirements` and never reads `mandatoryFields`. Kept as a copy
  so behaviour is unchanged, with the correction written out in the comment.
- **`PsfPublicFieldsTest`'s catalogue assertion was deleted, not rehomed.** The admin app has no
  test runner (`build`, `type-check`, `check:rbac`, `lint`), so there is nowhere to assert a
  frontend list against the forms. Accepted with the "no guard" decision.
- **The endpoint was not in the mobile contract** — `docs/mobile/BACKEND_INCOMING_CHANGES_FOR_MOBILE.html`
  greps clean for `form_shape` / `form shape`, and the routes were `admin/`-prefixed. CLAUDE.md
  hard rule 4 satisfied.
- **Removing it also closed a flagged exposure.** `admin/form-shape-fields` sat in the plain
  `auth:sanctum` group with no `admin.can` gate — readable by any sanctum token including a
  mobile guest's, recorded in docs `aa00b54` as an owner call. The route no longer exists.

## What the backend actually validates (established this session, unchanged by this task)

- **`extra_guest_mandatory_fields` IS enforced** — `GuestsController` loops the related guests and
  422s with `related guest #N missing required field: X`. **PSF and pep-v2 are byte-identical here.**
- **`mandatory_fields` and `mandatory_fields_admin` are NOT enforced.** They are stored
  (`CategoriesController` `json_encode`s the input with **no validation rule at all**) and read
  back out. For the main guest, "mandatory" lives entirely in the frontend. Posting the API
  directly without a field marked mandatory saves fine.
- **Nothing validates the field *keys*** against the shape, in either project. That is why a
  category switched between shapes could hold keys the new form never renders.

This is the `[~]` recorded on Task 033. It predates the seeded table, survives its removal, and is
the same in pep-v2 — so on this axis the change is exactly neutral.

## Known gap, deliberately not fixed

`attendance_purpose_other` is rendered by `psf/public/step-2` and is absent from the list, so it
cannot be marked mandatory. Left alone: the form already requires it whenever the box is visible.

## Log

- 2026-09-03 — opened and delivered. Admin `P039.2`: `data/mandatory-fields-by-form-shape.tsx`
  added, `mandatory-fields-select-multi.tsx` rewritten (no `useFetch`, no loading/error branch).
  Backend `P039.1`: `FormShapeField`, `FormShapeFieldController`, `FormShapeFieldsSeeder`
  (+ `DatabaseSeeder` entry), 2 routes + import, `FormShapeFieldsTest` and the
  `PsfPublicFieldsTest` catalogue case all removed; migration `2026_09_03_000001` drops the table.
  **Migration run on the dev DB** — table confirmed gone. Gates green: pint + phpstan +
  **632 tests** (640 → 632: `FormShapeFieldsTest` plus the catalogue case); admin `yarn type-check`
  + `yarn build` + `yarn check:rbac` + eslint.

## Definition of Done

- [x] Code merged to `dev` in the relevant sub-app(s)
- [x] EN + AR translations — none needed; the same `web:*` keys move across unchanged
- [x] Quality gate green (backend `pint --test` + `phpstan` + 632 tests; admin `yarn type-check`
      + `yarn build` + `yarn check:rbac`)
- [x] Docs updated (this TASK.md; index row; Task 033 and Task 038 corrected)
- [x] Mobile contract checked — endpoint absent from the contract doc; `routes/api.php` lost two
      `admin/`-prefixed routes only
- [ ] **`php artisan migrate` on every other environment** to drop the table there too
- [ ] Manual browser QA — categories edit on all five shapes: the picker lists the right options
      with no loading flash, and saving keeps the selections
- [ ] Tell Imtinan Almutairi, whose 2026-08-28 delivery this reverses
