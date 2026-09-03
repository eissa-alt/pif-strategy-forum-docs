# Task 037 — Trim the admins form: phone and Highlighted admin

- **Status:** `done (code)` — manual browser QA pending
- **Opened:** 2026-09-03
- **Owner:** Eissa
- **Sub-app(s):** backend + admin
- **Branch(es):** `dev`

## Goal

Take two inputs off the admins form at the owner's request — the phone number and the
"Highlighted admin" (`is_vip`) switch — and drop the phone column from the admins listing.
Hide them from the UI only: the columns, the model and the API responses stay, so no
stored value is lost and the change is reversible without a migration.

## Scope

- **In:** the shared `admins-form.tsx` (create **and** edit), the `admins-listing.tsx` phone
  column, and the `phone` validation rules in `AdminsController`.
- **Out (owner decision):** the `admins.phone` column, `Admin::$fillable`, `AdminsResources`
  and `AuthController::profile()` — all untouched. `phone` keeps being returned by the API;
  nothing else is reading it in the admin app today.
- **Out:** the Seating Plan Manager, which still consumes `is_vip` to highlight an admin's
  seating changes. The flag survives; only the switch that set it is gone.

## Decisions

- **Both fields removed from the shared form, not hidden per mode.** `admins-form.tsx` serves
  create and edit from one component. A `mode === 'create'` conditional would have left an admin
  created without a phone unable to be saved from the edit form, which still demanded one — and
  CLAUDE.md hard rule 3 bans dual code paths. Owner picked the full removal.
- **`phone` validation relaxed to `nullable`, and this was forced, not optional.** `store` had
  `'phone' => 'required|string|max:100'` (Task 024), so a form that no longer sends the key
  would have 422'd on every create. `update` went `sometimes|required` → `sometimes|nullable`
  for the same reason. **This reverses the Task 024 decision** "required on create *and* edit",
  which the owner had asked for at the time.
- **`is_vip` needed no backend change** — it was already `nullable|boolean` in both `store` and
  `update` (Task 033, P033.54).
- **`is_vip: false` stays in the form's `defaultValues`,** commented. It keeps create sending an
  explicit `false` exactly as before, and edit round-trips the stored value through
  `reset(d)` + the `...values` spread, so an existing highlighted admin is not silently cleared
  by an unrelated edit.
- **No translation work.** The form reused the shared `web:phone` label and `web:is_vip`
  ("Highlighted admin"); both keys stay, `web:phone` because the guest forms still render it.

## Log

- 2026-09-03 — opened and delivered. Backend `P037.1`: `AdminsController` store + update phone
  rules. Admin `P037.2`: phone field, `is_vip` switch, phone listing column, the two now-unused
  imports (`PhoneInputV2`, `libphonenumber-js`) and both keys in the 422 `setError` union.
  Row 3 of the form is back to `Email 1/2 + Status 1/2`, which is what its own comment described.
  Gates green: backend `pint --test` + `phpstan` + **640 tests**; admin `yarn type-check` +
  `yarn build` + `yarn check:rbac`.

## Definition of Done

- [x] Code merged to `dev` in the relevant sub-app(s)
- [x] EN + AR translations — none needed (no new or removed user-facing strings)
- [x] Quality gate green (backend `pint --test` + `phpstan` + 640 tests; admin `yarn type-check`
      + `yarn build` + `yarn check:rbac`)
- [x] Docs updated (this TASK.md; index row; Task 024 log line recording the reversal)
- [x] Mobile contract unaffected — admin CRUD is `/admin/admins`, not `mobile/*`;
      `routes/api.php` untouched
- [ ] Manual browser QA — create an admin with no phone, edit an existing one and confirm its
      stored phone and `is_vip` survive the save, and check the listing columns
