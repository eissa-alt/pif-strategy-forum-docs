# Task 044 — pif-strategy registration shape

- **Status:** `in-progress`
- **Opened:** 2026-10-08
- **Owner:** eissa-alt
- **Sub-app(s):** backend | admin | frontend | docs
- **Branch(es):** `dev`

## Goal

A registration form shape, `pif-strategy`, for the **PIF Partners' Series Strategy Forum**, on this
otherwise Internal Audit Community Forum–branded site. It is `pse-one-step-with-img` (page 1 guest
fields + badge photo, page 2 breakout sessions ranked) with the client's three changes.

## Scope

- In, on the new shape only:
  - **Photo examples cut to two:** `good` (accepted) and `background` (refused). The five-tile row
    read as overpowering.
  - **A description under each breakout session** (`web:breakout_<key>_desc`). These hold the
    client's text; the third gets a closing full stop.
  - **The forum brief on the success page:** four paragraphs, under "About the Forum".
  - **Strategy Forum branding:** `StrategyForum@pif.gov.sa` on both pages, and a photo-consent
    read-more that names the forum.
- Out:
  - The two PSE shapes are byte-for-byte unchanged, except the default-valued props below.
  - The tab title and share card stay site-wide (IAC).
  - Admin gets no new screens.
  - Session titles are unchanged. The client wrote "Synergy by Design; unlocking…"; the existing
    title, shared with PSE, is kept.
  - The PSE with-img page 1 still overrides its contact email to `StrategyForum@`. Whether it goes
    back to `IAC@` is the owner's call.
  - The complete-data link redirects to `/${lang}`, not to the success page, so it never shows the
    brief. That is existing behaviour.

## Log

- 2026-10-08:
  - Opened. Four scope questions settled with the owner, the recommended option each time:
    1. new shape only;
    2. the brief only on the success page;
    3. keep `good` + `background`;
    4. email + consent text.
  - Implemented in all three apps. Committed on `dev`: backend `d6c878f`, admin `1741030`,
    frontend `bc99a65`.
  - Gates:
    - **Admin:** `type-check`, `build`, `check:rbac` (11 existing warnings), eslint and prettier on
      the changed files: all pass.
    - **Frontend:** `type-check`, `build` (the only warning is the existing face-api one), eslint
      and prettier on the changed files: all pass.
    - **Backend:** `pint --test` and `phpstan` are clean; the new `CategoryVisibilityTest` passes
      3/3.
  - ⚠️ The **full backend suite is 37 failed / 718 passed**. Every failure is in `DealSubmissionTest`
    (23) and `DealSlotBookingTest` (14), and each is a 403. Those tests post `recaptcha: 'x'` without
    faking Google, and `App\Rules\ReCaptcha` only skips verification in `local`, so under `testing`
    it calls siteverify for real and fails. The failures are unrelated to this task: none of those
    tests calls the visibility endpoint. Task 043 recorded the suite green on 2026-09-28, so
    something in the local run has changed since. It needs its own fix, e.g. `Http::fake` of
    siteverify in those two tests.

## Decisions

- **The frontend copies, the admin reuses.**
  - The public steps are copies, as in P1.3 for the PSE shapes:
    `components/join/forms/pif-strategy/with-img/step-1.tsx` and `…/breakout-sessions/step-2.tsx`.
    Their step ids are `pif-strategy-info` and `pif-strategy-breakout-sessions`, so the PSE
    components stay as they were.
  - Admin has none of the differences, so its `pif-strategy` entry points at the PSE components
    under the PSE step ids. Each component only renders for its own id.
  - The two apps' step ids for this shape therefore differ, by design. The comments in both
    `form-shapes-config` files say so.
- **Shared components get optional props, and the defaults keep today's output:**
  - `PseBreakoutRanking` `withDescriptions`;
  - `PhotoConsentCheckbox` `modalTextId`.
  - `PhotoExamples` is unchanged: the copy filters the list, and two tiles keep the grid's tile size.
- **The success page learns the shape from the visibility endpoint.**
  - `GET /api/categories/visibility/{slug}` now also returns `form_shape`. This is additive, and the
    route is not in the mobile contract (checked `../../mobile/`).
  - `success-sections` renders `PifStrategyBrief` when the shape is `pif-strategy`.
  - Until the backend ships, `form_shape` is absent and the brief simply doesn't show, so the
    deploy order is free.
- **No new session keys.** The sessions are the same three; `Guest::BREAKOUT_SESSIONS`,
  `breakoutSessionFields` and the exports are unchanged, since the backend never checks the shape.
- **Arabic is drafted by us and pending the client's review.** This covers:
  - the three session descriptions;
  - the consent text;
  - the brief;
  - the forum's name, which we rendered as "منتدى الاستراتيجية ضمن سلسلة شركاء صندوق الاستثمارات العامة";
  - the admin label.

## Verification still owed (browser)

- [ ] Admin: create a category with shape **Strategy Forum registration (with photo)**. Mandatory
      fields offer the PSE with-photo list, breakout sessions included.
- [ ] Public, EN + AR:
  - Page 1 shows **two** photo tiles, the `StrategyForum@` contact, and a consent read-more naming
    the Strategy Forum.
  - Page 2 shows a description under each session, and ranking still works.
  - Submit, and the success page shows the brief.
- [ ] Admin create, edit and see-more on a pif-strategy guest, and both export buttons.
- [ ] Regression: a PSE category still shows five tiles, no descriptions and no brief.
- [ ] Arabic copy reviewed by the client.

## Before this runs anywhere

Deploy all three apps. No migration is needed. Then create the Strategy Forum category with shape
`pif-strategy` and tick its mandatory fields.

## Definition of Done

- [ ] Code merged to `dev` in the relevant sub-app(s)
- [x] EN + AR translations in the same commit (if any user-facing strings)
- [ ] Quality gate green (backend `pint --test` + `php artisan test`; Next apps `yarn type-check` + `yarn build`). Blocked on the unrelated deal-test failures above.
- [ ] Docs updated (this TASK.md set to `done`; index row updated; any drift fixed)
- [x] Mobile contract checked if `routes/api.php` touched (`../../mobile/`). The route is unchanged; one additive field on a non-mobile endpoint.
