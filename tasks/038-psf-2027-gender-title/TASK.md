# Task 038 — PSF 2027: gender leads, and Title on the public shape

- **Status:** `done (code)` — manual browser QA pending
- **Opened:** 2026-09-03
- **Owner:** Eissa
- **Sub-app(s):** frontend + admin + backend
- **Branch(es):** `dev`

## Goal

Two ordering/field changes to the PSF 2027 registration pages, found while testing them:
gender is asked first on both shapes, and the public (attendee) shape gains the courtesy
Title the private (guest) shape already had.

- `/en/join/guest` (`psf-private-2027`) — gender to the top, then Title.
- `/en/join/attendee` (`psf-public-2027`) — gender to the top, then a **new** Title field.

## Scope

- **In:** both public form shapes, the `title_id` catalogue entry for `psf-public-2027`, and the
  admin surfaces that have to show the new answer (create/edit form + see-more panel).
- **Out:** admin form field *ordering*. The admin trees still follow the sheet (names, then
  gender); only the public site leads with gender. Owner asked about the two public pages, so
  the admin forms gained Title but were not reordered. Flagged as known drift.
- **Out:** the `default*` shapes and the private shape's field list — untouched.

## Decisions

- **Gender first is functional, not only cosmetic.** `TitleSelectNew` filters its options by
  `gender` (`gender={watch('gender')}`). Asking for gender first is the order that makes the
  Title picker correct the first time it is opened, rather than after a correction.
- **`title_id` on the public shape is a deliberate departure from the client sheet.**
  "PUBLIC REGISTRATION PAGE 2027" does not list a title; the owner asked for it so both PSF
  shapes capture the same courtesy title. Recorded in the form docblock, the seeder docblock
  and a test assertion so it does not read as an accident later.
- **Sheet-row numbers in the comments still refer to the sheet, not to display order.** Both
  form docblocks now say so explicitly — the numbers were the only map back to the client
  document and renumbering them would have destroyed it.
- **No backend controller change was needed.** `title_id` is persisted generically
  (`'title_id' => $request['title_id']`), not gated per shape, so the public shape could already
  store it — the frontend was in fact already sending it, and already declared a `guestTitleId`
  prop that nothing read. Both are now wired up, so prefill / complete-data links carry it.
- **The admin had to change or the answer would be write-only.** `title_id` was absent from the
  admin create/edit form *and* the see-more panel for this shape, so a title a registrant chose
  would have been invisible and uneditable. The admin tree uses `TitleSelect` (no gender filter),
  matching its sibling private form rather than the public site's widget.
- **Catalogue order mirrors the forms.** `sort_order` drives the admin mandatory-field picker;
  both PSF lists now lead with gender so the picker reads in the same order as the page.

## Log

- 2026-09-03 — opened and delivered. Backend `P038.1`: `FormShapeFieldsSeeder` — `title_id` added
  to `PSF_PUBLIC_FIELDS`, both PSF lists reordered gender-first, plus an assertion in
  `PsfPublicFieldsTest` pinning `title_id` on the public shape. Admin `P038.2`: Title on the
  public create/edit form and the see-more panel (which also needed `lang` destructured).
  Frontend `P038.3`: both shapes reordered, Title added to the public form with its import,
  prop destructuring and prefill. **Seeder re-run on the dev DB** — 74 rows across 5 shapes,
  public shape verified as `gender, title_id, first_name, last_name, company, position_level`.
  Gates green: backend `pint --test` + `phpstan` + **640 tests**; admin + frontend
  `yarn type-check` + `yarn build`; admin `yarn check:rbac`.

## Definition of Done

- [x] Code merged to `dev` in the relevant sub-app(s)
- [x] EN + AR translations — none needed; `web:title` already exists and was already used by the
      private shape
- [x] Quality gate green (backend `pint --test` + `phpstan` + 640 tests; both Next apps
      `yarn type-check` + `yarn build`; admin `yarn check:rbac`)
- [x] Docs updated (this TASK.md; index row)
- [x] Mobile contract unaffected — no route added, removed or renamed; `routes/api.php` untouched
- [ ] **`php artisan db:seed --class=FormShapeFieldsSeeder` on every other environment.** Done on
      the local dev DB only. Without it `title_id` is not tickable as mandatory on the public
      shape and the picker order is stale.
- [ ] Manual browser QA — both pages in EN **and AR**, the Title picker re-filtering when gender
      changes, and a public registration round-tripping a title into the admin see-more panel
