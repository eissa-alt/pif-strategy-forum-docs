# Task 024 — Add phone number to admins

- **Status:** `done (code)` — **superseded in part by [Task 037](../037-admins-form-trim/TASK.md)**
- **Opened:** 2026-07-27
- **Owner:** —
- **Sub-app(s):** backend + admin
- **Branch(es):** `dev`

## Goal

Give admins their own phone number, captured on create/edit and shown in the listing, using the
**same field name and input widget as guests** so the two stay consistent.

## Scope

- In: DB column, model, store/update validation + persistence, API resource + profile response,
  create/edit form field, listing column, TS interface.
- Out (explicitly deferred): a **phone search/filter** on the admins listing (needs a backend
  `index` filter), and an **admins export** (none exists today — guests have exports, admins don't).

## Decisions

- **Column name is `phone`, not `phone_number`.** Guests use `phone` everywhere (DB, API, forms), so
  admins match it. Owner confirmed "use what we have for guest to be consistent." (A commented `phone`
  placeholder already sat in `AdminType`, and unused `admin_phone` labels already sat in EN/AR — both
  vestiges of an earlier intent; we reused the shared `web:phone` label instead.)
- **DB column nullable; "required" enforced at the request/form layer.** Existing admins predate the
  field, so a NOT-NULL column is impossible without a destructive reset (`migrate:fresh` is banned).
  `store` validates `required`; `update` validates `sometimes|required` (tolerates a partial update that
  omits it, but rejects an empty value). Consequence, by owner's choice (required on create **and**
  edit): editing a pre-existing admin now forces entering a phone before it can be saved.
- **Widget = `PhoneInputV2`** (country-flag picker + libphonenumber E.164 validation), the same
  component the guest forms use — not the plain `CustomInput` the other admin fields use. Owner's call,
  for consistency and real phone validation.
- Two small create-form defaults landed in the same admin commit at owner request: **status defaults
  ON** and the **Data scope** section **starts expanded**. Edit mode is unaffected (`reset()` loads the
  real record).

## Log

- 2026-07-27 — opened. Backend `P024.1` (`f1c3dc3`): migration `2026_07_27_000002`, `Admin` `$fillable`,
  `AdminsController` store/update validation + persist, `AdminsResources` + `AuthController::profile()`.
  Admin `P024.2` (`f743fae`): `PhoneInputV2` field on the form, listing column, `AdminType` typed,
  create defaults, and a shared-component fix — `PhoneInputV2` error text was `text-error` (undefined in
  this app's Tailwind v4 theme → rendered gray), switched to `text-red-500`; also fixes the guest admin
  forms that share the component. Dev DB migrated (`php artisan migrate`, not `fresh`). Ledger D45.

- 2026-09-03 — **the form field is gone again.** [Task 037](../037-admins-form-trim/TASK.md) removed the
  phone input from `admins-form.tsx` (create *and* edit) and the column from the listing at owner
  request, which forced `store` / `update` validation from `required` / `sometimes|required` to
  `nullable`. **The "required on create and edit" decision above no longer holds.** Everything else
  this task built — the migration, the column, `$fillable`, `AdminsResources`, the profile response —
  is untouched and still carries the stored numbers.

## Definition of Done

- [x] Code merged to `dev` in the relevant sub-app(s)
- [x] EN + AR translations reused (`web:phone`, `validation:invalid_phone`) — no new keys
- [x] Quality gate green (backend `pint --test` + `phpstan` + 489 tests; admin `yarn type-check` + eslint)
- [x] Docs updated (this TASK.md; index row; ledger D45; handoff)
- [x] Mobile contract unaffected — admin CRUD is `/admin/admins`, not `mobile/*`; `routes/api.php` untouched
- [ ] Manual browser QA — create + edit an admin, verify persistence, listing column, and the (now red) required-phone error
