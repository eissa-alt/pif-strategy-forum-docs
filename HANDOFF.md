# Handoff — current state

> Rolling pointer, overwritten each session. For the durable record see the per-task `TASK.md`,
> `decisions/LEDGER.md`, and `upgrades/UPGRADE_SUMMARY.md`. Full plan: `upgrades/CYAN_FEATURE_PARITY_MASTER_PLAN.md`.

**2026-10-08 (latest) — Task 044: the `pif-strategy` form shape for the PIF Partners' Series
Strategy Forum. Implemented in backend + admin + frontend and committed on `dev` (backend
`d6c878f`, admin `1741030`, frontend `bc99a65`).**
- **[044](tasks/044-pif-strategy-shape/TASK.md).** It is PSE with-photo plus:
  - two photo examples;
  - a description under each breakout session;
  - the forum brief on the success page;
  - the forum's contact email and consent text.
  
  The PSE shapes are unchanged.
- **Gates:**
  - Both Next apps pass `type-check`, `build`, and eslint + prettier on the changed files; admin
    also passes `check:rbac`.
  - Backend `pint --test` and `phpstan` are clean.
  - ⚠️ The backend suite is **37 failed / 718 passed**. All of the failures are in
    `DealSubmissionTest` / `DealSlotBookingTest`: a live reCAPTCHA call under `testing`, unrelated
    to 044 (see its Log).
- **Owed:** a browser walk (EN + AR), and the client's review of the Arabic copy.
- **Not reflected below:** the P1.1–P1.6 rebrand to the Internal Audit Community Forum (2026-09-23/24)
  and [043](tasks/043-pse-breakout-sessions/TASK.md), the PSE breakout sessions (2026-09-28, still
  `in-progress`: browser checks owed).

**2026-09-21 — Tasks 040 to 042: the client's registration feedback, deals as a per-guest
link, and deal signing slots on GFAI's model. Everything COMMITTED, PUSHED and MERGED to `main`
in all three code repos; `dev` = `main` = origin. Backend pint + phpstan + 741 tests; both Next
apps type-check + lint + build; admin check:rbac.**
- **[040](tasks/040-registration-feedback-2026-09/TASK.md), registration feedback (2026-09-20).**
  - Job title note states the 30-character cap.
  - A **required** photo-consent T&C.
  - The PIF communications question moved under the photo.
  - Inquiries email `psf@pif.gov.sa`.
  - Arabic "الاتفاقيات".
  - The **photo check now runs in the browser** (Tourise's face-api) and **only advises**. The
    old server heuristic is **off** behind `PHOTO_REVIEW_DRIVER`, with an Azure driver dormant
    for lack of a key ([D50](decisions/LEDGER.md)).
  - Also from Tourise: the Mobile Number field and the photo example plates.
- **[041](tasks/041-deals-per-guest-link/TASK.md), there is no deals category ([D48](decisions/LEDGER.md)).**
  The private category's guests get a single-use link from a 🤝 Deal action in the guest
  listing; the deal is stored against the guest. The category route and shape are removed, and
  the old category is **switched off by migration, not deleted**.
- **[042](tasks/042-deal-signing-slots/TASK.md), signing slots ([D49](decisions/LEDGER.md)).**
  - Each agreement books a slot the team opened. A pending deal holds it; reject or return
    releases it.
  - Admin: Signing schedule / Time slots / Venues / Settings, a Signing slot column, and
    ✓ / ✗ from the row and the grid through a confirm window. **Delete is removed.**
  - Emails come from the templates in Deals > Settings.
- **⚠️ Before this runs in production, in order:**
  1. `php artisan migrate`: five new migrations since 2026-09-20.
  2. `yarn install` on the frontend (`@vladmandic/face-api`), then build both Next apps.
  3. Private category: switch on **Deals Signing Ceremony** and pick a template with the
     **Deal Link** variable.
  4. **Deals > Time slots:** open slots on the signing days. **With none, guests cannot submit
     a deal.**
  5. Deals > Settings: pick the templates.
- **⚠️ Browser coverage is partial.** The guest deal page was walked in headless Chrome (EN + AR)
  and two bugs were fixed from it. **Not yet opened in a browser:** the admin deal screens, and
  the registration photo upload with its face check.
- **Owner-facing leftovers:**
  - "See more" still holds a review section; it is the only way to mark a deal *returned*.
  - The deal delete endpoint still exists behind its permission.
  - Moving a booked slot does not email the guest.
  - The owner's local DB has a demo schedule (5, 6 and 7 Oct).

**2026-09-01 — Task 034: the PSF 2027 registration sheet is implemented, except Deals.
Two new guest form shapes ship end to end (backend + admin + frontend + docs). ALL GATES GREEN —
backend pint + phpstan + **635 tests** (618 → 635, 17 new); both Next apps `yarn type-check` +
`yarn build`; admin `yarn check:rbac`. NOTHING COMMITTED, NOTHING PUSHED.**
- **What shipped.** `psf-private-2027` (1 step) needed **no migration** — every field on the
  "PRIVATE PAGE 2027" sheet already existed. `psf-public-2027` (2 steps) needed **6 new columns**:
  `position_level`, `ecosystems` (json, capped at 2), `attendance_purpose` + `_other`,
  `pif_communications_consent`, `company_profile`. Both shapes have public forms, admin
  create/edit, see-more panels, export columns, seeder catalogue entries and EN+AR.
- **⚠️ The Deals tab is NOT built.** It produces no `guests` row — two signing parties, 3 attendees
  each, monetary values, a review lifecycle and its own dashboard — so it cannot be a form shape.
  Specified in full at [`tasks/035-psf-2027-deals/TASK.md`](tasks/035-psf-2027-deals/TASK.md) with
  8 decisions to lock at kickoff. Scoped out with the owner, not overlooked.
- **⚠️ Uploads now accept PDF, on one narrow path only.** Every upload in the repo was jpg/png
  (`UploadService::ALLOWED_EXTENSIONS`, `Base64Image`), so "upload your company profile" could not
  be submitted at all. Added `Base64Document` + `UploadService::storeBase64Document()`, which is
  **private-disk-only** and always sniffs the decoded bytes. **`storeBase64()` is behaviourally
  unchanged** — the public-disk callers pass `imageOnly: false` and lean entirely on the extension
  allowlist, and widening it for them is the exact bug its docblock warns about.
- **⚠️ The guests listing has TWO export buttons hitting DIFFERENT classes** — this was found in a
  verification pass, after the first implementation had already "finished". "Export all" →
  `GuestsExport`; **"Export guests" → `GuestsExportGuestView`**, which was missing the six new
  columns entirely. Now fixed in all three guest exports (`GuestsExportView` too — live routes, no
  admin caller today), and the export tests are parameterised over the three classes asserting both
  width and heading presence, so a fourth cannot be missed the same way.
- **⚠️ Export columns were APPENDED, never inserted.** `GuestsExport` formats by hardcoded column
  letter (`COL_TITLE` 'D', `COL_EMAIL` 'I', `COL_PHONE` 'J', `DATE_COLS` M/Q/X) and
  `ExportColumnAlignmentTest` pins them; the six new columns start at Z. That test was re-run alone
  and is green, plus a new case pins `headings()` and `map()` to the same width.
- **⚠️ Nothing has been verified in a browser**, and **the migration has only run on sqlite**
  (`:memory:`, via RefreshDatabase) — there is no MySQL on this machine, so `->after()` placement
  and the real `json` type are unconfirmed. Click-list + both gaps in the task's
  "Verification still owed".
- **Two things the sheet asks for that are deliberately NOT built:** photo white-background
  detection (p1 r9 "Simple detection solution to reject photos") — stated in the upload note and
  left to a human; and `ecosystems` in the `where_null`/`where_not_null` filter, because that
  filter does `orWhere($f, '')` and MySQL rejects that on a `json` column.
- **Before this runs anywhere:** `php artisan migrate` **and**
  `php artisan db:seed --class=FormShapeFieldsSeeder` — without the seeder the two new shapes have
  no mandatory-field rows and the admin picker falls back to `default`'s list.
- **Also this session:** `.env.local` created for admin + frontend and `.env` for the backend (all
  gitignored, none previously present), modelled on the sibling projects and this repo's own
  `.env.example_prod`. Admin :3000, frontend :3001, API :8000.
- **Next:** commit the four repos, run the browser click-list, then Task 035 (Deals).

**2026-08-08 — Task 033: section 4 CLOSED and section 5 STARTED. 20 commits total across
backend / admin / frontend / docs. ALL COMMITTED, NOTHING PUSHED. Backend tests 489 → 494. A live
production defect was found and fixed: reCAPTCHA was rejecting every login and registration under
`config:cache`.**
- **⚠️ READ FIRST — `GOOGLE_RECAPTCHA_SECRET` was returning `null` in production.** It was read with
  `env()` outside a config file (`ReCaptcha.php:31`), and Laravel stops loading `.env` once the
  config is cached. Proven on a real config cache: `env()` → `NULL`, `config()` → present. Google
  received an empty secret, answered `invalid-input-secret`, and **every login and every
  registration was rejected**, at all 13 call sites. Fixed in backend `bf31b85`.
  **larastan was already catching this** (`noEnvCallsOutsideOfConfig`) — the finding sat suppressed
  in `phpstan-baseline.neon`. **12 entries for the same rule remain across 13 other files**;
  `DynamicSmtpService` (4 calls) is the next to look at — ledger D971 says it has already caused an
  incident there.
- **Section 5 items shipped (8):** admin `4151422` every dropdown freezing the page (14 panels /
  13 files) · `ec9ab15` guests listing dropping 9 filters on pagination · `6cc17d5` email-log flags
  reading "No" forever · `9f10242` Reconfirmation filter wired to the API · `67710b3` three blank
  boolean columns · `864db95` invitations form showing the real 422. Frontend `2d66436` the same
  dropdown freeze on the public registration form (**no fork ever fixed this side**) · `1d542c4` the
  complete-data link creating a duplicate guest instead of updating one.
- **New doc — [`upgrades/FORK_PORT_BACK_LEFTOVERS.md`](upgrades/FORK_PORT_BACK_LEFTOVERS.md).**
  Everything hit and deliberately **not** acted on, with the reason: the 12 remaining `env()` calls,
  8 dead filter keys with no rendered control, the `ExportEBadgesFiltered` park-or-delete call, the
  `check_in_time`/`checked_in_at` duplication, and the click-list of what still needs a browser.
- **⚠️ The findings doc keeps being wrong in its summaries** (its file/line refs have been right
  every time). Corrections recorded in place: the email-log bug is **5 files not 4**, and the fifth
  (`automation-details`) **must not** be changed — it renders string columns, so `=== 'yes'` is
  correct there; only **1 of 9** "dead filter keys" is a real bug; the dropdown `anchor` +
  `modal={false}` fixes do **not** have to land together, and `anchor` was skipped on purpose;
  `print-logs.tsx` needed no fix. **Keep opening the file before editing.**
- **Nothing has been verified in a browser** this session — all reasoned from source with every gate
  green. Click-list in the leftovers doc §4.
- **Next:** section 5 has ~30 items left. **Pushing all 20 commits is still outstanding.**

**2026-08-08 — Task 033: section 4 of the port-back backlog is CLOSED. Nine items shipped,
one closed as won't-do, 11 commits across backend / admin / frontend / docs. ALL COMMITTED, NOTHING
PUSHED. Backend tests 489 → 494. The admin/frontend build gate was repaired and ran green for the
first time.**
- **Commits (none pushed).** Backend `dev`: `afd4eee` (test backdoors) · `117da56` (invitation link
  + 5 tests) · `7bf1d9c` (phantom `dinner_invite`) · `880a7c0` (`checked_in_at`) · `e3591cc` (the
  `dd()`) · `9631e17` (reCAPTCHA guard) · `a19e8de` (rate limit 200→500). Admin `dev`: `9fb01fa`
  (upload paths) · `2db85bd` (plain `build`). Frontend `dev`: `38f42d8` (plain `build`). Docs `main`:
  `a36d172` (sidebar won't-do). Full per-item detail in
  [`upgrades/FORK_PORT_BACK_FINDINGS.md`](upgrades/FORK_PORT_BACK_FINDINGS.md) §4 and
  [`tasks/033-fork-port-back/TASK.md`](tasks/033-fork-port-back/TASK.md).
- **⚠️ THE GATE CHANGED — `yarn production` → `yarn build`.** The Next apps had *only*
  `env-cmd -f .env.<name>` scripts, and every one of those files is gitignored, so the build half of
  the gate **could never run in CI and needed a throwaway file locally** — that is the real reason
  this handoff has said "`yarn production` not run" for batch after batch. A plain
  `"build": "next build"` now exists in both apps. **The gate is `yarn type-check` + `yarn build`
  (+ `yarn check:rbac` on admin)**, all runnable with no `.env` present. `CLAUDE.md` rule 5 and
  `process/SETUP_AND_UPDATE.md` updated. `yarn production` remains, local-only.
- **⚠️ Read before touching production: `GOOGLE_RECAPTCHA_SECRET` is read with `env()` outside a
  config file** (`ReCaptcha.php:31`). Laravel does not load `.env` when config is cached, so under
  `php artisan config:cache` the secret is **`null`** → Google returns `invalid-input-secret` →
  **every login and every registration is rejected**, at all 13 call sites. There is no `recaptcha`
  key in `config/` at all. Found this session, not by the audit. **Confirm whether the deploy runs
  `config:cache` before anything else.**
- **⚠️ Two findings-doc items had wrong descriptions** — file/line refs were right, the summaries
  drifted. The upload item named `/admin/guests/upload`, which as an *axios* path resolves to
  `api/admin/admin/…` (the P20 trap); the correct path is `guests/upload`. The sidebar item said
  three links; it is **nine** plus a section header, hidden deliberately in `1e449dc`, and is now
  recorded as **not ported**. **Open the file before editing — it caught both.**
- **What section 4 actually bought:** multi-use invitations work again (a 50-use link died on the
  first registration, and never opened at all if that email had registered anywhere on the site);
  `GET /admin/guest-data-offline` works for the first time ever; gate scans now register in the
  Seating Manager; two remotely-triggerable 500s and a committed `dd()` are gone; two admin upload
  widgets that 404'd on **every** use are fixed; both Next apps can build in CI at all.
- **Deferred by owner decision (all recorded, none lost):** pep's single-check-in 422 rewrite (needs
  out-of-repo iPad-scanner sign-off); the reCAPTCHA `ConnectionException` 500 (access is already
  fail-closed — error-surface only); the mobile-CMS sidebar block; whether `ExportEBadgesFiltered`
  gets a PARKED docblock or deletion; and the `check_in_time` ↔ `checked_in_at` duplication.
- **Next:** sections 5–9, ~50 items. The five large ones need decisions rather than ports — worth a
  planning pass, not a straight roll-in. **Pushing all 11 commits is still outstanding.**

**2026-08-06 — Task 033 opened: a three-pass audit of the three clone projects produced a
port-back backlog. Analysis is DONE and committed; NOTHING has been ported. Also: the flaky
`SessionsTest` is fixed and the two P031 follow-ups are closed — all three pushed.**
- **The deliverable is [`upgrades/FORK_PORT_BACK_FINDINGS.md`](upgrades/FORK_PORT_BACK_FINDINGS.md)**
  (docs `7f049d3`, working process `212d58a`). ~60 verified items, 87 checkboxes: **17 one-liners**,
  20 small, 8 medium, 5 large. Grouped by project, quick wins first. The task record is
  [`tasks/033-fork-port-back/TASK.md`](tasks/033-fork-port-back/TASK.md); that TASK.md carries the
  locked decisions and open questions, the findings doc carries the checkbox list.
- **⚠️ Binding working process for this backlog: ONE item at a time, and no commit until the owner
  has reviewed the actual diff** (not a summary). Pushing is a separate approval. Written at the top
  of the findings doc. Reason: much of the list touches shared code — one `ListingTable` edit reaches
  46 listings, one `ui-select` edit reaches every admin dropdown.
- **Three things worth knowing before anything else:** (1) a multi-use invitation link **dies after
  the first registration**, and because the check isn't scoped to the invitation, if that email ever
  registered anywhere else on the site the link **never opens at all** —
  `InvitationsController.php:467` runs before the remaining-uses gate at `:475`; (2) the tokenized
  "complete your data" link **creates a duplicate guest** instead of updating the record, in **2 of
  our 3 form shapes** — a link we email to real registrants; (3)
  `GET /admin/guest-data-offline` **has never worked** — it filters on `dinner_invite`, a column no
  migration creates, and the code's own comment at `GuestsController.php:2704` already says so.
- **The audit also found defects in THIS baseline that no fork fixes**, so no fork-diff could have
  surfaced them (findings doc §9): **two forced-500 test hooks on public unauthenticated routes**
  (`GuestsController.php:3732`, `InvitationsController.php:441`), a **`dd()` in committed controller
  code** (`:2638`, breaks our own hard rule 8), **public unthrottled `/pdf/{id}` routes**
  (`routes/web.php:24-25`) that run Imagick + write a file per request, five unrouted OTP endpoints
  that skip reCAPTCHA, and **hardcoded object-storage buckets belonging to other ALT clients** on
  live PDF render paths (devego, ims, tourise, hci-2026, glmc, faf).
- **⚠️ Three earlier conclusions were WRONG and are corrected in the doc** — if you remember them the
  old way, re-read: **P029.1 only half-fixed the title 500** (`gender_specific_ar` still assigns a
  raw array to an uncast `text` column in both `store` and `update`); **the email-log boolean fix
  landed on the backend but not the admin**, so every Sent/Delivered/Opened cell renders "No" today;
  and **the guests listing drops 9 filters on pagination**, not none — plus the "Reconfirmation"
  filter is dead end-to-end.
- **Two items are deliberately NOT ported, with reasons in the TASK.md:** gfeai's SMS non-production
  guard would **reverse our own P23.2 decision** (`28eca40` removed those guards on purpose, *after*
  gfeai forked), and gfeai's CSP widening would undo the env-derived origin design.
- **Open, needs a human:** does the admin CSP actually block reCAPTCHA on login? Two agents reached
  opposite conclusions from source — **settle it in a browser**, not by reading more code. And check
  whether the backend's `GOOGLE_RECAPTCHA_SECRET` is Google's *test* secret before enforcing
  reCAPTCHA in production (we don't read `.env`).
- **Code shipped this session (all pushed to `origin/dev`):** backend `4adaaec` (**P033.1** — fixes
  the `SessionsTest::test_search_finds_matching_sessions` flake that made the last gate 488/489; the
  decoy's `name_ar`/`description_en`/`description_ar` are now pinned instead of faker prose, so a
  stray "ai" substring can't match — passed 5/5 before commit). Admin `ecf3ffc` (**P031.2** — the
  hardcoded `Run` / `Send To All` / `Split by 500` literals now go through `useTranslate`, EN + AR)
  and `a6ab9dc` (**P031.3** — deletes the commented-out "More" dropdown). Those close the two
  follow-ups the 2026-07-28 entry left open. All working trees clean, all repos in sync.

**2026-08-06 — release hygiene, no code change: `dev` → `main` merged in all three apps, the
merged `feat/seating` branches pruned, and the P025–P032 quality gate finally re-run.**
- **`dev` → `main` is done.** Backend PR #4 (`fcc2541`), admin PR #4 (`4405a04`), frontend PR #2
  (`75f7e3a`), all merged 2026-08-06 ~12:17. In each repo `origin/main`'s **tree hash now equals
  `origin/dev`'s** and `origin/dev` is 0 commits ahead — the whole backlog (153 backend / 113 admin /
  46 frontend commits, through P032) is on `main`. Seating was already equal; docs is single-branch.
- **Branches pruned.** `feat/seating` deleted local + remote in **backend** (`2334445`) and **admin**
  (`982747e`) — both fully merged into `dev` *and* `main` (PR #3, backend `076cb8d` / admin `0f34b2d`),
  0 unique commits. The P021 work is untouched in history. Every repo now carries just `dev` + `main`
  (docs: `main`), all in sync, all working trees clean.
- **⚠️ Quality gate re-run 2026-08-06 — green, with two caveats worth knowing:**
  - **Backend:** `pint --test` **passed**; `phpstan` **0 errors** (361 files); `php artisan test`
    **488 passed / 1 failed**. The one failure is **pre-existing flakiness, not a P025–P032 regression** —
    see the next bullet.
  - **Admin + frontend:** `type-check` **passed in both**. **`yarn production` not run** — no local
    `.env.production`, which is the **expected, documented** state (D22 convention: DevOps creates it on
    deploy; see `decisions/LEDGER.md` "its absence is why `yarn production` can't run locally — expected,
    not a fault"). It **can** be run via the throwaway-copy workaround in
    [`process/SETUP_AND_UPDATE.md`](process/SETUP_AND_UPDATE.md) — `cp .env.local .env.production &&
    yarn production; rm -f .env.production` — which was **not** done here. So the build half of the gate
    is unverified for this batch, consistent with every prior batch.
- **⚠️ Flaky test — `SessionsTest::test_search_finds_matching_sessions` (~10% fail rate).** Re-run 10×
  on unchanged code: **9 pass, 1 fail**. Cause: the test creates a decoy session `'Cybersecurity Panel'`
  and asserts `keyword=AI` returns exactly **1** row, but `MobileSessionController::search()` also
  matches **`name_ar` / `description_en` / `description_ar`**, which `SessionFactory` fills with faker
  prose — any generated word containing the substring `ai` (*said, again, maintain, explain, fail*)
  makes the decoy match too. **Not yet fixed; no task folder opened.** Fix is to pin the decoy's other
  searchable columns in the factory call rather than to loosen the assertion.

**2026-07-28 (documented 2026-08-01) — an eight-item batch (P025–P032) across backend + admin:
two category features, three bug fixes, and the automation manual-run option. Tasks 025–032, ledger D46 +
D47 + addenda to D39/D42. All working trees clean; all commits pushed and, as of 2026-08-06, merged to
`main` (see the entry above).**
- **Why this entry is late:** the code landed 2026-07-28 but nothing was written up — no handoff entry,
  no ledger entries, no task folders (the board stopped at 024 while commits ran to P032). A drift sweep
  on 2026-08-01 caught it; this batch is the write-up. **No code was changed while documenting.**
- **Pushed + merged (was flagged unpushed here until 2026-08-06):** backend `6e7fd94` (P031.1) · admin
  `3fc30c9` (P031.1) + `77cea97` (P032.1) are all on `origin/dev` **and** `origin/main`. The rest of the
  batch was already on `origin/dev`. Frontend, seating and docs are in sync.
- **⚠️ Biggest deploy trap — D47 defaults OFF.** Task 028's migration `2026_07_28_000001` adds the five
  guest row-action switches **defaulting to false, with no backfill**, so **the moment it runs on prod,
  every guest row action vanishes from every category — resend email/SMS/WhatsApp, print badge, and
  *mark collected* — until an admin turns each one on.** Plan the switch-on as part of the deploy, not
  after. (`with_issued_visa` in `000002` *is* backfilled, so the issued-visa template keeps working.)
- **Task 027 — category cloning now actually copies everything (D46).** The old `clone` ran attributes
  through `fill()`, so **any column missing from `$fillable` was silently dropped** — and would keep
  being dropped for columns added later. Now `replicate()` inside one transaction, plus badge
  assignments, admin data scope, meeting-room links, and share-poster files **duplicated** to fresh names
  so a clone never shares an image with its source. Note the access-surface consequence: copying the
  admin scope means **the clone inherits its source's visibility** rather than starting private.
  Backend `0be944e`. **No test covers `clone`.**
- **Task 026 — guests filter by registration date.** `created_from`/`created_to` went into the **shared
  `applyFilters()`**, so the listing *and* every guests export honour the range for free; `whereDate`
  keeps both bounds inclusive and date-only, either alone is valid. New shared admin
  `date-range-input.tsx` emits `YYYY-MM-DD` straight into the query params. Backend `6abee02`, admin
  `6978795`.
- **Task 031 — automations gain a `manual` schedule (D39 addendum).** Saved as a draft
  (`schedule_type='manual'`, `send_status='draft'`, not dispatched on create) and run later through the
  **existing** `POST /automations/send/{id}`. **No migration needed** — `schedule_type` is `string(20)`,
  not an enum. Also: draft-gated Run action + `ConfirmModal`, direct Run/Split buttons on the details
  page, and run-neutral wording ("Send immediately"→"Run immediately", status "Sent"→"Completed";
  admin `af2a24a`, confusingly also stamped `P027.1`). **D39 parked item #2 is still open** — the
  *details* page's Run is still not gated on `send_status`, so it can fire a scheduled automation early.
- **Task 025 — the Seating Manager link is gone from the guests listing (D42 addendum).** Button only;
  the RBAC feature, the seating sub-app, the endpoints, the component and its labels all stay, so
  re-adding is one line. Read D42 as "seating is integrated, but not advertised from the admin."
- **Three bug fixes:** **029** title creation failed when its optional switches were untouched
  (`show_in_badge`/`show_in_user_form` are NOT NULL; an unchecked switch arrived as `null`) — coerced
  with `$request->boolean()` in `store` *and* `update`. **030** automation creation was blocked unless a
  guest status was picked — `guest_status_id` lacked `nullable` *and* its `required_if` matched the
  string `'yes'` while the form sends a boolean, so it never fired. **032** `{{ reconfirmation_url }}`
  added to the email editor's variable palette (the resolver already substituted it since Task 020).
- **Gates — re-run 2026-08-06, green.** Every commit passed its **pre-commit hook** (Pint on staged PHP;
  eslint/prettier on staged TS) at commit time. The full gate has now been run across the batch:
  backend `pint --test` + `phpstan` (0 errors) clean and **488/489 tests pass** (the 1 failure is the
  pre-existing `SessionsTest` flake, not this batch); admin + frontend `type-check` clean.
  **`yarn production` remains unverified** — needs a `.env.production`, absent by design (D22); runnable
  via the `process/SETUP_AND_UPDATE.md` workaround if wanted. See the 2026-08-06 entry at the top.
- **Dev DB is migrated** (`migrate:status`: `2026_07_28_000001` + `000002` both **Ran**). **Prod is not.**
- **Small follow-ups left open** (all in `tasks/031-automation-manual-run/TASK.md`): the details-page Run
  button's label is a hardcoded `'Run'` literal, not a translation key (pre-existing — it was
  `'Send To All'` before — so it won't render in Arabic), and `automation-details.tsx` carries the old
  "More" dropdown commented out rather than deleted.

**2026-07-27 — Admin phone field (Task 024, D45) + guest-access single-record scoping
(Task 023, D44). All committed on `dev` and PUSHED across backend + admin + docs. Dev DB migrated; prod
`migrate` + manual QA pending.**
- **Admin phone (D45):** admins now have a `phone` — same field name + `PhoneInputV2` widget as guests
  (owner: "use what we have for guest to be consistent", so `phone`, **not** `phone_number`). Additive
  migration `2026_07_27_000002` (nullable at the DB; **required** at the request/form layer — so editing a
  pre-existing admin now forces entering a phone). Exposed in `AdminsResources` + `AuthController::profile()`;
  form field + listing column + typed `AdminType`. **Same admin commit, owner request:** create form now
  defaults **status ON** and the **Data scope** section **expanded** (edit mode unaffected). **Shared fix:**
  `PhoneInputV2` error text was `text-error` — undefined in this app's Tailwind v4 theme, so validation
  messages rendered gray; now `text-red-500` (also fixes the guest admin forms, which share the component).
  **Deferred (owner):** a phone search/filter on the listing + an admins export. Commits: backend `f1c3dc3`
  (P024.1), admin `f743fae` (P024.2).
- **Guest-access scope (D44):** `GET /admin/guests/{id}` + `GET /admin/history-logs/{id}` are now scoped to
  the admin's categories/statuses via new `App\Support\GuestAccessScope::denies()` (a twin of the list-scope
  filter). A scoped admin opening an out-of-scope guest by UUID now gets **403** — closes a hole Task 022
  widened. Backend `ae1c210` (P023.1) — pushed earlier this session, **now documented** (folder 023 + D44).
  ⚠️ **Mirror rule:** if the list filter or `GatesController::deniesGateScope` changes, change
  `GuestAccessScope` too.
- **Gates (both):** pint + phpstan clean, **489 tests** (5 new via P023.1); admin `type-check` + eslint green.
  Not mobile-facing — both live under `/admin/*`; `routes/api.php` untouched.

**2026-07-27 — Task 022: guest history now shows WHAT an edit changed. Committed + PUSHED on `dev`
(backend + admin). Ledger D43.**
- **What:** `history_logs` gained `previous_payload` / `payload` (additive migration
  `2026_07_27_000001`), filled in `updateGuest`; the admin renders a `Field | From | To` table with the
  old value struck through, plus a "No changes in edit" note. Ported in spirit from `115-cyan-basecode`,
  which had it first — but **re-authored**, not copied.
- **⚠️ A DELTA, NEVER A SNAPSHOT — do not "finish" this by adding full snapshots.** Cyan snapshots the
  whole record because its guests table is one `form_data` JSON blob with no PII columns and public-disk
  uploads. 121 has ~110 columns incl. `religion`, `birth_date`, `document_number` and five private-disk
  file fields (D14), and the **Seating SPA writes every seat drag through the same `PUT
  /admin/guests/{id}`** — a snapshot would persist a passport number twice per seat move, with no
  erasure path. File fields store a `[file]` sentinel, never the path.
- **`[]` ≠ `null`:** `[]` = "changed nothing" (shows the note); `null` = unknown — a pre-task row or an
  action that never carried an edit (keeps the one-line format). **No backfill is possible**, so the
  list stays permanently mixed.
- **Moved into the see-more modal** (`5ae2afb`). The `/guests/{slug}/extra/{id}` page it lived on is
  reachable by **direct URL only** — the listing's see-more button opens the modal — so the feature was
  effectively invisible. Also the better RBAC host: the API needs `guests_listing,see_more` but that
  page resolves to feature-level `guests_listing`.
- **Deleted a misleading UI element:** the Email/First/Last table under the log was read from
  react-hook-form — the guest's values *right now*, unrelated to any row.
- **Not a complete audit trail (be accurate about this):** `importGuestsExcel`, the `upload*` methods,
  `attend()`/`guestsSyncOffline()`, `reGenerateSMP()` and `MobileAuthController::updateProfile` still
  write **no history row at all**.
- **Commits (PUSHED):** backend `0df228e` (P022.1); admin `a959703` (P022.2) + `5ae2afb` (P022.3).
  **Gates:** pint + phpstan clean, **484 tests** (4 new — `history_logs` had zero coverage), admin
  `type-check` + `eslint` + `check:rbac` green, EN/AR 1760/1760. Dev DB migrated; **prod migrate
  pending.** Verified in the browser: diff table, no-changes note, modal tab.
- **Known gaps left open (in `tasks/022-guest-history-payload/TASK.md`):** `HistoryLogsController::index`
  has no `applyAdminGuestAccessFilter` scoping (it would now leak a field diff to an out-of-scope admin)
  and no pagination; an unchanged save still writes a "Guest updated" row.

**2026-07-27 — Prod queue/scheduler runbook added + a docs-accuracy sweep. Docs repo only; no
code touched. All four code repos remain clean and in sync with `origin`.**
- **New runbook:** `process/QUEUE_SETUP_PROD.md` (+ PDF twin for DevOps) — the supervisor program for
  `queue:work`, both scheduler options (cron `schedule:run` or a `schedule:work` program, `numprocs=1`),
  `queue:restart` after every deploy, and verification commands. Brought over from the **123** clone and
  **de-branded** (server path, both program names, both log files now read `alt-static-basecode`); PDF
  re-printed via headless Chrome to match. This is the written-up form of D39's prod dependency —
  **installing it on the prod box is still outstanding.**
- **Task 020 status corrected:** both the tasks-index row and `020-reconfirmation/TASK.md` claimed the
  work was uncommitted / "NOT pushed". It is **pushed** — backend `27b764f`+`045223a`, admin `e12d548`,
  frontend `854b44d`, all contained in `origin/dev`. Now `done (code)`; dev-DB migrate + manual QA +
  mobile notice remain the open items.
- **Also corrected:** `PHASE22_PARKED_TODO.md` §2 claimed "the ledger stops at D34" — it runs to **D42**.
  The two findings it names (SMTP mailer memoisation / `forgetMailers`, and the `GuestsResources`
  date-only fix) *are* still genuinely unrecorded, so the item stays open, scoped to just those two.
- **Cross-links added** so the runbook is findable: docs map + a "Deploying to production" quick-link,
  `SETUP_AND_UPDATE.md` (beside the local `queue:work`/`schedule:work` notes), and a D39 addendum.
- **New clone-checklist item:** the runbook's **PDF is binary**, so Bucket 1's `*.md` rename sweep misses
  it — a clone would hand DevOps a PDF naming the wrong project. Flagged with the fix.

**2026-07-26 — Task 021 seating MERGED, and the SPA re-based to the current upstream `85fecfb`.**
- **Merged + pushed:** backend + admin `feat/seating` → `dev` (`076cb8d` / `0f34b2d`); docs on `main`. Backend
  480 tests green; admin `type-check` + `check:rbac` green.
- **SPA re-based (important):** the first import was stale (`d145e13`). Rebuilt on the current 120 upstream
  **`85fecfb`** — the version the client actually tested (dark-mode theming, room presets, table-shape geometry,
  a SyncStatus indicator; 3 backend-less modals removed) — keeping the **full 120 git history** so future upstream
  updates can be pulled. Retarget re-applied (`guests-update/{id}`→`guests/{id}` + `mapAdminToUser` reads
  `permissions.seating`). Published to `eissa-alt/alt-static-basecode-seating` (`dev`+`main` `09d23a5`),
  `npm run build` clean; 123 `pif-pep-v2-seating` re-cloned to match (wired via `basecode-local`).
- **Backend + admin unchanged** — verified the `85fecfb` API contract == what was built. The D-6 "full port"
  (building the `seating-audit-log` GET/POST) turned out to be exactly what the newer SPA calls.
- **⚠️ Pending owner:** set `.env` (backend `CORS_ALLOWED_ORIGINS`, admin `NEXT_PUBLIC_SEATING_MANAGER_URL`,
  seating `VITE_PIF_API_BASE_URL` + `VITE_GOOGLE_RECAPTCHA_KEY`); `php artisan migrate` (dev/prod); live
  end-to-end QA. Seat/attendance write-ops also need `guests_listing,edit` (the write rides the guests endpoint).

**2026-07-25 — Task 021 Seating Plan Manager implemented end-to-end (Phases A/B/C) — ledger D42.
Code-complete + all gates green, on FEATURE BRANCHES, NOT pushed.**
- **What:** ported the standalone Seating Plan Manager (Vite/React SPA from v1/120) into the baseline as a
  **new 4th sub-app** `alt-static-basecode-seating` (API-wire, no UI merge) + backend endpoints + a new
  `seating` RBAC feature + an admin deep-link launch button. Plan + decisions: `tasks/021-seating/TASK.md`; D42.
- **Backend (`feat/seating` `2334445`, P021.1):** migrations `2026_07_25_000005..000008` (seating_layouts +
  versions + audit_logs + 6 guest attendance columns); SeatingLayouts/SeatingAuditLogs controllers;
  `/admin/seating-layout*` + `/admin/seating-audit-log` gated `admin.can:seating,<action>`; data-wrapped
  `/admin/me`; `updateGuest` accepts the 6 attendance keys (seat + attendance write-back persists) + a
  `Guest::booted()` bridge to legacy `check_in`. Gates: pint + phpstan clean, **480 tests pass** (6 new).
- **Seating SPA (`dev` `e1b4191`, P021.2, on import `e66c2df`):** retargeted `pifApi.js`
  (`guests-update/{id}` → `guests/{id}`) + `mapAdminToUser` now reads the `seating` RBAC actions
  (view/check_in/manage) instead of the removed `admins.type`. `npm run build` clean.
- **Admin (`feat/seating` `982747e`, P021.3):** `seating`-gated "Seating Manager" deep-link on the guests
  listing (own-login, no token — D-4), EN+AR, `NEXT_PUBLIC_SEATING_MANAGER_URL` in `.env.example_prod`.
  `type-check` + `check:rbac` green.
- **123 inheritance:** `pif-pep-v2-seating` cloned + wired (`basecode-local`) — one `--ff-only` pulls this in.
- **⚠️ Pending owner (nothing pushed):** review + merge the 3 branches → `dev` + push; create the GitHub seating
  remotes (`eissa-alt/alt-static-basecode-seating` + `pif-pep-v2-seating`); set `.env` (`CORS_ALLOWED_ORIGINS`,
  `NEXT_PUBLIC_SEATING_MANAGER_URL`, seating `VITE_PIF_API_BASE_URL` + `VITE_GOOGLE_RECAPTCHA_KEY`); run
  `php artisan migrate` on dev/prod; live end-to-end QA against a running stack. Seat/attendance write-ops also
  need `guests_listing,edit` (rides the guests endpoint). `yarn production` not run (needs `.env.production`).

**2026-07-25 — Automation scheduling (D39) + details page restyle (D40) + a WhatsApp RBAC fix
(D41). All **pushed** (`dev` + docs `main`); the older unpushed D38 picker went out in the same push.
Dev DB migration applied. Still pending: prod scheduler + prod migrate + manual smoke test. (This session
also landed D37 single-channel + D38 filter-and-select picker.)**
- **WhatsApp RBAC fix (D41):** `check:rbac` was red (4 errors) — WhatsApp (D36) shipped routes + sidebar
  features but no `inferFeatureId` rules (the D33 first-match-wins trap). Added 3 rules mirroring SMS; now
  green. **`yarn check:rbac` is now in the documented admin gate** (it's a standalone script, not covered by
  type-check / production / composer qa). Fix lives in the baseline so clones `--ff-only` catch up.
- **Scheduling (D39):** run-automation modal gains a Scheduling step — *Send immediately* (auto-dispatches
  on Create) or *Schedule for later* (clean masked date + time, **not** the native `datetime-local`). A
  shared `AutomationDispatchService` holds the per-guest fan-out (fires the existing send events — it does
  NOT replace the event→listener architecture) and is called by all three entry points: the manual Send
  button, immediate-on-create in `store()`, and the cron command `automations:dispatch-scheduled`. Listing
  gains a `send_status` badge + a Cancel action (only while `send_status='scheduled'`).
- **Details page (D40):** `/automation/details/[id]` rebuilt on the shared listing primitives
  (`useListingState` + `ListingTable`/`Filters`/`Footer` + `TopSection`) to match the admin theme; MORE
  (Split/Send) + Export + status filter preserved; **Clicked** column hidden; orphaned search deleted;
  shared `web:sent_at` label tidied (`"sent_at"` → `"Sent at"`, also affects logs/e-visa headers).
- **⚠️ Deploy / run (NEW dep):** prod must run the **Laravel scheduler** (`cron schedule:run` OR a supervisor
  `schedule:work` program) for scheduled automations to fire — separate from the existing `queue:work`
  supervisor (reused unchanged for sending). Immediate automations don't need it. Run `php artisan migrate`
  (`2026_07_25_000002_*`) on deploy **and on the dev DB before testing**. Runbook updated:
  `process/SETUP_AND_UPDATE.md`.
- **Parked (raised, holding):** (1) the listing shows two near-redundant status columns — legacy **Operation
  status** vs new **Send status** — collapsing to one is a pending UI call; (2) the details page's manual
  "Send To All" can dispatch a *scheduled* automation early (not gated on `send_status`); (3) automation
  `created_by`/`updated_by` (no update endpoint yet). `send_status='sent'` means "handed to the queue", not
  "delivered".
- **Commits (pushed):** backend `38729eb` (P24.25); admin `673f00d` (P24.25 scheduling UI) + `903013c`
  (P24.26 details) + `fe1ac09` (P24.27 WhatsApp RBAC fix); docs on `main` through `28a5e19` (+ D41/gate note
  in this update).
- **Gates:** backend `pint`/`phpstan`/`test` (469) green; admin `type-check` + `eslint` clean. `yarn
  production` not run (no local `.env.production`). **Manual testing pending** (needs the migration first).

**2026-07-24 — P24: WhatsApp channel added end-to-end (Meta Cloud API), built as a twin of the
email/SMS stack. Committed on `dev` (backend 12 + admin 5 + frontend 1) + the plan doc on `main`, NOT pushed.
Ledger D36.**
- **What:** a full WhatsApp delivery channel across all three apps — provider-config + templates CRUD, a
  `WhatsAppSender` transport, the invitation send pipeline (whatsapp as a third `channel`), guest/automation
  notifications (register / accept / reject), read-only logs, an inbound RSVP webhook, and WhatsApp OTP in the
  public join form. Every ALT-facing piece mirrors our own email/SMS code; `x-hci-campaign` was a **reference
  only** for the Meta-specific mechanics (Graph payload, webhook signature, template variable/button mapping).
  Plan: `upgrades/WHATSAPP_INTEGRATION_PLAN.md`.
- **Twin, not port:** `WhatsAppProviderConfig`↔`SmsProviderConfig`, `WhatsAppTemplate`↔`SmsTemplate`, queued
  event/listener sends, separate `invitation_whatsapp` / `guest_whatsapps` observability tables, category
  `with_whatsapp` master gate, and admin listing/form layouts all copied from the SMS stack. Transport is
  `WhatsAppSender` keyed on `provider_key` (`meta_cloud` only in v1).
- **⚠️ Schema is additive forward-only** — four new `2026_07_24_*` migrations ADD the whatsapp tables/columns;
  no existing migration was edited (`migrate:fresh` is banned now that the DB has real data).
- **Inbound RSVP webhook:** `GET /webhooks/whatsapp` verifies the provider `verify_token`; `POST` validates the
  Meta HMAC-256 signature (`app_secret`), updates delivery status on the log row, sets `reply_status`
  `confirmed`/`declined` from quick-reply buttons, and fires a QR follow-up on confirm.
- **WhatsApp OTP shares the SMS phone code** — same `phone_verifications` row + `phoneConfirmation`; only the send
  endpoint (`POST /guests/whatsapp-verification`, a `purpose='otp'` template + `otp_whatsapp_config_id`) differs.
  Registration accepts `with_sms_otp` OR `with_whatsapp_otp`; the join form offers WhatsApp reusing the phone
  step/confirm verbatim. **Channel note:** with both OTP flags on, the form has no picker and prefers WhatsApp.
- **Mobile:** additive only — new **public** routes (`GET` + `POST /webhooks/whatsapp`,
  `POST /guests/whatsapp-verification`) + an additive `with_whatsapp_otp` field on verify responses; nothing
  removed/renamed. Check `docs/mobile/BACKEND_INCOMING_CHANGES_FOR_MOBILE.pdf` before wiring mobile to it.
- **Commits (NOT pushed):** backend `d33affa`, `f185608`, `28ec5d8`, `d4daaf8`, `51b0622`, `2ee06bf`, `555a91e`,
  `114bc35`, `7a370b0`, `f7e8ea8`, `14eff38`, `c224dd3`; admin `8d4f2f3`, `030df8b`, `0f09641`, `e536590`,
  `f58872f`; frontend `657c173`; docs `1d9ee7f` (plan, on `main`). **Gates:** every commit green (backend
  `pint --test` + `phpstan` No-errors; admin/frontend `yarn type-check`), EN + AR in-commit.
- **Remaining:** manual QA against a real Meta WABA (template approval, live send, webhook signature + RSVP
  round-trip, OTP delivery) — nothing exercised against Meta yet; and the both-OTP-flags-on channel-picker decision.

**2026-07-24 — P23 email/SMS audit hardening: 33 confirmed fixes across all three apps. Committed on
`dev` (backend 12 + admin 1 + frontend 1), NOT pushed. Ledger D35.**
- **What:** a full audit of the email/SMS subsystem surfaced 45 candidates → **33 verified + fixed** in small
  reviewed batches (P23.1–P23.14). Highlights: accept/reject notifications un-suppressed (H1 — the D31 master
  gate was starved by a partial eager-load of `with_email`/`with_sms`); automation send made **idempotent**
  (409 on re-send) + side-effects on the email path + e-badge on split; invitation `extractBulk`
  channel-scoped (no more double-message); phone OTP hardened (CSPRNG + 30-min TTL + persist-after-send);
  email-log boolean filters fixed; `env`→`config` for `config:cache` safety; SMS test-send made real,
  mobile-OTP SMTP re-applied in the worker, `workflow_value` tagged on every guest-SMS.
- **⚠️ Behaviour changes now live on `dev`:** **SMS sends in every environment** (guards removed — real
  Unifonic texts/cost from dev/stage, incl. bulk runs); phone OTP **expires after 30 min**
  (`PHONE_OTP_TTL_MINUTES`); register SMS now also goes to **partner guests**; an automation setup can be
  **sent only once** (re-click → 409). **Email deliberately still sends everywhere** — H2's non-prod guard was
  rejected (breaks local testing); control delivery via `.env` `MAIL_MAILER`.
- **Deferred with a precondition (not dropped):** **L-otpgate** (gate the OTP *send* on the master flag) needs
  a 1-row data audit first — `with_email`/`with_sms` have no backfill migration and `migrate:fresh` is banned,
  so gating blind could 500 registration; query in ledger D35. **L-reminderemail** (SMS-only invitations can't
  be reminded) is a feature → fold into WhatsApp.
- **Commits:** backend `d0cd0b0`,`28eca40`,`5b934c1`,`0c8ee04`,`d1ae700`,`b1f07e8`,`d7bf834`,`42e6269`,
  `14085a6`,`eaa76be`,`7eb1e54`,`e8da311`; admin `f17ed0a`; frontend `c69f35c`. **Not pushed** — awaiting
  review. **Gates:** every commit green (backend pint+phpstan; admin/frontend type-check). `routes/api.php`
  untouched → no mobile delta.

**2026-07-22 — P22 client-name sweep (P22.1–P22.6). ALL PUSHED.**
- **Every past-client name is out of the three code repos** — PIF, HCI, EDGEx, TOURISE, DeveGO, plus
  dead FAF/DGCA/ICAO templates. `components/join/forms/pif/` is now `forms/default/`; `pif-one-step`
  was deleted rather than renamed (no category used it and the form-shape select never offered it).
  `docs/ai/AI_RULES.md` rule 4 was rewritten in the same breath: it protects the *form-shapes pattern*,
  not the `pif/` folder name, so a clone renames `default/` rather than resurrecting `pif/`.
- **Two dead-code finds carried real branding.** `emails/notify_guest/{en,ar}` (12 hardcoded
  @hci_ksa social links) had **zero** references anywhere — the live guest-email path is
  `emails.base.waypoint`, which renders socials from the `social_media_links` table.
- **The invitation-PDF feature is parked, not fixed** (`343e6e8`, `efabc6d`). Four call sites loaded
  `pdf.invitation`, a view that has **never existed in this repo's history**; `dinner_invite` is not a
  column either. Routes, the automation attachment branch and the admin toggle are gone; the DB column
  and API contract are untouched, so re-enabling needs no migration.
- **Badge QR codes now render locally** (`9d4d5f3`). They used to `<img src>` api.qrserver.com, so mPDF
  made an outbound request per badge — printing hung when that service was slow, and every guest's
  registration number went to a third party. New `App\Services\QrCodeGenerator`; badge printing
  **manually confirmed** by the owner. Note: **no automated test covers badge/PDF rendering.**
- **⚠️ The email QR still calls qrserver.com on purpose** — Gmail/Outlook strip `data:` images, so the
  PDF fix does not transfer. Parked with the decision written up.
- **Commits:** backend `57f6d19`, `8a937d4`, `343e6e8`, `9d4d5f3`; admin `2c1cb75`, `cdaf902`,
  `4b27162`, `efabc6d`; frontend `3822549`, `52f6ebf`, `aa78591`; docs `ed5d187`.
- **Parked follow-ups → [`tasks/PHASE22_PARKED_TODO.md`](tasks/PHASE22_PARKED_TODO.md)** — the email QR,
  the ledger debt below, 622 unused translation keys, branded PDF backgrounds needing real artwork, and
  the test-coverage gaps. **Read it before starting work in any of those areas.**

**2026-07-22 (later) — P20 correction pass (ledger D33). ALL PUSHED.**
- **`inferFeatureId` swept, not spot-checked.** The first-match-wins trap turned up **six** times, two
  of them in already-shipped code: `sms_logs` (`/logs/*-sms` gated on `email_logs`, shipped P18.2),
  `/emails/smtp-configs` and `/gate-scan` (both long-standing, the latter defeating the gate-agent role
  D23 exists for). A ~20-line script comparing every live sidebar link's declared `featureId` against
  `inferFeatureIdFromPath(href)` found the two nobody had spotted. **All 28 live links now agree —
  turn that script into a test.**
- **The logistics screen was dead on arrival** (`7193f6f`). It called `/admin/guests/...` while the
  admin axios base already ends in `/api/admin`, so every request 404'd and the screen rendered its
  error state on every load. It shipped that way in P19.4 and was only caught when re-reading the code
  to answer a question. **Rule: admin Axios paths never carry an `/admin/` prefix.**
- **Logistics 403 resolved** (`436d225`): the loader's `Promise.all` spans two features, so the guest
  leg is now tolerant — it hides the guest-declared fields when unreadable and drops them from the PUT
  so they cannot be nulled unseen. A logistics-only role stays viable.
- **`CategoriesExport` column shift** (`ecce5a6`): `map()` omitted `visibility`, printing 20 values
  under 21 headings.
- **`composer qa` is green end to end (467/0)** (`c9bbdf9`) and **`php artisan test` no longer wipes the
  dev database** — `phpunit.xml` now pins sqlite `:memory:`. The "465 pass / 3 pre-existing" baseline is
  retired; treat any failure as real.
- **Commits:** backend `ecce5a6`, `c9bbdf9`; admin `621abdb`, `7193f6f`, `1bd50e5`, `436d225`.
- **⚠️ Still unexercised:** none of Task 019 or P20 has been opened in a browser. The logistics screen
  is the priority — it has had zero real use. RBAC needs a real restricted login too; the sweep proves
  the map is self-consistent, not that the whole chain works.

**2026-07-22 — Logistics + e-visa re-added from the P5.trim (Task 019, ledger D32). Since PUSHED —
8 commits across backend / admin / frontend / docs.**
- **What:** hotels, rooms (nested), traveling-status, per-guest `guest_logistics` + 4 exports + the
  logistics screen with hotel assignment, and e-visa package generation / PDF / issued-visa handling /
  admin console. Recovered from this repo's own pre-trim code where it still matched (`08d542e^`,
  `e3a0677^`) and from hci-2026 `origin/main` for e-visa, which is a newer generation than what we
  deleted (`e_visa_files` with real audit timestamps, not `e_visa_exports`).
- **Modernized on the way in:** `status` → `is_active` booleans, six string date/time columns → `date` +
  `string(5)` `HH:mm`, `flight_costs` → decimal, `block`/`activate` → one `toggle-status`, RESTful
  groups + `admin.can` gating, `useFetch`, `masked-date-input`, `CustomSwitchInputBoolean`.
- **Six pre-existing defects found and fixed** (none were the task): `valid_visa` silently discarded on
  every registration; room inventory leaking on re-assignment; `GuestsExportFlights` exporting 20 blank
  columns; the visa PDF shipping literal `contact-email` placeholder text to embassies; every e-visa
  admin endpoint 404'ing on the wrong base path; and the `inferFeatureId` first-match-wins trap three
  times over.
- **Deliberately NOT ported:** February's e-visa "operations console". It is absent from hci `main`
  (never merged, abandoned), and it drives `e_visa_status` with `pending`/`in_process`/`received`
  against the shipped `in_progress`/`issued` — two state machines on one column.
- **Commits:** backend `303c629`, `0ed06f3`, `0ea04fb`, `89f1673`; admin `8641f65`, `01764ab`,
  `83ba223`; frontend `b0e7e49`.
- **Gates:** backend `pint --test` + `phpstan` **No errors** + `migrate:fresh --seed` clean; admin
  `yarn type-check` + `next build` **123/123**; frontend `type-check` + build clean; EN/AR parity
  1611/1611. **Mobile contract unaffected** (all new routes `/admin/*`, nothing removed/renamed) — no
  notice issued.
- **Still open:** the push itself (the `composer qa` gate runs `php artisan test`, which wipes the dev
  DB — see the phpunit.xml note below); `sendIssuedVisa` not ported, leaving `issued_visa_send_count` /
  `issued_visa_sent_at` written by nobody; the logistics screen's `Promise.all` failing hard for an
  admin with `guest_logistics` but not `guests_listing,see_more`; and manual QA of the whole flow.
- **⚠️ `phpunit.xml` runs `RefreshDatabase` against the real MySQL dev DB.** The obvious fix
  (uncommenting the sqlite/`:memory:` lines) makes it worse — 5 failures vs the documented 3, because
  `EventDaysTest` is MySQL-dependent. Unresolved; it blocks a clean `composer qa`.

**2026-07-21 — Single-channel invitations (D30) + SMS logs (D31) + categories comms restructure + P17 UX
batch. ALL PUSHED across backend / admin / frontend `dev`; docs on `main`.**
- **Single-channel invitations (Task 017, ledger D30):** an invitation collection now sends on **exactly one**
  channel (`email` | `sms`; `whatsapp` reserved/disabled), reversing D29's parallel send. `channel` enum
  folded into migration `000006`; store/update scope the template + provider to the chosen channel; extract +
  collection-edit propagate `channel`; `invite` guard is channel-aware (checks `phone` for SMS); SMS success
  bumps `is_sent`/`send_count`. Admin: channel picker gated on a configured default (disabled + link when
  not), per-channel override sections on **both** create and collection-edit forms, channel badge in listing,
  SMS-history tab in see-more, wider bulk-send modal (channel/phone/template columns), reorganized update-info
  modal, `DialogShell` 4xl/5xl. Backend `0fcd3c5`, admin `f1589df`.
- **SMS logs (Task 018, ledger D31):** read-only Guest SMS + Invitation SMS log pages mirroring the email
  logs, behind a new `sms_logs` RBAC feature (view/export); `guest_sms` + `invitation_sms` each get a
  controller/resource/export + super-gated `/logs/*` page + sidebar link. Backend `34e09e7`, admin `8b0960f`.
- **Categories comms restructure (ledger D31):** master `with_email` / `with_sms` switches, **enforced** in
  `Category::getNotificationTemplate` (master off → never sends on that channel); `with_otp` →
  `with_email_otp` rename in lockstep across backend + admin + **frontend join pages** (mobile payload rename
  → notice `docs/mobile/MOBILE_NOTICE_CATEGORY_WITH_EMAIL_OTP_RENAME.md`); new "Admin access" tab +
  `assignable-admins` endpoint; validation-error surfacing; form width/gating. Backend `3c73f0f`, admin
  `702d9b1`, frontend `0d0d82b`.
- **P17 UX batch (earlier this session, admin):** gate scanning → its own `/gate-scan` page (`dac5c14`);
  sidebar SMS grouped under Communications + SMS-templates link (`2cf9409`); categories 5-tab form + provider
  override switches + OTP gating (`4c83678`); status dropdown → toggle switch across entity forms (`69ddd13`).
  Seeder: categories seeded with OTP off until a default SMTP exists (backend `c50ebb8`).
- **Also:** guests listing bottom spacing (admin `8c57f55`); the previously-unpushed LinkedIn OAuth commit
  (frontend `e8d7991`, Task 012) was pushed in the same round.
- **Gates:** backend `pint --test` + `phpstan` **No errors**; `composer qa` tests **465 pass / 3 fail** (the
  3 are pre-existing avatar-URL + `/`→403 env failures — verified my diff touches none of those areas). Admin
  + frontend `yarn type-check` + eslint/prettier (husky) green. **`yarn production` NOT run** — needs the
  gitignored `.env.production`. **Remaining:** manual QA with live SMTP/SMS providers; automation-form UX
  polish (mirror the invitations pass); WhatsApp channel (deliberately deferred — next big piece).

**2026-07-20 — Task 016 (SMS flow parity) code COMPLETE across backend + admin `dev` (ledger D29). ⚠️
Backend + admin + docs are UNCOMMITTED working-tree changes.**
- **What:** closes the SMS-vs-email gaps. Before, SMS only fired on register-complete + phone-OTP; now it
  fires on **accept/reject**, **automations**, and **invitations** too — each with its own optional SMS
  provider override, mirroring D27/D28.
- **Stage 1 (accept/reject):** `GuestsController` accept/acceptToCategory/reject create a `guest_sms` row
  (snapshot `sms_config_id`) + dispatch `SendGuestSMSEvent`; category "SMS notifications" picker relabelled
  register/accept/reject.
- **Stage 2 (invitations):** migration `2026_07_20_000006` — `invitations`/`invitation_collections` gain
  `sms_template_id`+`sms_config_id`; new `invitation_sms` table. New `InvitationSms` +
  `SendInvitationSmsEvent`/`SendInvitationSmsListener` (invitation-specific placeholders) fired from
  invite/bulk/reminder alongside the email; extract-bulk inherits/overrides the SMS template. Admin SMS
  template + provider pickers on invitation + collection forms + the extract modal.
- **Stage 3 (automations):** migration `2026_07_20_000007` — `automation_setups` gains `with_sms_template`
  + `sms_template_id` + `sms_config_id`. `AutomationController::send` creates a `guest_sms` row + dispatches
  `SendGuestSMSEvent` per guest when the toggle is on (independent of email); `split` carries the fields.
  Admin `with_sms_template` toggle + SMS template/provider pickers on the automation form.
- **Design:** guest-backed flows (accept/reject, automations) **reuse `guest_sms` + `SendGuestSMSEvent`**
  (same path as register-complete, non-prod block still applies); token-based invitations get their **own**
  `invitation_sms` table + listener (mirroring `invitation_emails`). Provider override rule = D28
  (blank/inactive → active-default, snapshot at create).
- **Touched (backend):** migrations `000006`+`000007`; `InvitationSms` + invitation SMS event/listener (+
  `EventServiceProvider`); `InvitationsController`/`InvitationsCollectionController` persist+dispatch;
  `AutomationSetupsController` store/split; `AutomationController::send`; `AutomationSetup`/`Invitation`/
  `InvitationCollection` fillable+casts; `Invitation`/`InvitationCollection`/`AutomationSetups` resources.
- **Touched (admin):** `sms-template-select` (`errors` prop optional) + provider/template pickers on
  invitation / collection / extract-bulk / automation forms + `with_sms_template` toggle + interfaces + EN/AR
  (`sms_override`, `with_sms_template`).
- **Gates:** backend `pint --test` passed + `phpstan` No errors; admin `yarn type-check` + eslint green;
  `mobile/*` untouched. **Still open:** manual QA with a real active provider. **Needs commit + push.**

**2026-07-20 — Task 014 (OTP SMS → dynamic provider + per-flow SMS override) code COMPLETE across backend +
admin `dev` (ledger D28). ⚠️ Backend + admin + docs are UNCOMMITTED working-tree changes.**
- **What:** deleted the hardcoded FGC OTP gateway (`cnc.fgc.sa` + committed `sdbankApi`/`SDB` password) from
  `AuthController` — **no static SMS code left**. Phone-OTP now flows through the DB-driven provider stack
  (D26: `SmsProviderConfig` + `SmsSender`, Unifonic today). Added the SMS mirror of D27's SMTP pickers:
  category has **two** SMS provider pickers — register-complete + phone-OTP. Blank/inactive → active-default.
- **⚠️ Behaviour change:** OTP calls `SmsSender` directly, so it still sends on **dev/stage** (a code the
  user waits for) — but now via the **real active-default provider**, not the old FGC test gateway. Register/
  notification SMS keeps the listener's non-prod block.
- **Touched (backend):** migration `2026_07_20_000005` (`categories.sms_config_id` + `otp_sms_config_id`,
  `guest_sms.sms_config_id`) + `AuthController::phoneVerification` rewrite + `SendGuestSMSListener` snapshot +
  `GuestsController` register/resend snapshot + `Category`/`GuestSMS` fillable + `CategoriesController`
  validation/persist + `CategoriesResources` + `SmsProviderConfigController::selectList` + `GET
  admin/sms-provider-configs/select`.
- **Touched (admin):** reusable `sms-provider-config-select` + two category pickers + `interfaces/category`
  + EN/AR. No frontend change (join form already forwards the `category` slug).
- **Gates:** pint + phpstan clean; admin `yarn type-check` + eslint green; `mobile/*` untouched. **Still
  open:** manual QA with a real active provider. **Needs commit + push.**

**2026-07-20 — Task 015 (per-flow SMTP override) CLOSED + pushed (ledger D27).** Backend `2adc387`, admin
`76ae079`, docs `8c9ef34`.
- **What:** admins can pick which SMTP account sends each email flow instead of always the default.
  Category has **two** pickers: notifications (register/accept/reject) + guest email-OTP. Also overrides
  on automations and invitations/collections. Blank = default. Inactive/deleted override → fall back to
  default. Automation override beats `MAIL_HOST_BULK`. **Still open:** manual QA with multiple SMTP accounts.

**2026-07-20 — Task 013 (SMS provider config) CLOSED + pushed (ledger D26).** Backend `96a15ce`, admin
`661f134`, docs `f3802b3`. Manual Unifonic prod QA still pending.

**2026-07-20 — Task 012 (LinkedIn automatic "Share on LinkedIn") code COMPLETE across backend + admin +
frontend `dev` (ledger D25). ⚠️ All three apps + docs are UNCOMMITTED working-tree changes.**
- **What:** completed the `automatic` half of the per-category social share the admin form already
  advertised. Per-category LinkedIn app creds (`linkedin_client_id`/`linkedin_client_secret`, additive
  migration `2026_07_20_000001`) → `LinkedInController` (auth-url / call-back / post, cyan's Pint/phpstan-clean
  version, v2 consumer Share surface) → 3 **public** routes → guest OAuth flow on the success page that posts
  the generated social card. ALT keeps its **blade** social card (cyan's layout-designer Tier B was out of
  scope). Best-of-both port (cyan P37.4 + hci) adapted to ALT conventions.
- **Touched:** backend `LinkedInController` + migration + `Category` + `CategoriesController`
  (`getVisibility` now returns `share_type`; `update()` + `CategoriesResources` round-trip the creds) +
  new `config('app.frontend_url')` (phpstan: no `env()` in a controller) + `routes/api.php`; admin
  `categories-form.tsx` (creds inputs when `share_type=automatic`) + `interfaces/category.tsx` + EN/AR
  `web.json` (incl. `share_manual_hint`/`share_automatic_hint`); frontend new `linkedin-redirect.tsx` +
  `success/sharebtn-sections.tsx` (automatic OAuth flow, ALT-native lucide/toast/getApiError) +
  `success-sections.tsx` + `join/[category]/success.tsx` (thread `share_type`+`category_slug`).
- **Gates:** backend `composer qa` green (pint + phpstan No errors + tests 465/3 pre-existing); admin +
  frontend `yarn type-check` + eslint green; `mobile/*` untouched. **Still open:** manual QA needs a real
  LinkedIn "Share on LinkedIn" app + `PUBLIC_FRONTEND_URL` set + a running stack. **Needs commit + push.**

**2026-07-20 — Task 010 (api.php cleanup/reorg/RESTful rename) CLOSED (ledger D24). ⚠️ Backend
`routes/api.php` + docs edits are UNCOMMITTED working-tree changes — not yet committed/pushed.**
- **Reconciliation:** Tiers 0–4 were already committed + pushed earlier (backend `4cf7036`→`c5a3a31`→
  `9328d65`→`68723ee`, admin `e36b384`, frontend `53d42e0`) and Task 011 built on top. The "uncommitted,
  pending review" notes in the old task log were **stale** — the cutover had shipped. Re-verified the
  committed baseline green: backend `composer qa` (pint + phpstan No-errors + tests 465/3 pre-existing).
- **Final leftover folded in (this session, uncommitted):** the last two non-RESTful, ungated, zero-caller
  endpoints `POST /admin/guests-upload-zip` + `POST /admin/match-guests-images` → renamed into the guests
  group as `POST /admin/guests/upload-zip` + `/match-images` behind `admin.can:guests_listing,edit`. Pure
  rename (route count stayed 384); no in-repo or mobile caller to move; controller methods unchanged.
- **Left as-is:** the four offline-sync endpoints (`attend`, `guest-data-offline`, `guest-data-sync`,
  `guests-printed-since`) stay at their `/admin/*` URIs behind `admin.can:scanning` — the deliberate Task
  011 (D23) decision, not re-litigated.
- **Gates:** backend `composer qa` green (465/3 pre-existing); dead-link grep across all repos for every old
  path = 0 code references; `mobile/*` untouched → no mobile delta. **Still open:** manual browser smoke
  test per renamed/gated feature (needs a running stack + role matrix). **Needs commit + push** (backend +
  docs).

**2026-07-19 — Task 011 (scan-into-admin) code COMPLETE on backend + admin `dev` (ledger D23). Gate
scanning is now a first-party, RBAC-gated admin feature; the standalone "agent admin" scanner is retired.**
- **What:** ported the on-site scanner into the admin dashboard (from 108/112) and wired it onto ALT's RBAC
  — no `admins.type='gate'`, no separate agent login. New **`scanning`** catalog feature (`['view']`,
  distinct from `gates`/`areas`/`scans`); the dashboard shows the scanner when
  `checkFeaturePermission('scanning', user)` is true.
- **Backend `211e17d`→`cd66c21`:** new `/admin/gate-scan` group behind `admin.can:scanning`; server-side
  **data-scope enforcement** (`GatesController::deniesGateScope()` — bound `gate_id` and/or `area_id`, super
  short-circuits); dropped `loginAgent` + route + the 4 no-`/admin` scan aliases + the dangling
  `validate-check-in` route; offline-sync endpoints kept but re-gated; `admins/select` now filters by
  `scanning`; `tests/Feature/GateScanTest.php` (happy-path, RBAC 403, area/bind scope denial, super bypass,
  recovery search+link).
- **Admin `1c87ff0`:** `components/admin-modules/dashbaord/gate-scan/*` (setup / current-gate / camera +
  hardware-wedge modes / wrong-QR recovery), wired to the renamed endpoints **through the BFF proxy**
  (HttpOnly token, no manual Bearer). Camera lib **`react-web-qr-reader` → `html5-qrcode`** (maintained,
  dynamic-imported); dropped `react-lottie` for a CSS pulse (net deps ≈ 0). Recovery now **searches by name
  and links the guest to the orphan scan** (108/112 only uploaded a photo). Re-enabled "Gates & Scans" nav,
  added `/scans` icon, removed dead `type=gate` bits. EN + AR in the same commit.
- **Lifts Task 010's RENAME_MAP §F freeze** — recorded in a new §G there.
- **Deferred / open:** the `scans.gate_id` FK (still on the `gate_name` string link); and **live browser-QA +
  the RBAC/scope manual matrix** (needs a running stack + camera) — validated so far by feature tests +
  type-check/build only.
- **Gates:** backend `composer qa` green (pint + phpstan + tests incl. `GateScanTest`); admin `yarn
  type-check` + `next build` green (`yarn production` needs the gitignored `.env.production`). **Not
  mobile-facing** (`routes/api.php` mobile surface intact). **PUSHED** — backend `origin/dev` `cd66c21`,
  admin `origin/dev` `1c87ff0`, docs `origin/main` `eafc056`.
- **Queue parked (user, 2026-07-20):** the remaining "done (code) — QA pending" items (011 live
  browser-QA, 005 HttpOnly, 006 private docs, 007 RSVP), the deferred `scans.gate_id` FK debt, task 001
  boolean-cleanup, and the mobile-ack chase are all **intentionally set aside** — user is starting a new
  type of task. Pick these back up later.

**2026-07-19 — Task 007 (API response unification) COMPLETE on backend `dev` (ledger D22). All controllers
now return the standard `ApiResponse` envelope; mobile (Tier C) deltas documented + IMPLEMENTED.**
- **What:** admin (Tier A/B) landed earlier this session; this pass finished **mobile (Tier C)** — every
  `mobile/*` controller migrated to `BaseApiController` + `apiSuccess`/`apiError` (`MobileAuth`,
  `MobileEventDay`, `MobileSpeaker`, `MobileSponsor`, `MobileAttendee`, `MobileSession`(+`Feedback`),
  `MobileWorkshop`(+`Feedback`), `MobilePublication`, `MobileMediaCenter`, `MobileQr`, `MobileRoom`,
  `MobileNotification`, `MobileChat`). Payloads now live under `data`; `success`+`status`+`message` added.
- **`AppConfigController` intentionally left unwrapped** (config documents, not resources — delta §18).
- **Mobile break is documented:** `docs/mobile/RESPONSE_SHAPE_DELTAS.md` flipped **PLANNED → IMPLEMENTED**,
  per-endpoint. This is the "adapt later" artifact — the mobile team adapts the Flutter client against it.
- **`routes/api.php` unchanged** (body refactor only). Backend feature tests updated in lockstep.
- **Gates:** pint + phpstan **No errors**; tests **457 pass / 3 fail** (the 3 are pre-existing D14
  signed-avatar/env failures, not from this work).
- ⏳ **Blocked — mobile ack:** backend `dev` → `main` now waits on the mobile team acknowledging **both** the
  D14 avatar signed-URL change **and** these D22 envelope deltas. User will bring the mobile repo into the
  parent project folder soon and update the Flutter client directly.

**2026-07-18 (session 2) — Four-step guest-draft `invitation_token` gap closed + task board tidied.**
- **invitation_token (task 008 follow-up, DONE + PUSHED):** the pif **four-step** form now forwards the
  invitation token into its OTP request, so abandoned four-step registrations capture `invitation_token`
  like one-step forms already did. Frontend `98cb380` — 2 files: `renderFormSteps.tsx` (the
  `personal-info-1` branch was the only one dropping `token`) + `pif/fours-steps/step-1.tsx` (prop +
  `formData.invitation_token`). Gates green (`type-check` + full `next build`). Closes the last known
  limitation from task 008 (D19). Backend already accepted the field — no backend/mobile change.
- **Task board:** **005** (admin HttpOnly) + **006** (private doc storage) marked **`done`** — code
  shipped + pushed on `dev`; the `dev`→`main` merge is **deferred to the user's own repo check**. ⚠️ 006
  still needs **mobile-team ack** (avatar → 24h signed URL) before that merge.
- **009 (useFetch) — ✅ CLOSED 2026-07-19 (by user).** Clean-candidate set fully converted (14/14, 19
  adopters); convention JSDoc lives atop `admin/hooks/useFetch.ts`. Further reach needs the
  `enabled`/`refetch` hook extension (its own task if/when needed); new fetch-once screens adopt it
  opportunistically. Removed from the active planning queue.
- **004** migration-squash is being **re-planned separately** (user + another agent); **001** boolean
  cleanup parked for a later check.
- ⏳ **Uncommitted (awaiting user review):** the doc updates above + the `useFetch.ts` convention note are
  edited but **not yet committed** (admin + docs repos).

**2026-07-18 — Guest-drafts feature shipped (D19). Abandoned-registration capture, ported from deve-go
`60fe949`; the admin UI existed across clones but its backend was never built. PUSHED, in-browser QA'd,
all app repos in sync. Task: `tasks/008-guest-drafts-port/TASK.md`.**
- **What:** a registrant who requests an OTP but never finishes is upserted into a new self-contained
  `guest_drafts` table (keyed by email), deleted on completion → a follow-up/drop-off list for the event
  team. Backend `7a96707`, admin `270a60d`, frontend `a8a94ec`.
- **Dedicated `guest_drafts` RBAC permission** (view/export/see_more), **route-enforced via `admin.can`** —
  grantable independently of `guests_listing` (and stricter than it — a deliberate deviation, since
  `guests_listing` routes have no `admin.can` gating). Shows as its own row in the roles editor.
- **Captures** gender/title/personal_image (frontend OTP payload now sends them), `category` (slug →
  `category_id`), and `invitation_token`; `personal_image` served as a **signed** URL (D14). Employee-ID +
  Days dropped from the see-more modal.
- **Known limitation:** the pif four-step form has no invitation-token prop → its drafts don't capture
  `invitation_token` (one-step forms do). **task 009** (useFetch adoption) remains the next parked item.

**2026-07-17 — Env templates unified (D17) + guest document/day fields completed (D18). All PUSHED; all
four repos clean and in sync with `origin`. Gates green throughout.**
- **D17 — one tracked `.env.example_prod` per app.** Backend `.env.example2`→`.env.example_prod`
  (`c7dd2ee`, `4dfdca9`); admin `.env.example`→`.env.example_prod` (`a5e83e6`→`0dcf74a`); frontend gained
  its **first** tracked template (`27bad95`, `a22bd5b`). Frontend cookie-age env vars → code constants
  (`7248f39`, `b5df5d3`, `220e65e`), parity with admin `a361586`. `CLONE_CHECKLIST` corrected (docs
  `2b118d5`). Backend env audited against Laravel 12.62 — **nothing unsupported**; it deliberately uses the
  **old** var names (`CACHE_DRIVER`/`BROADCAST_DRIVER`) because `config/` reads those — don't "modernise".
- **D18 — guest fields that had UI but no data layer.** `visa_copy`/`issued_visa`: upload 422'd
  ("failed to verify path name") and had **no column / no `$fillable` / no save** — files were silently
  dropped. Fixed `00fe02a` (allow-list + TYPES) → `4883f9d` (columns, persistence, signed `*_url`).
  `days`: a **phantom** field whose filter returned **500**; shipped the column (`ef218f6`) since only
  `98-pif-2026` had ever added it — **still has no writer** (write sites commented, no UI submits it).
  Also guarded `json_decode(null)` on `days` + `interests` (deprecation on every guest response), and
  repaired `GuestFactory` (dead `status` column) + added `CategoryFactory` (`9042919`).
- **Email:** admin-invite rebranded onto the OTP branded base template (`3e19f36`); `EmailTemplatesSeeder`
  genericised — no more TOURISE naming (`7603810`). **`EmailConfigSeeder` was checked and is clean.**
  `event_name_en` is NULL on a fresh clone **by design** — the super admin sets it after deploy.
- **Join form (frontend):** remove-photo button was an invisible X on a black circle → self-contained
  `CircleX` (`c548551`); photo-consent + app-visibility toggle hidden and `photo_consent`'s `required`
  dropped so the form still submits (`d010d3d`); terms text unlinked; visa label now says jpg/png, not PDF
  (`3c64c8a`) — the endpoint only accepts `jpeg/jpg/png`.
- **⚠️ Regression caught + fixed:** D16's sweep missed the **Blade email templates** — 22 refs, so every
  email poster rendered `nullemails-config/…` once the var was dropped. Fixed `a9a1ed4`. See the D16
  addendum.
- **Resolved (D21):** frontend GTM is kept and correctly wired — code reads `NEXT_PUBLIC_GTM`, and the var
  ships in `.env.local` + `.env.example_prod` (empty = disabled until a clone sets `GTM-XXXX`). Admin uses
  no GTM at all.

**2026-07-13 — Storage-URL env-var consolidation DONE (all 4 phases, ledger D16 + its 2026-07-17
addendum). Pushed. Plan: `upgrades/STORAGE_URL_CONSOLIDATION_PLAN.md` (status = DONE). Mobile contract
UNCHANGED (byte-identical URLs, tinker-verified) → no ack needed.**
- **Backend now keeps ZERO `PUBLIC_STORAGE_URL*` vars**, admin keeps ONE (`NEXT_PUBLIC_STORAGE_URL` =
  storage root + `utils/storage.ts`), frontend ZERO. Commits: FE `89c1ce3`; admin `b5bb5b2`→`9137fd9`→
  `fd628cd`; backend `58ca08c` (new public `social_card_image_url`) + `5cebb86` (46 `env('PUBLIC_STORAGE_URL2')`
  sites / 28 files incl. 7 mobile resources → `Storage::disk('public')->url()`; phpstan baseline pruned
  45→18 env ignores, masking nothing). Also earlier `efcc027` (dead `// 'url' =>` comment cleanup).
- **Gates:** FE+admin `type-check`+`production` green; backend `composer qa` green (pint + phpstan **No
  errors** + **452 pass/3 fail** pre-existing, confirmed identical on stashed parent) + `migrate:fresh
  --seed` clean.
- **Bugs fixed en route:** `social_card` see-more URL had a wrong path (`/social_card` vs
  `/uploads/social_card`) — now from the API; `visa_copy`/`issued_visa` confirmed dead (no column) so no URL
  added. **Deferred correctness item:** guest `custom-file-input{,-3}.tsx` still rebuild a *public* URL for
  now-*private*-disk files (the `/upload` endpoints return `data` only) — likely broken preview; fix = return
  a signed `url` from those endpoints. Tracked in the plan follow-up.
- **`.env` edits — ✅ DONE by the user (2026-07-17).** admin `.env.local` retargeted
  `NEXT_PUBLIC_STORAGE_URL` → `…/storage` and dropped `NEXT_PUBLIC_STORAGE_URL2` +
  `NEXT_PUBLIC_STORAGE_URL_ATTACHMENTS`; frontend `.env.local` dropped `NEXT_PUBLIC_STORAGE_URL`;
  backend `.env` dropped `PUBLIC_STORAGE_URL` + `PUBLIC_STORAGE_URL2`. Originals kept under `backup.env/`.
  (Any **new** clone must retarget the same way — the code appends `/uploads` to the root, so a
  `…/storage/uploads` value double-suffixes.)

**2026-07-13 — Two Tailwind v4 regression fixes (Saudi `FIX_TAILWIND_V4_REGRESSIONS.md`, ledger D15).
Committed on `dev`, gates green — NOT pushed. className-only, no logic/backend/mobile impact.**
- **Fix 1 — error focus-ring → `/50`** (v4 dropped `ring-opacity-*`): admin `5d99b43` (11 files) + frontend
  `052f16f` (5). **Fix 2 — drop `rtl:space-x-reverse`** (v4 `space-x-*` is now RTL-aware → the class
  double-flips): admin `aadadf8` (124 occ/69 files) + frontend `721d458` (25 occ/9). Four separate commits.
- **Gates:** `type-check` + `production` green on both apps. **Visual EN/AR QA pending** (soft red ring on
  invalid inputs; RTL spacing on checkboxes/radios/back+share buttons/toolbars).
- Note: an early scripted Fix-2 attempt corrupted indentation (global whitespace collapse); fully reverted
  and redone surgically. Final diffs are proportionate (one line per token).

**2026-07-12 — Private document storage + signed URLs (Saudi P1 backport, task 006, ledger D14). Code
DONE on backend `dev`, gates green, runtime-verified — NOT yet committed/pushed (working tree). MOBILE
CONTRACT CHANGE — hold `dev`→`main` until mobile acks. Plan: `CLEANUP_AND_HARDENING_MASTER_PLAN.md` Task
005 (Track B); log: `tasks/006-private-document-storage/TASK.md` (folder 006).**
- **What & why:** registrant PII (`guests.personal_image` photos + `document_copy` passport/ID) was on
  the **public** disk at raw unauthenticated CDN URLs — anyone with the URL fetched a passport. Now on a
  **`private`** disk (`storage/app/private`, never web-served), served only via short-lived **signed
  URLs** (`GuestDocumentController` + `signed`-middleware route `GET /api/files/guest-doc/{type}/{file}`).
  Un-deferred pre-launch (no clone has prod data).
- **Landed (backend, 13 files + 2 new):** private disk; serving controller (signedUrl + stream w/
  basename traversal guard, allow-list, no-store); writes repointed incl. **mobile avatar upload**
  (`MobileAuthController`); admin resource URLs → 30m signed, **mobile `avatar` → 24h signed**; **16
  server-side read-backs** (badges/PDF/social-card/email-photo, 7 files) → `disk('private')->path()`;
  idempotent `guests:migrate-docs-to-private --dry-run` command; 2 stale phpstan-baseline env ignores removed.
- **Gates:** `composer qa` green (pint + phpstan No-errors + tests **452/3 pre-existing** — the 2 avatar
  failures confirmed to fail on the clean parent too → no regression); `migrate:fresh --seed` clean.
  **Runtime verified:** private file streams **200** on valid signature, **403** on tamper/no-sig, **404**
  on traversal, **404** at the old public `/storage` path.
- **MOBILE CONTRACT:** `avatar` is now a signed 24h-expiring URL (same field/type; mobile must re-fetch
  after expiry, not build the URL). Flagged in `docs/mobile/MOBILE_NOTICE_PRIVATE_AVATAR_SIGNED_URL.md`.
- **Outstanding:** commit + **mobile team ack** + real-env QA (admin preview render, heavy export/PDF/email
  photo) before `dev`→`main`. NOTE: the plan's bundled UploadService extraction (Todo-2D) was NOT done.

**2026-07-12 — Fixed `migrate:fresh --seed` (TitleSeeder null bug, ledger D13). Backend `dev` `a6fe3d1`,
pushed.** 6 `TitleSeeder` rows passed `show_in_user_form => null` into a NOT-NULL `boolean default(false)`
column → `SQLSTATE[23000]` crash mid-seed. Fixed the **seeder** (`null` → `false`; `null` meant "not
shown"), not the schema. This was the pre-existing bug the dropped migration-squash recon flagged. Verified:
full `migrate:fresh --seed` clean, 8 seeders green, 12 titles (6 shown / 6 hidden), Pint + Title tests pass.
Not mobile-facing.

**2026-07-12 — Admin HttpOnly token + Next BFF proxy + full CSP (Saudi P2 backport, task 005, ledger
D12). Code DONE, gates green, runtime-verified — committed + pushed (admin `dev` 4 commits
`d95a2e5`→`b006123`; docs `main` `2939d0b`). Real-env browser QA still outstanding before `dev`→`main`.
Plan: `upgrades/cleanup-hardening/CLEANUP_AND_HARDENING_MASTER_PLAN.md` Task 004 (Track B); log:
`tasks/005-admin-httponly-token/TASK.md` (folder 005 — 004 is the dropped squash).**
- **What & why:** the admin bearer was a JS-readable cookie (XSS → account takeover). The Phase-1 fix
  (`af2298b`, secure+sameSite) couldn't close the XSS-read vector — only `httpOnly` can, and only a server
  can set it. So the token now lives ONLY in an **HttpOnly cookie** written by a **Next BFF proxy**; the
  browser never handles it. Un-deferred from Track B now because the basecode is **pre-launch (no clone has
  prod data)**, so the 135-file codemod is cheap to bake into every clone.
- **Landed (admin only, 144 files, net −399 lines):** new `utils/{auth-cookies,server/proxy}.ts` +
  `pages/api/{proxy/[...path],auth/{login,login-confirmation,logout}}.ts`; isomorphic `utils/axios.ts`
  (browser→`/api/proxy`, SSR→direct); provider/withAuth/login+verify onto a JS-readable flag cookie
  (`alt_admin_auth`) + `authenticated` marker; codemod removing 136 dead `cookie.get('token')` reads + 261
  `Authorization: Bearer` headers (proxy injects auth server-side now); **full CSP** in `next.config.js`
  adapted to alt (env origins, reCAPTCHA only, **no iconify** per D5, `'unsafe-eval'` dev-only).
- **Gates:** `yarn type-check` + `yarn production` **green**. **Runtime verified** against a stub upstream
  on `next dev`: login strips token + sets `HttpOnly; SameSite=Strict; Max-Age=6h` cookie, proxy injects
  `Bearer` from the cookie, logout clears both, OTP + multipart streaming + CSP header all confirmed.
- **Mobile:** untouched — admin-web only, `routes/api.php` unchanged.
- **Outstanding:** commit (4 commits) + **real-env browser QA** (live backend login, reCAPTCHA, heaviest
  export/upload through the proxy) before `dev`→`main`. Saudi hotfix-reverted their P2 once over these edges.

**2026-07-11 → 07-12 — Env-var / dead-code cleanup pass (admin + frontend). Committed on `dev`.
Frontend pushed and in sync with `origin/dev` (`64037eb`). Admin was 4 ahead of `origin/dev` at the time
— `f6bcf7b` → `a361586` → `37cf1a1` → `8345f19`; all four pushed since, and on `main` from 2026-08-06.**
- **Admin (the 4 commits):** retired baseline env vars that were config-noise, moving the values
  to code constants. `f6bcf7b` drop `NEXT_PUBLIC_LISTING_PER_PAGE_LIMIT` from listing URLs
  (`utils/fetch-data-url.ts`, print-logs). `a361586` move cookie-age env vars → code constants
  (`auth/provider.tsx`, `i18n/provider.tsx`). `37cf1a1` retire `NEXT_PUBLIC_ENV` from 9 `data/*-select.tsx`
  files (incl. `status-types-select`, `sidebar-links`). `8345f19` remove unused `callback_url` /
  `back_link` from `guests` step-4 + `verify-email-form`. Pure config/dead-code hygiene, no behaviour
  change.
- **Frontend (pushed, `dev` @ `64037eb`):** `9a9a850` + `dedb4f6` clean up `utils/axios.ts` (drop unused
  token header/variable). `e75c9bf` remove `@vercel/analytics` dep + imports (`package.json`, `_app.tsx`,
  `yarn.lock`). `64037eb` untrack `.env.production` + add to `.gitignore`.
- **Gates:** admin `yarn type-check` **clean** + `yarn production` **green** (verified 2026-07-12).
  Not mobile-facing (`routes/api.php` untouched). **Admin still needs its 4 commits pushed to `origin/dev`.**

**2026-07-08 — Backend tooling & code-quality chain (task 003, ledger D10). All work items DONE;
committed on `dev` — backend `96413df` (W1, already pushed) + `bb61db9` (W2+W6) + `9741e90` (W5+W8)
+ `de75eed` (W7); docs on `main`. Backend `dev` is 4 ahead of `origin/dev` (W1 pushed earlier). Plan:
`upgrades/BACKEND_TOOLING_CHAIN_PLAN.md`; log: `tasks/003-backend-tooling-chain/TASK.md`.**
- **What landed:** brings the backend's quality chain to parity with the Next-app pass. **W1** —
  `pint.json` (laravel preset + `no_unused_imports` + `ordered_imports`) + one repo-wide Pint baseline
  (172 files, formatting-only) → repo is now **Pint-clean**, gate flips `pint --dirty` → **`pint --test`**.
  **W2** — **Larastan** static analysis at **level 0** + generated baseline (124 real structural
  errors), with a committed **ratchet** (shrink → bump the level toward 6; runs in `composer analyse`/
  `qa`, never the hook). Fixed 1 non-ignorable finding at source (`GuestOtpNotification::$locale`
  redeclared a native type over Laravel's untyped parent). **W6** — composer scripts `lint`/`lint:fix`/
  `analyse`/`test`/`qa` (one-command gate). **W5** — PHP-native `.githooks/pre-commit` runs Pint on
  staged `*.php` + graceful-skips if Pint absent (parity with admin/FE husky+lint-staged), auto-installed
  via `composer install`. **W7** — `.vscode/` **un-ignored + committed** with `[php]`→Pint (fixed an
  inconsistency: admin/FE tracked `.vscode/`, backend gitignored it). **W8** — dropped stale
  `pestphp/pest-plugin` allow-plugin; `composer validate --strict` valid; audit clean. **Rector + CI
  left out** (out of scope). Not mobile-facing (`routes/api.php` untouched).
- **Gates:** `composer qa` = `pint --test` green + `phpstan analyse` **No errors** (baseline-green) +
  `php artisan test` **452 pass / 3 fail** (pre-existing, unrelated). **New backend gate going forward
  is `composer qa`.**

**2026-07-07 (later) — `catch (e: any)` → `unknown` cleanup, closing the "cheap cleanups" phase
(ledger D9). Committed on `dev` (admin `5ceacc3`, frontend `8544c39`) — NOT yet pushed (part of the
same review batch as task 002). All original audit sub-phases ("fix first", "cheap cleanups") are now
complete; "later/opportunistic" is parked — see `tasks/PHASE3_PARKED_TODO.md`.**
- **What landed:** 94 `catch (error: any)` blocks across 82 files (68 admin + 14 frontend) → `catch
  (error: unknown)`. New shared helper `utils/api-error.ts` (`getApiError(unknown) → typed axios
  `ApiErrorResponse | undefined`) in both apps; response-reading catch bodies route through it, log-only
  bodies just re-annotate to `unknown`. Behaviour unchanged (same branches/toasts/status checks). Casts
  added only where an untyped value flows into a typed sink (RHF `setError`, `toast.error`). Gates green:
  `type-check` + lint 0 warnings, both apps. Not mobile-facing.
- **Also this session:** verified the other three "cheap cleanups" items were already done by a prior
  agent (iconify→lucide, 28+2 dead files deleted, commented-`// console.*` swept) — all confirmed clean
  against current code. Wrote **`docs/mobile/MOBILE_NOTICE_AGENDA_DATE_WALL_CLOCK.md`** — an actionable
  notice for the mobile team about the D8 venue-local `date` change (must not TZ-convert agenda `date`).

**2026-07-07 — Date/time (timestamp) DB cleanup + refactor (ledger D7, task 002). Committed on `dev`
(backend `86961dd`, admin `c6ee625` + follow-up `f340a0e`, frontend `5f2c55a`) and `main` (docs
`aeb4528` + `155d94b`) — NOT yet pushed (awaiting review). Full plan + per-step log:
`tasks/002-datetime-db-cleanup/TASK.md`.**
- **Display consistency pass (admin, `f340a0e`):** 34 views switched from `format(new Date(x))`
  (viewer's browser TZ) → shared `formatDateTime` (`Asia/Riyadh`) for real UTC Laravel timestamps
  (listing `created_at`, `registered_at`, session media `created_at`, guest-draft
  `created_at`/`updated_at`). Export-filename timestamps left on `format()`.
- **Agenda-date fix (ledger D8, MOBILE CONTRACT CHANGE):** session/workshop `date` was served as UTC
  `…Z` but entered as naive `datetime-local`, so the admin edit form pre-fill shifted the time −3h on
  every save. Switched all 9 `date` serializations (admin + mobile resources) from `->toISOString()`
  to naive-local `->format('Y-m-d\TH:i:s')` (venue = Asia/Riyadh). No FE logic change (pre-fill +
  `format(new Date())` display self-correct with a `Z`-less string). **Mobile must parse `date` as
  wall-clock, NOT convert to device TZ** — flagged in `docs/mobile/…FOR_MOBILE.html` §24. Backend
  `pint --dirty --test` green.
- **Backend:** date-only columns → real `date` (cast `date:Y-m-d`), datetime columns → `timestamp`
  (cast `datetime`, ISO 8601 UTC), flight times → `string(5)` `HH:mm`. Migrations edited in place +
  `migrate:fresh` (no prod data). Touched `guests` (+ `add_check_in_out_dates`), `invitation_emails`,
  `guest_emails`, `guest_sms`, `automations`, `bulk_prints`, `badge_print_logs`, `history_logs`;
  added casts to Guest/InvitationEmail/GuestEmail/BulkPrint/Automation/HistoryLog; printed-range
  query → `whereBetween` w/ Carbon; 3 `GuestsExport*` binders now format Carbon values; removed 4 dead
  unrouted debug methods from `OperationActionsController` (+ their commented routes).
- **FE + admin:** dropped the react-day-picker modal for cyan's **Cleave masked inputs**. Added
  `cleave.js` + `@types/cleave.js`, `components/shared/forms/masked-date-input.tsx` +
  `masked-time-input.tsx`, and shared `utils/date.ts` (`formatDate` / `formatDateTime`). Every
  `CustomDayInput` site → `MaskedDateInput` (`YYYY-MM-DD`); flight times → `MaskedTimeInput` (`HH:mm`,
  fixes the admin field that still emitted `hh:mm AM`); all active `timeZoneFix()` display → the new
  helpers. TZ conversion happens **at display** (`Asia/Riyadh`) so out-of-KSA users are correct.
  `masked-datetime-input.tsx` deleted (no user-entered datetime in alt). **Kept** `custom-day-input.tsx`
  (unused, for future) + therefore **kept** `timezoneFix.ts` (its only remaining consumer) — a
  deliberate revision of the "delete timezoneFix" plan.
- **Gates:** backend `pint --dirty --test` green, `migrate:fresh` clean, `php artisan test` 452 pass /
  3 fail (same pre-existing ExampleTest 403 + 2 avatar failures — unrelated). Admin + frontend
  `type-check` + `production` **green**. Mobile contract unaffected (guest dates only in Admin
  resources; offline QR `check_in_time` round-trips via the `datetime` cast; `routes/api.php`
  unchanged). No i18n keys moved.

**2026-07-06 (night) — Boolean DB cleanup + refactor, two tracks (ledger D6). Working-tree only on
`dev` across all three app repos — NOT yet committed/pushed (awaiting review). Full plan +
per-step log: `tasks/001-boolean-db-cleanup/TASK.md` + `upgrades/cleanup-hardening/BOOLEAN_REFACTOR_PLAN.md`.**
- **Track A** (mirrors cyan's documented refactor): pseudo-booleans (`yes`/`null`, `yes`/`no`,
  `with_`/`is_`) → real `boolean`s. Migrations edited in place (`string(...)->nullable()` →
  `boolean()->default(false)`) across guests flags, categories (`with_*` + notification fields),
  badges, email configs/templates, automation setups, countries, titles, invitations, guest
  logistics, gates, guest_sms; model `$casts` added; controllers/resources/blade `=== 'yes'` →
  `$request->boolean()` / `=== true`; admin `CustomSwitchInput` → `CustomSwitchInputBoolean` +
  boolean interfaces; frontend join-form radios (`is_saudi`, `require_*`, `valid_visa`) → booleans,
  SSR `=== 'yes'` boundary drop. **Intentional string keeps** (cyan-aligned): CSV `Exports` yes/no,
  input normalization, and the 3-state consent fields `display_photo_in_app` / `photo_consent` /
  `will_attend` / `terms`.
- **Track B** (net-new, beyond cyan): entity `status` (`active`/`blocked`) → `is_active boolean
  default(true)` on ~16 tables (speakers, sponsors, speaker/sponsor labels, zones, areas, gates,
  badges, categories, titles, admins, sms/email templates, invitations, guest_statuses, countries);
  `$casts`/`$fillable`; ~18 controllers `where('status','active')` → `where('is_active',true)` +
  `block()`/`activate()` setters (route names preserved); `Mobile{Speaker,Sponsor}Controller`
  filters; notification senders. Admin: shared `Status` badge → `isActive:boolean`,
  `status-types-select` → `true`/`false` options, all entity listings (`FilterFieldDef key:'is_active'`
  → serializes to `?is_active=`), 8 status forms, `bulk-update-badges-modal`, ~12 interfaces.
- **Excluded (multi-value / process statuses):** `app_notifications`, `login_attempts`,
  `email_attachments`, sms send-state, guest-workflow `status`/`guest_status_id`, mobile guest
  status, `users` (dead `UsersController`), automation `is_sent/is_delivered/is_open/is_clicked`.
  `meeting_rooms` + `smtp_configs` were already boolean `is_active` by design (no change needed).
- **Gates:** backend `pint --dirty` green, `migrate:fresh` clean, `php artisan test` **452 pass /
  3 fail** — all 3 are **pre-existing, unrelated** (confirmed by re-running on a stashed clean tree):
  `ExampleTest` GET `/` → 403, and two avatar tests that assume `display_photo_in_app` defaults to
  `'yes'` (it's nullable, intentionally left a string). Admin + frontend `type-check` + `production`
  **green**. **Mobile contract unchanged** — storage/internal only; no converted-entity `status`
  is exposed in any mobile resource.

**2026-07-06 (late pm, 2) — "Cheap cleanups" pass (admin + frontend), two separate commits each.
(1) Deleted dead files: 28 unreferenced `interfaces/*.tsx` in admin (`f6cffae`) + 2 stale duplicate
copies in frontend (`i18n/link copy.tsx`, `data/area-of-interset-generic-select_.tsx`, `864f7df`).
(2) Removed commented-out `// console.*` debug lines: 115 across 43 admin files (`f66ede8`) + 33 across
11 frontend files (`88ac243`). Pure hygiene, zero behaviour change; all four gates green. Both `dev`
branches pushed. The `zod`-removal and `@iconify/react`-dep items from that phase are moot (zod is
load-bearing; iconify already removed by the lucide migration).**

**2026-07-06 (late pm, 1) — Icon-library unification complete. Both Next apps now use a single icon
library, `lucide-react`; `@iconify/react` (P1) and `@heroicons/react` (P2) are fully removed from
source + `package.json` + `yarn.lock`. All four gates green (`type-check` + `production` on admin &
frontend). See ledger D5.**

**Earlier 2026-07-06 (pm):** Forgot-password unified onto the invite reset-by-token flow (admin +
backend). The whole tooling/hygiene batch + admin email-invite flow is now merged to `main` via PR #1 on
all three app repos. The backend has adopted a `dev` branch — it no longer commits straight to `main`
(see ledger D4).

**Earlier this session (am):** tooling + hygiene pass across the two Next apps (ESLint 9 flat config,
Prettier 3, zero lint warnings, husky + lint-staged, GTM removed) + dead-dependency / dead-code cleanup
+ docs cleanup (dropped the `-landing` app). `origin` is set on all repos
(`github.com/eissa-alt/alt-static-basecode-*`) and each branch tracks + matches its upstream.

## SHAs as of 2026-07-06 — HISTORICAL, do not read as current

> ⚠️ This snapshot is 102 backend commits stale (`4e1d532` was HEAD on 2026-07-06). It is kept as a
> record of that session, not as the current state — see the dated entries at the top of this file.
> Everything below this line is likewise historical: statements about gates, versions and outstanding
> work were true when written and have not been rewritten.

- `alt-backend` — `dev` = `main` @ `4e1d532` (PR #1 merge). Forgot-password backend at `a8184ca`; admin
  email-invite / `password_mode` **backend** flow at `04001b3` (P2.ST8). **Backend uses `dev`** and PRs
  into `main`, matching admin/frontend. Untouched by the icon migration.
- `alt-admin` — `dev` @ `f66ede8` (**cheap cleanups**, pushed): console-sweep `f66ede8` + dead-file
  delete `f6cffae`, on top of icon P2 `de87b4b` / P1 `d2565d9`. Before that `main`=`ed2e679` (PR #1
  merge), forgot-password repoint `70e646c`, tooling pass `29548e3`, admin-invite **UI** `d3ed5db` /
  `f43543f`. **`dev` is now well ahead of `main`** (icon migration + cleanups all `dev`-only).
- `alt-frontend` — `dev` @ `88ac243` (**cheap cleanups**, pushed): console-sweep `88ac243` + dead-file
  delete `864f7df`, on top of icon P2 `c74b82c` / P1 `3033a18`. Before that `main`=`f6b61e8` (PR #1
  merge), tooling pass `41ae698` + form-shape `default-` prefix rename + a `react-select` → Headless UI
  Listbox select refactor. **`dev` is now well ahead of `main`.**
- `docs` (on `main`) — this handoff refresh + ledger **D5** (previously `37176b7`: stale stack-version
  fix; earlier D3/D4, `-landing` drop / **D2**, husky hook).

## What landed recently

- **"Cheap cleanups" pass** (admin + frontend, hygiene, two commits each): **(1) dead-file deletion** —
  28 unreferenced `interfaces/*.tsx` in admin (`f6cffae`) + 2 stale duplicate copies in frontend
  (`864f7df`); **(2) commented-`console.*` sweep** — 115 lines / 43 files in admin (`f66ede8`) + 33
  lines / 11 files in frontend (`88ac243`). Full-line comment removals only (verified no inline/multiline
  cases); complements the earlier no-console rule (which handled live calls). Zero behaviour change, gates
  green. The other two items from that phase are moot: **`zod`** stays (load-bearing runtime peer dep of
  the email-template editor — removing it crashes the editor); **`@iconify/react`** was already dropped by
  the lucide migration.
- **Icon-library unification → `lucide-react`** (admin + frontend, ledger **D5**): dropped **both**
  baseline icon libs. **P1** removed `@iconify/react` (admin `d2565d9` — 146 conversions / 40 files +
  the event-day DB registry; frontend `3033a18` — 5 files incl. the `bi:tiktok` inline-SVG replacement).
  **P2** removed `@heroicons/react` (admin `de87b4b` — 82 files; frontend `c74b82c` — 13 files) and
  stripped it from `package.json` + `yarn.lock`. Machine-verified 1-for-1 name map (e.g.
  `ArrowTopRightOnSquareIcon`→`ExternalLink`, `PencilSquareIcon`→`SquarePen`, `PaperAirplaneIcon`→`Send`,
  `TrashIcon`→`Trash2`, `DocumentDuplicateIcon`→`Copy`); `className` sizing preserved throughout. Gates
  green both apps; **admin `production` build now run (was outstanding)**. Not mobile-facing.
- **Forgot-password → reset-by-token unification** (admin + backend): new
  `AuthController::forgotPassword()` reuses the `AdminInvite` token machinery + shared `admin_invite`
  blade (EN/AR) behind a public `POST /admin/forgot-password` route; the admin `forgot-password-form`
  now posts `{ email, back_link }` there instead of the legacy `v2/password/forgot` guest flow, so admins
  land on the same `reset-password/[token]` page as invites. **Enumeration-safe** (always returns success
  — a deliberate deviation from cyan). Cyan parity, see **LEDGER D3**. Mobile contract unaffected
  (admin-only, additive route).
- **ESLint 9 flat-config migration** (admin + frontend) + `@next/eslint-plugin-next`; **Prettier 2 → 3**;
  all lint warnings → **0** (autofix, optional catch binding + config `ignoreRestSiblings` /
  `argsIgnorePattern`, scoped `exhaustive-deps` / `<img>` disables each with a `-- reason`).
- **husky + lint-staged pre-commit hook** (admin + frontend): staged `*.{js,jsx,ts,tsx}` → `eslint --fix`,
  other types → `prettier --write`, re-staged. Installed via `prepare: husky`. Verified end-to-end.
- **GTM removed** from both apps (`react-gtm-module` + dead `utils/analytics.ts`).
- **Dead-dependency / dead-code cleanup**: removed `@svgr/webpack` (+ dead icon assets), `swiper`
  (frontend), `filepond-plugin-image-transform`, and many orphaned components/modals; admin swapped its
  legacy Portal modal for a shared `DialogShell`; frontend replaced `react-select` with a Headless UI
  Listbox `ui-select`. **This supersedes the KEEP verdicts in `upgrades/DEPENDENCY_AUDIT.md`** for
  `@svgr/webpack` / `swiper` (see the dated addendum there).
- **Editor settings**: Tailwind `suggestCanonicalClasses` silenced; deprecated `typescript.tsdk` /
  `enablePromptUseWorkspaceTsdk` → `js/ts.tsdk.path` / `js/ts.tsdk.promptToUseWorkspaceVersion`.
- **Docs**: dropped the 4th `-landing` app from all current-state docs (now a **three sub-app** baseline:
  `-backend`, `-admin`, `-frontend` + `docs/`); documented the pre-commit hook; ledger **D2**. Historical
  `upgrades/*` and D1's clone-source SHAs left intact (accurate record).
- **Admin email-invite / `password_mode` flow — merged + pushed** (backend `04001b3`, admin `d3ed5db` +
  `f43543f`): `AdminInvite` model + `admin_invites` / `password_mode` migrations, invite/resend/reset
  endpoints, SMTP-gated password-mode UI + reset-by-token page.

## Gates

- **Admin / Frontend:** `yarn type-check` + `yarn production` **green**; ESLint **0 warnings**. The
  pre-commit hook enforces Prettier/ESLint autofix on every commit.
- **Backend:** `pint --dirty --test` **green** on the forgot-password change. Run `php artisan test`
  before the next backend push. (Repo wasn't Pint-clean at baseline then — the advice at the time was
  `pint --dirty`. **Superseded:** ledger D10 made the repo Pint-clean and the gate is now the full
  `pint --test`; see CLAUDE.md hard rule 5.)
- **SMTP smoke test: DONE** — invite + reset-password email delivery verified against the active DB SMTP
  config (`DynamicSmtpService`).

## Next / outstanding

> Refreshed 2026-07-17. **Everything is pushed — all four repos are clean and in sync with `origin`.**
> Any "NOT pushed / not yet committed" wording in the dated entries above is point-in-time history, not
> current state.

- **Blocked — mobile ack:** backend `dev` → `main` is held until the mobile team acknowledges **both**:
  (1) the **D14** contract change (`avatar` is now a signed, 24h-expiring URL — mobile must re-fetch, not
  rebuild it; notice `docs/mobile/MOBILE_NOTICE_PRIVATE_AVATAR_SIGNED_URL.md`), and (2) the **D22** Task 007
  envelope deltas (every `mobile/*` payload now under `data`; `docs/mobile/RESPONSE_SHAPE_DELTAS.md`,
  IMPLEMENTED). User will bring the mobile repo into the parent folder and update the Flutter client directly.
- **Browser QA — visa upload (new):** `visa_copy`/`issued_visa` now persist end-to-end (D18) but have only
  been verified via tinker + signed-URL checks. Worth a real run: DB + local storage were wiped clean on
  07-16, so it's a clean slate.
- **`days` column removed (D20):** the phantom `guests.days` (no writer, read-only consumers, dead in the
  `122-gfeai-v2` clone too — superseded there by `forum_days`) was dropped fully across all three repos.
  Supersedes D18's ship-it call.
- **GTM decided (D21):** kept in **frontend** (`_app`/`_document` read `NEXT_PUBLIC_GTM`; the var ships in
  `.env.local` + `.env.example_prod`, empty = disabled until a clone sets `GTM-XXXX`). **Not in admin** —
  no code, no env var; nothing to remove.
- **Task 010 — api.php cleanup, reorg & RESTful rename (NEW, todo):** our `routes/api.php` is 966 lines of
  mostly-flat, ungated, non-RESTful legacy admin routes vs cyan's 559-line grouped + `admin.can:`-gated +
  `whereUuid` + RESTful shape. Tiered plan: cleanup → dead-code → reorg+**RESTful rename**+`whereUuid`
  (backend) → **cross-repo lockstep** (admin+frontend+mobile-if-touched) → RBAC gating. Scope now includes
  renaming as a **hard cutover** (reverses the initial "URIs frozen" call) and replacing `block`/`activate`
  with `toggle-status`. `admin.can`/`EnsureAdminPermission` confirmed present. `mobile/*` stays RESTful,
  only touched deliberately + documented. See `tasks/010-api-routes-cleanup/TASK.md`.
- **Browser QA** — forgot-password + invite create paths + reset-by-token page; plus the migrated
  listings + sidebar accordion (LTR/RTL) from the earlier P5.trim / cyan-parity session, which compiled
  green but were never browser-tested. **Add a visual pass on the migrated icons** (both apps) — the
  swaps compiled + built green but weren't eyeballed for glyph/size parity.
- **Merge `dev` → `main`** on admin + frontend when ready — the icon migration (P1+P2) **and** the cheap
  cleanups currently live only on `dev`; `main` is still at the PR #1 merge. (User asked to leave the PRs
  for now.)
- **`catch (X: any)` → `unknown`** — ✅ **DONE** (ledger D9, admin `5ceacc3` / frontend `8544c39`, on
  `dev`). Closed the "cheap cleanups" phase.
- **Phase 3 (later/opportunistic) — ✅ CLOSED 2026-07-19** (re-checked against code): `cont-list.ts`
  drift reconciled (now byte-identical admin↔frontend), `xlsx` now `await import()`, chart.js widgets all
  `next/dynamic({ ssr:false })` + `ChartCanvas` runtime-imports `chart.js/auto`, and `useFetch` adoption
  promoted to task 009 (closed). Only the *optional* drift-check script remains unbuilt. See
  `tasks/PHASE3_PARKED_TODO.md`.
- **Mobile team notice** — `docs/mobile/MOBILE_NOTICE_AGENDA_DATE_WALL_CLOCK.md` written; the mobile team
  still needs to be actually told + confirm receipt before the D8 change releases.

> Pint note (updated 2026-07-08, ledger **D10**): the backend is now **Pint-clean** (task 003 added
> `pint.json` + a repo-wide baseline, `96413df`). The gate is the **full `pint --test`** (`composer
> lint`), no longer the old `pint --dirty` workaround. Full backend gate = **`composer qa`** (`pint
> --test` + `phpstan analyse` + `php artisan test`). A `.githooks/pre-commit` hook also runs Pint on
> staged PHP (installed via `composer install`).
