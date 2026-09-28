# Task 043 — PSE breakout sessions (registration page 2)

- **Status:** `in-progress`
- **Opened:** 2026-09-28
- **Owner:** Imtinan Almutairi
- **Sub-app(s):** backend | admin | frontend | docs
- **Branch(es):** `dev`

## Goal

Add a second registration page to the PSE shapes on which the guest picks their top two
breakout sessions, in order of priority. This is the partners forum's "session preferences" question,
cut down to three sessions and asked with a lighter control.

## Scope

- In: both PSE shapes (`pse-one-step-with-img`, `pse-one-step-no-img`), which share page 2.
  The sessions are:
  - `third_party_capital`: Mobilizing Third Party Capital: From funding to partnership
  - `synergy_by_design`: Synergy by Design: Unlocking value across the PIF portfolio
  - `data_driven_performance`: Strategy Execution Through Data-Driven Performance
- Out: session descriptions (the client supplied titles only), capacity limits, and
  assigning guests to sessions.

## Decisions

- **Columns `breakout_session_1` / `breakout_session_2`**, the same names as the partners
  forum's, holding keys. Titles live in `Guest::BREAKOUT_SESSIONS` (backend) and the
  translations. Both Next apps carry the keys in `data/pse-breakout-sessions.tsx`, and all
  three copies must stay in step.
- **Shape keys are not renamed.** They still say `one-step`, because `categories.form_shape`
  stores the key.
- **Public UI: tap-to-rank cards, not two dropdowns.** With three sessions, two dropdowns
  would show the same list twice. The first tap is 1st priority, the second tap is 2nd,
  and tapping a ranked card removes it. Removing the 1st moves the 2nd up. Admin keeps
  two plain selects.
- **Backend normalizes every write** (`GuestsController::breakoutSessionFields`). It drops
  unknown keys, drops a session that appears twice, and moves a lone 2nd priority up. This
  applies to public store, admin create, admin edit (only when sent) and the complete-data
  link.
- **Required is decided per category** through mandatory fields (`breakout_session_1`,
  `breakout_session_2`). Neither field is forced.
- **Consents stay on page 1**, where they were.
- **"Synergy by Design; unlocking…" was punctuated with a colon** to match the other two
  titles. Revert the change if the client wants the original.

## Log

- 2026-09-28: implemented end to end.
  - Backend: migration `2026_09_28_000001_add_breakout_sessions_to_guests_table`, model
    fillable plus the list, controller (4 write paths), both resources, and all 3 guest
    exports (appended, English titles).
  - Frontend and admin: step 1 now advances instead of submitting (OTP stays on step 1),
    plus the shared step 2, the see-more panel, the mandatory catalogue, the null filters,
    and EN + AR text.
  - Gate: backend pint, phpstan and **752 tests** (11 new in
    `tests/Feature/PseBreakoutSessionsTest.php`). Both Next apps pass type-check and
    build. Admin passes `check:rbac`.
- 2026-09-28: local env files added (admin/frontend `.env.local`, backend `.env`), laid
  out like PEP v2's. All three are gitignored.

## Verification still owed (browser)

- [ ] Public, both shapes: page 1 → Next → rank → Submit, then check that the guest row
      holds both keys. Back from page 2 should keep page 1.
- [ ] OTP-enabled category: the verify modal appears on page 1 and then advances to page 2.
- [ ] The complete-data link re-opens with the ranking already shown.
- [ ] Admin create and edit on both shapes, see-more page 2, and both export buttons.
- [ ] Arabic copy reviewed by the client.

## Before this runs anywhere

`php artisan migrate`. Then tick `breakout_session_1` (and optionally `_2`) as mandatory on
the PSE categories if a choice must be made.
