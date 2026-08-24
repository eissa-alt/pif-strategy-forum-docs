# Fork port-back — findings and TODO

**Date:** 2026-08-06
**Scope:** what we should copy back into `alt-static-basecode` from three clone projects
**Status:** analysis done, nothing ported yet

---

## How we work through this list

> **One item at a time. No commits until the owner has reviewed the changes.**

The loop for every item on this list:

1. **Pick one item.** Not a batch, not "all the one-liners". One.
2. **Make the change.**
3. **Run the quality gate** for that app:
   - Backend: `composer qa` (Pint + PHPStan + tests)
   - Admin / frontend: `yarn type-check` + `yarn production`
4. **Show the actual diff.** Not a summary of it — the real `git diff`, so the owner reads
   the code that would enter history.
5. **Stop and wait.** The owner reviews.
6. **Only after an explicit "commit"** — commit it. Pushing is a separate approval again.

**Why one at a time:** these changes touch shared components and shared controllers. Several items
in this document look small but reach far — one edit to `ListingTable` touches 46 listings, one edit
to `ui-select` touches every dropdown in the admin. Batching them makes a bad change hard to isolate
and hard to reject cleanly. Reviewing one diff at a time means "no" costs nothing.

**Leaving the working tree dirty between steps is fine.** That is the review point, not a problem to
tidy up.

---

## 1. What this is

We cloned this baseline three times. Each clone found and fixed bugs. Those fixes stayed in the
clone. This document lists what is worth copying back, so the next clone starts from a better place.

The three clones:

| Project | What it is | Fork point | Its own commits |
|---|---|---|---|
| `122-gfeai-v2` | UNESCO Global Forum on Ethics of AI | 2026-07-08 | 220 backend / 173 admin / 100 frontend |
| `123-pif-pep-v2` | PIF Private Events Platform | 2026-07-28 | 10 / 11 / 4 / 2 seating |
| `124-ewc-2026-v2` | EWC 2026 | 2026-07-28 | 2 / 4 / 7 |

**Out of scope on purpose:** branding, colours, logos, fonts, SEO images, copy changes, and each
project's own form shapes. We only want fixes and features that any clone would benefit from.

---

## 2. How we checked

Three passes, 34 agents. Each pass fixed a mistake in the one before it.

1. **Pass 1 — commit by commit.** Read every commit in every fork. Then a second set of agents tried
   to *disprove* each finding by reading the baseline's own source. Two agents crashed mid-run and
   left gaps.
2. **Pass 2 — fill the gaps.** Read the 166 commits nobody had read. Also challenged the pep and ewc
   findings, which pass 1 had never checked.
3. **Pass 3 — file by file, not commit by commit.** Compared every shared component file directly
   between the baseline and each fork.

**Pass 3 mattered most.** Reading commits misses things, because a shared-component fix can hide
inside a commit whose title says "feat: rebrand". Comparing files directly found bugs that three
passes of commit-reading had walked past — including two that break the public registration form.

Every item below was checked against the baseline's actual source code, with a file and line number.
Nothing here is "the commit message said so".

---

## 3. What we found

**About 60 items worth porting.** Roughly a third are one-line fixes.

| Size of job | Count |
|---|---|
| One line | 17 |
| Small (one file, an hour or two) | 20 |
| Medium (a few files, half a day) | 8 |
| Large (needs planning) | 5 |

We also found **problems in the baseline that no fork fixes**. No amount of comparing forks would
have found these — they only turned up because agents audited the baseline directly. They are in
section 8.

### Three things to know first

**1. The invitation link dies after one use.**
A multi-use invitation for 50 people stops working after the first person registers. Worse: the
check is not scoped to the invitation, so if that email address ever registered anywhere else on the
site, the link never opens at all — not even once.

**2. The "complete your data" email link creates a duplicate guest.**
The guest clicks the link, fills the form, hits Submit. Instead of updating their record, it creates
a second one. This is a link we email to real registrants.

**3. One admin endpoint returns a 500 error on every single call.**
`GET /admin/guest-data-offline` filters on a database column that does not exist. The code's own
comment already says the column is missing. It has never worked.

---

## 4. Do these first — the quick wins

All one-line or near-one-line. All verified twice. None depend on each other. Together they close
most of the high-severity list.

> **✅ SECTION 4 IS CLOSED — 2026-08-08.** Nine items shipped, one closed as won't-do. Commits are
> on `dev` in each app, **not pushed**. Backend tests went 489 → 494. Two items' *descriptions* were
> wrong and are corrected in place below — the file/line references were sound both times, which is
> why "open the file before you edit" is the rule that saved them.

- [x] **Fix the invitation link.** Move the "already submitted" check so it only runs when no uses
      remain. `InvitationsController.php:467` runs before the `remaining === 0` gate at `:475`.
      Copy from **ewc `e647bb9`** — it is the same fix plus a 127-line test.
      **DONE — backend `117da56`.** Gated on `remaining < 1`, not `=== 0`, so a negative `remaining`
      (over-used row) can no longer read as a valid link. ewc's `InvitationLinkTest` taken verbatim
      (5 tests); confirmed failing without the controller change before it was kept.
- [x] **Fix the endpoint that always 500s.** `GuestsController.php:3615` does
      `Guest::where('dinner_invite', 'yes')`. No migration creates that column. Remove the filter.
      Same phantom column is also used as a listing filter at `:216-221`.
      **DONE — backend `7bf1d9c`.** Also added `with('scannedBy')`: the resource reads
      `scannedBy?->email` per row and this code had never executed, so switching it on without
      eager-loading would have introduced an N+1 over the whole guests table on its first success.
      `ExportInvitations` (PARKED, unrouted) and the null `dinner_invite` key in `GuestsResources`
      were left alone; the admin still forwards 3 dead `dinner_invite` params — fold into the
      dead-filter-keys item in section 5.
- [x] **Make gate check-in write `checked_in_at`.** `attend()` at `GuestsController.php:3585-3611`
      writes four other fields but not this one, so the Seating Manager and the admin both think
      nobody has checked in. From pep `976ae89`.
      **DONE — backend `880a7c0`.** Additive part only; pep's single-check-in 422 guard and
      `check_in_count = 1` were **not** taken (they change `attend()`'s response contract — still
      needs iPad-scanner sign-off). Verified the `Guest::booted()` bridge does not double-increment
      `check_in_count`: its guard requires `check_in`/`check_in_count` clean, and `attend()` dirties
      both.
- [x] **Fix the two upload paths.** `custom-file-input.tsx:212` posts to `/upload`,
      `custom-file-input-3.tsx:104` posts to `/upload-document`. Neither route exists under the
      admin API base. Real routes are `/admin/guests/upload` and `/admin/guests/upload-document`.
      Copy from pep `fd41aed` (fixes both) — gfeai only fixed one.
      **Also remove the fallback on the next line of each file.** It builds a public URL for a file
      stored on the private disk, so the preview breaks even after the path is fixed.
      **DONE — admin `9fb01fa`.** ⚠️ **The fix named above is wrong as written.** The admin axios
      base already ends in `/api/admin`, so `/admin/guests/upload` would resolve to
      `api/admin/admin/guests/upload`. The correct axios path drops the prefix: **`guests/upload`**
      (the P20 rule). Also: deleting the fallback is not enough on its own — `uploadAdmin` /
      `uploadDocumentAdmin` return only `data`, **never a `url` key**, so `response.data.url` is
      always undefined and the preview would vanish. Both now fall back to the local base64 already
      in hand, as pep does.
- [x] **Delete two test backdoors.** `GuestsController.php:3732` throws a 500 when the token is
      `IUFRP3GYZ4ISPUEU`. `InvitationsController.php:441` does the same for `RWYI82IWDG`. Both are on
      public, unauthenticated routes. Both marked `// TODO: remove this test hook`.
      **DONE — backend `afd4eee`.** Grepped all four repos: exactly these two, no third hook.
- [x] **Delete the `dd()` in committed code.** `GuestsController.php:2638`. Breaks our own rule 8.
      The method is not routed today, but it ships in every clone.
      **DONE — backend `e3591cc`.** Backend now has zero `dd()`/`dump()`/`var_dump()`. Note this
      quietens the method rather than fixing it: `ExportEBadgesFiltered` is unrouted with **zero
      callers in any repo**, builds a per-guest PDF into `$guestPdfOutput` it never uses, and returns
      `''`. **Open:** mark it PARKED (the convention `ExportInvitations` already uses) or delete it.
- [x] ~~**Un-comment the sidebar links.**~~ **NOT PORTED — owner decision, 2026-08-08.** See the
      row in section 10. ⚠️ **This item's description was wrong.** The commented block is not three
      links: it is the whole **"Mobile App Section"** — a section header plus **nine** modules
      (`event_days`, `conference`, `sessions`, `workshops`, `publications`, `media_center`,
      `notifications`, `meeting_rooms`, `speakers`). All nine are complete end-to-end — page,
      `inferFeatureId` rule, and a real action set in `AdminPermissions.php:69-80`. They were hidden
      deliberately in admin `1e449dc` (2026-06-24, a sidebar-restyle commit), and the inline comment
      records that the pages stay reachable by direct URL.
- [x] **Add a plain `build` script** to admin and frontend `package.json`. Today there are only
      `env-cmd`-wrapped variants, so neither app builds on Vercel or in CI. From ewc `557228b` /
      `28c79ee`.
      **DONE — admin `2db85bd`, frontend `38f42d8`.** ⚠️ **This repairs the quality gate itself.**
      Every build script is `env-cmd -f .env.<name>`, and every one of those files is gitignored —
      which is *why* the handoff has recorded "`yarn production` not run" for batch after batch. The
      runnable gate is now **`yarn type-check` + `yarn build`** (+ `yarn check:rbac` on admin);
      `yarn production` stays for local prod-config builds. Verified with no `.env` present: admin
      build exit 0 (92.3s), frontend exit 0 (52.7s). **`CLAUDE.md` rule 5 and
      `process/SETUP_AND_UPDATE.md` updated to match.**
- [x] **Add a null guard to reCAPTCHA.** `app/Rules/ReCaptcha.php:36` is
      `return $response->json()['success'];`. If Google returns a non-JSON body or the network
      fails, `json()` is null and this throws — turning a bot-check failure into a 500 on admin
      login *and* public registration. From gfeai `eafd3e4`.
      **DONE — backend `9631e17`.** ⚠️ **Correction:** Google's siteverify docs say the response is
      *always* JSON carrying `success`, errors included — so this is **not** Google's behaviour. The
      guard defends the transport (proxy, WAF, captive portal, an incident serving HTML). Verified
      by probe: HTML body / empty body / JSON-without-`success` each **500 before, `false` after**,
      while the two documented shapes are byte-identical either side — which is what makes it safe.
      **A connection failure still 500s** (`??` cannot catch a `ConnectionException` thrown before
      any response exists); left deliberately — access is already fail-closed, so it is an
      error-surface gap, not a security one. Only the guard was taken from `eafd3e4`; its
      environment-skip half remains a section 5 item.
- [x] **Raise the API rate limit.** `RouteServiceProvider.php:49` is still `perMinute(200)`. gfeai
      went to 300, then 500, under real load. Multi-step registration forms burst past 200.
      **DONE — backend `a19e8de`, 200 → 500.** `throttle:api` is on the whole `api` middleware group
      (`Kernel.php:71`), so this one number governs **all 476 routes**; the 30/min `public-api` /
      `sensitive-api` / `store-guest-api` limits stack on top and are unchanged, so the
      abuse-sensitive paths stay tight. Anonymous traffic is keyed by **IP**, so a venue full of
      guests behind one NAT shares a single budget — the main reason 200 was too tight. Honest note:
      gfeai's 200→300 (`40763f6`) carries a written rationale; its 300→500 (`c99aa58`) does **not**.
      "Under real load" above is not supported by that commit.

---

## 5. TODO — 122-gfeai-v2

### Broken things users hit

- [x] **Super Admins see no invitation collections and export empty files.**
      `InvitationsCollectionController.php:33/176/192` turn `null` into `[]`, then treat `[]` as "no
      access". A Super Admin has no category list, so they get nothing. There is no `is_super`
      bypass here, unlike `GuestsController::applyAdminGuestAccessFilter:368`.
      **The fork's fix is incomplete.** The same bug is in `InvitationsController::index`,
      `::getAllInvitations` and `::export`. Fix all four. From gfeai `7ce4225`.
      **DONE — backend `6f37068`.** ⚠️ **It is 7 sites in 3 controllers, not 4 in 2.**
      `DashboardStatsController::applyAdminGuestAccessFilter` is not mentioned above at all — it is
      a near-copy of the Guests one with the `is_super` early return missing, so a Super Admin also
      saw a **dashboard of zeroes**. ⚠️ **The exports needed a different fix from the listings:** the
      Export classes distinguish `null` (no filter) from `[]` (deny all) —
      `InvitationsCollectionsExport:30/37` — and the controllers were flattening a Super Admin's
      `null` into `[]`, which the export then read as deny-all. Leaving it `null` is the fix; adding
      a `whereIn` would not have been.

- [x] **The guests listing loses 9 filters when you turn the page.**
      Filter the list, click page 2, and page 2 comes back unfiltered while the filter chips still
      show. `guests-listing.tsx:247-292` rebuilds the URL from a hand-written list of 36 keys, but
      the search form pushes 43. The fix is one line — `searchQueries` on line 245 already holds
      the whole query. From gfeai `a4dab7a`.
      **Same file, related:** `print-logs.tsx` has the same pattern.
      **DONE — admin `ec9ab15`.** The nine were derived rather than taken on trust, by diffing
      paginate's keys against what the search form pushes: `accommodation_requests`,
      `admin_check_in_date`, `admin_check_out_date`, `flight_comments`, `reconfirmed_will_attend`,
      `require_accommodation`, `require_flights`, `self_booking`, `transportation_comments`. **All
      nine are logistics/accommodation filters** — the Task 019 set, added long after this key list
      was written, which is the argument for spreading rather than adding nine more lines.
      `lang`/`slug`/`id` are destructured out (route segments, not filters).
      **`print-logs.tsx` needed no fix** — it takes exactly `{mode,page,query,status}`, so it
      already carries every filter it has. Same fragile shape, zero data loss today.

- [x] **The "Reconfirmation" filter does nothing at all.**
      Pick Yes or No, hit Search, the URL changes, the results do not.
      `search-guests-by-super.tsx:216` pushes `reconfirmed_will_attend` into the URL, but
      `fetch-data-url.ts:83-126` has no parameter for it, so it never reaches the API.
      **9 keys total** are pushed but never forwarded. From gfeai `9ea56fd`.
      **DONE — admin `9f10242`.** ⚠️ **Only ONE of the nine is a user-facing bug.** Each was checked
      against both ends:

      | key | control rendered? | backend filter? |
      |---|---|---|
      | `reconfirmed_will_attend` | **yes** | **yes** (`applyFilters:143`, `pending`/`yes`/`no`) |
      | `require_accommodation` / `require_flights` | no | yes |
      | the other six | no | no |

      Only `reconfirmed_will_attend` has a control a user can actually operate, so only it was
      wired. The other eight exist solely as a type, a `defaultValues` mapping and a submit-payload
      key — nothing renders them, so they are unreachable except by hand-editing the URL. Same
      phantom-plumbing pattern as `dinner_invite`. **Left alone deliberately → see
      `FORK_PORT_BACK_LEFTOVERS.md`:** deleting dead plumbing vs adding six backend filters is a
      decision, not a port.

- [x] **Every email log row says "No".**
      Sent, Delivered, Opened, Clicked all show a grey "No" badge forever — including for messages
      that were definitely delivered. The backend now sends real booleans; the admin still compares
      against the string `'yes'`. **4 files, 14 comparison sites.** The correct helper already
      exists at `guest-sms-logs-listing.tsx:26`. From gfeai `7eff9cc` + `af620c0`.
      **DONE — admin `6cc17d5`.** ⚠️ **It is 5 files, not 4** — the two see-more modals
      (`see-more-admin.tsx` → `guest_emails`, `table-see-more-invitation-modal.tsx` →
      `invitation_emails`) were not listed. Four are fixed. **The fifth must NOT be:**
      `automation-details.tsx` renders the `Automation` model, whose `is_sent`/`is_delivered`/
      `is_open` are **string** columns (migration `2024_05_17_155941`) with no boolean cast, and the
      backend filters them with `where('is_sent','yes')`. Its `=== 'yes'` is correct — changing it
      would be a regression. Filter option values (`'yes'`/`'no'`) are also correct and untouched;
      they match the SMS listing and the backend has accepted them since P23.11.

- [x] **Three columns are permanently blank.**
      React renders `true` and `false` as nothing, so you cannot tell the two apart. In the
      invitations listing (`:252` `render: row => row.is_sent`) and two badge/verified chips.
      Found by checking all 84 boolean columns against both apps.
      **DONE — admin `67710b3`.** All three confirmed boolean-cast on their models: invitations
      "Sent" (`Invitation`), badges "Show background" (`Badge`), and the verified chip in
      `see-more-admin.tsx` (`Guest`) which rendered its icon with no label. Now an explicit Yes/No
      badge, a Yes/No label, and the existing `web:verified` string — all three keys already existed
      in EN + AR, so no translation change was needed. `guests-listing.tsx:707` already did this
      correctly (`row.verified === true && <BadgeCheck/>`) and is untouched.

- [x] **Every dropdown in the admin freezes the page.**
      Open any select, action menu, export menu or filter, and the page will not scroll — and
      clicks outside the panel are swallowed, because the rest of the page goes inert.
      Headless UI 2.2.10 defaults `modal` to `true`; we never pass `modal={false}`.
      **14 sites across 13 files.** `ui-select.tsx` is the main one —
      `custom-select.tsx` is just a re-export of it, and 39 admin files import that.
      **Two fixes must land together:** `anchor="bottom start"` (gfeai `cd0a861`) stops the panel
      being clipped, `modal={false}` (gfeai `60c0e26`) stops the freeze. Porting `anchor` alone
      would likely *introduce* the freeze.
      **DONE — admin `4151422`.** 14/14 panels across 13 files, coverage checked by count.
      ⚠️ **The "must land together" claim above is wrong**, verified by reading
      `@headlessui/react@2.2.10` rather than the commit summaries. In `listbox.js`:
      `{ anchor:s, portal:a=!1, modal:_=!0 }` — `modal` defaults true **independently of `anchor`**;
      `o && (a=!0)` shows `anchor` merely forces `portal`; and the freeze is
      `useScrollLock(modal && open)` + `useInertOthers(modal && open)`. So `modal={false}` alone
      fixes it, and `anchor` neither causes nor prevents it. **`anchor` was deliberately NOT taken:**
      every one of our panels positions manually with `absolute` classes, so switching to portal
      positioning risks 14 layout regressions for no bug-fix value. The clipping `anchor` addresses
      is real but separate and needs visual QA.

- [x] **The public registration form has the same freeze.**
      No fork ever fixed this side. Three components —
      `countries-select-new.tsx:224`, `titles-select-new.tsx:165`, and the static select — back
      every dropdown a registrant touches. On a phone, tapping Country freezes the page.
      **DONE — frontend `2d66436`.** All three (`countries-select-new` uses `ComboboxOptions`,
      confirmed to carry the same `modal:b=!0` default as `ListboxOptions`). `modal={false}` only,
      same reasoning as the admin side.

- [x] **Invitations form hides the real error.**
      A duplicate collection name comes back as a clear 422 message, but `invitations-form.tsx:469`
      logs it to the console and shows a generic "something went wrong" toast. Next 15's dev overlay
      then reports it as a crash. From gfeai `0be14a0`.
      **DONE — admin `864db95`.** The 422 branch already set the per-field errors correctly; it just
      fired the generic `res:500` toast afterwards regardless. The API returns its messages under
      `data` keyed by field with no top-level `message`, so the toast now shows the first field
      message. Removing the `console.error` also cleared the **last** `console.*` call in the admin
      app — `components/`, `pages/` and `utils/` are now clean under hard rule 8.

- [x] **Invitations can be created for people who already registered.**
      These can never be redeemed — the register endpoint rejects duplicate emails and
      `guests.email` is unique. Needs the check server-side *and* in both admin fill modes (Excel
      and manual). From gfeai `4d4fed0` + `ec316cc`.
      **DONE (server side) — backend `faac288`.** Premise verified: `guests.email` is `unique` in
      the migration, and register refuses duplicates at `GuestsController:547`. `store()` now
      refuses the batch **before `DB::beginTransaction()`**, so a refused batch mints nothing (the
      test asserts `Invitation::count() === 0`, not just the 422). Both sides are lowered/trimmed —
      the test submits `TAKEN@example.com` against a stored `taken@example.com`, since the naive
      version would let that through and create exactly the dead link this prevents.
      ⚠️ **The admin half is deliberately NOT done.** The server gate holds for every caller and is
      where correctness lives; the admin pre-check is UX — warn before submitting 200 rows rather
      than after — and is a separate change across two fill modes (Excel and manual). Not bundled
      in silently. See LEFTOVERS.

- [x] **Invitation create writes NULL into three NOT NULL columns.**
      `prefilldata`, `lock_data`, `with_from`. Same bug we already fixed for `is_sent` in
      `1f3839d` — it missed these three. From gfeai `4d4fed0`.
      **DONE — backend `5414ca3`.** ⚠️ **Worse than this item implies:** it reads as a
      data-integrity nit, but it is a **hard 500 on invitation creation**. The admin form only
      sends those keys when someone flips the switch, so the *default* path inserted three nulls
      and the create failed outright. Verified by reverting: `received 500`. Fixed with
      `$request->boolean()` (the idiom P029 already used for the title switches), which also
      handles the `'true'`/`'1'`/`'on'` string forms `?? null` did not. Adds **first coverage for
      `POST /admin/invitations`** — itself a listed gap — including a case with an explicit
      `false`, which a naive `?? false` would pass while still being wrong.

- [x] **Saving a title with Arabic gender names throws a 500.**
      `normalizeGenderSpecific` returns an array, the column is untyped `text`, and
      `TitlesResources` reads it back with `json_decode`. Needs `json_encode()` in **both**
      `TitlesController.php:88` (store) and `:149` (update).
      *Note: we previously believed P029.1 fixed this. It only fixed half.*
      Also add `?? 0` on `order` in update, and the 62-line test — we have **zero** title tests.
      **DONE — backend `f1609b4`.** Root cause captured: `ErrorException: Array to string
      conversion` in `Connection.php`.
      ⚠️ **This item is wrong about "both".** `update()` was **never broken** — assigning to a
      model attribute and calling `save()` lets Eloquent encode the array on the way out, while
      `Model::create()` → insert does not. Measured, not assumed (`save() => OK`, column holds
      valid JSON). Found only because the update test kept passing unfixed. Encoding now lives in
      the helper so both paths are explicit rather than one silently depending on that framework
      behaviour. **The `order` half is a real second bug** and is fixed: `order` is a NOT NULL
      integer with no default, and `has()` is true for a present-but-null key.
      **Four tests added** (titles had zero, which is how P029.1 shipped as a half-fix): store and
      update with Arabic names, the resource round-trip, and `order: null`. Unfixed, three of four
      fail — the passing one is the update-encoding case, consistent with the correction above.

- [ ] **Export column letters point at the wrong columns.**
      Not "fragile" — already wrong. `GuestsExport.php:36-39` has `DATE_COLS = ['S','W']` labelled
      "Printed At / Created At", but S is *Source* and W is *Reconfirmed Will Attend*. The real date
      columns are M, Q, X. Same in `GuestsExportGuestView.php`, `GuestsExportView.php`, and
      `InvitationsCollectionsWithInvitationsExport.php` (where `COL_TITLE_FROM='Z'` runs the title
      converter over *genders*, blanking them).
      **28 classes in `app/Exports/` all use hardcoded letters. None use a lookup.**
      **Port the mechanism, not corrected letters** — a headings-based lookup (gfeai `5c39fd8` /
      `7ce4225`), plus the alignment test (gfeai `741ad17`). We have **zero** export tests today.

      **🔎 AUDIT — 2026-08-12, corrected 2026-08-12.** ⚠️ **My first audit here was wrong twice**
      and is replaced below. Both errors came from one cause: a regex that read `headings()`
      *including commented-out entries*. It claimed a 36-vs-33 headings/map mismatch (there is
      **none** — 34 vs 34) and claimed column `Z` is "Tier Name" (it is **`Gender (From)`**). **The
      original item text above was right: the title converter did run over genders.**

      **Root cause:** `'Tier Name'` was commented out of `headings()` and its map value removed,
      shifting every guest-field column **left by one**. Two constants never moved with it.

      | Class | Constant | Was | Landed on | Now |
      |---|---|---|---|---|
      | `InvitationsExport` + `…CollectionsWithInvitations…` | `COL_TITLE_FROM` | `Z` | `Gender (From)` → `Title::findOrFail('male')` → **uncaught, whole export 500s** | `AA` |
      | both | `COL_PHONE_FROM` | `AF` | `Email (From)` → phone text-formatting | `AG` |
      | `GuestsExport` | `DATE_COLS` | `S`,`W` | Source · Reconfirmed Will Attend | *unchanged — see below* |
      | `GuestDraftsExport` | all four | — | correct | ✅ |

      Layout in both invitation classes: `A–Y` base row (25), `Z–AG` guest fields (8), `AH` Created
      At. **Only 5 of the 28 export classes use column letters at all** — the other 23 bind by row
      number and are unaffected.

      **DONE (the crash) — backend `fa4f4a8`.** Both constants corrected in both classes, plus
      `ExportColumnAlignmentTest`: 12 cases resolving every constant against its own `headings()`.
      Verified load-bearing — with the old values it reports
      `COL_TITLE_FROM = 'Z' points at 'Gender (From)', not 'Title (From)'`.

      **STILL OPEN — two follow-ups:**
      1. **`GuestsExport` / `GuestsExportGuestView` `DATE_COLS`.** Cosmetic: headings and map are
         aligned (25/25), the letters are simply wrong — date conversion lands on Source and
         Reconfirmed Will Attend while the three real date columns (**M, Q, X** — note `X` is
         missing from the constant entirely) get none.
      2. **The headings-lookup mechanism** (gfeai `5c39fd8` / `7ce4225`). Worth noting *why*: every
         `COL_TITLE` / `COL_EMAIL` / `COL_PHONE` constant is currently correct **by luck** — right
         only because nothing has been inserted before them yet. The alignment test now catches
         drift, which removes most of the urgency.

- [x] **"Send test" is clickable while creating a template** and posts to
      `/email-templates/send-test/undefined`. From gfeai `b194523`.
      **DONE — admin `e3aa1de`.** Now edit-only; there is nothing to test against until the template
      exists. Its label was also a hardcoded `'Send test'` literal that would never render in
      Arabic — now goes through `useTranslate`, reusing the existing `web:send_test_email` key, so no
      translation files changed.

- [x] **Date fields say "required" when they clearly contain a date.**
      Type a real date that falls outside min/max and `masked-date-input.tsx:124-134` reports it as
      missing instead of out of range. 8 files, ~14 call sites. From gfeai `daee938`.
      **DONE — admin `ece978e`.** A new `rangeError` prop wins over `error` while the box holds a
      **complete** date that min/max rejects. That completeness check is the subtle part: half-typed
      input is unfinished, not out of range, so without it the message would flash on every
      keystroke. Committing `null` is kept, so a bad value still cannot be submitted — only the
      message changed. EN + AR `validation:date_out_of_range` added.
      ⚠️ **"8 files, ~14 call sites" is wrong.** There are 15 `MaskedDateInput` sites but only
      **two** pass `minDate`/`maxDate`, and a bounds error is impossible without bounds —
      `automation-settings-fields.tsx:599` (scheduling in the past) and
      `fours-steps/step-2.tsx:502` (birth date in the future). The other 13 needed no change.

- [x] **Download errors always show the generic 500 toast.**
      On a `responseType: 'blob'` request the real message ("no default SMTP config") is inside the
      Blob and never read. **7 call sites.**
      **DONE — admin `5c31268`.** New `readApiErrorBody()` / `readApiErrorMessage()` in
      `utils/api-error.ts`, wired into all seven sites.
      **Three things this item does not mention, all found while doing it:**
      1. **It hid 422 validation errors too.** Three sites read `apiError?.data?.data` to fill
         per-field form errors — equally undefined on a Blob, so `responseErrors` was always `{}`
         and field errors never reached the form. Hence the helper returns the parsed *body*, not
         just a string.
      2. **Two modals showed nothing at all.** `export-modal` and `custom-export-modal` had
         `catch {}` with the entire body commented out, so a failed export looked like a successful
         no-op — worse than a wrong message.
      3. **An error path that is not an exception.** `e-visa-listing.downloadGenerated` checks for
         `content-type: application/json` on a **200** and toasted a bare 500, while the API puts
         the reason in that body.
      ⚠️ `e-visa-listing` has two near-identical catch blocks; the first (`:315`) is **not** the blob
      one. Easy to patch the wrong one — I did, briefly.

- [x] **Social-media modal sends bad data.** `order` goes as a string (backend wants an integer,
      422), `link` has no URL validation, and every platform tile draws a check mark, not just the
      selected one. From gfeai `b194523`.
      **DONE — admin `5abb472`.** ⚠️ **`order` does NOT 422** as claimed: Laravel's `integer` rule
      uses `filter_var(FILTER_VALIDATE_INT)`, and `'3'` passes — tested. Fixed anyway because the
      form type declared `order: number | null` while `register()` on a number input stored a
      **string**, and `3.5` *does* 422. The link now has a client-side URL pattern (EN + AR
      `validation:url`), and the tick renders only on the selected tile — it used to draw on every
      tile and rely on colour alone. The 8 broken social icons in the same file are the parked
      LEFTOVERS 2.6 bucket decision, deliberately untouched.

- [ ] **Admin CSP blocks 8 social icons.** `soical-media-modal.tsx:43+` hardcodes them on
      **another ALT client's bucket** (`emails-alt.eu-central-1.linodeobjects.com`). It is never
      derived from env, so the CSP at `next.config.js:54` cannot cover it and all 8 render broken.
      **Do not copy gfeai's fix** — it widens the CSP to a wildcard bucket host, which undoes our
      env-derived design. Move the icons to the project's own storage instead.

### Missing features

- [ ] **Add an `is_sent` filter** to the invitations listing (backend + admin). gfeai `12f6a62` /
      `7effa14`.
- [ ] **E-visa bulk download** — selected / filtered / all pages / in-progress only, as one zip.
      Today "Download all" fires one browser download per guest and browsers block the burst.
      Paired backend + admin job. gfeai `01582a2`, `2f917ec`, `48b8578`, `528c977`, `b6b84b9`.
- [ ] **Clone an email template** (with its attachments). We already clone categories, SMTP configs
      and badges — templates are the gap. Paired backend + admin. gfeai `1a423f5` / `3f97f70`.
- [ ] **Bulk missing-data notify** — `POST /admin/guests/notify-bulk` with a sent/skipped tally.
      Hidden inside a commit titled "feat: rebrand". gfeai `4c6187e`.
- [x] **Harden base64 uploads.** `UploadService.php:61` takes the file extension from whatever mime
      the caller declares, with no allow-list — `data:x/phtml;base64,…` writes a `.phtml` file to
      the **public** disk. There is also no size limit. 4 call sites are affected.
      **DONE — backend `113c9fd`.** The service validated the *bytes* but took the extension from
      the caller-declared mime — different inputs, so real PNG bytes sent as `data:x/phtml;base64,…`
      passed the sniff and landed as `.phtml`. Now: honour the declared extension only when
      allow-listed (`jpg`/`png` — all the admin components ever send), else fall back to sniffing,
      else **refuse** rather than guess `jpg`. Plus a 10 MB ceiling.
      ⚠️ **`gif`/`webp`/`svg` had to come out of BOTH the allow-list and the sniff fallback** —
      stripping only the allow-list left them reachable through the fallback, and every test stayed
      green, so it looked done and was not. Two code paths decide the extension.
      **Also fixed a real bug nobody reported:** SVG was stored as `.svg+xml` (everything after the
      slash in `image/svg+xml`), an extension no server serves as an image — so SVG upload never
      worked. A test was *asserting* that broken output rather than catching it.
      **Four null guards added alongside**, because refusals became far more likely and Badges /
      Sponsors / Speakers / Categories returned `status: success` with a **null** filename and a URL
      ending in `/`, so the admin stored the null. They now 422, matching `GuestsController`. All 9
      call sites across 6 controllers guarded. Path traversal on the caller-controlled `$dir` was
      tested and is blocked by Flysystem. 5 new regression tests; 498 pass.
      **Still open (pre-existing, logged in LEFTOVERS §3):** those four endpoints have **no request
      validation at all**, and `$dir` is built from unsanitised `moduleName`/`inputName`.
- [ ] **Skip reCAPTCHA outside production** (opt back in with `RECAPTCHA_ENFORCE`). Local dev
      currently cannot log in or register. gfeai `eafd3e4`.
      ⚠️ That commit also reports that the backend's Google secret is Google's **test** secret and
      does not match the site key the apps use. We could not verify this (we do not read `.env`) —
      **please check before enforcing reCAPTCHA in production.**
- [ ] **Multi-file upload input** with in-browser image downscaling. gfeai `df4c152`.
- [ ] **Searchable guest-type picker** instead of an unbounded card list. gfeai `c127e36`.
- [ ] **Per-option help text on radio groups** — `custom-radio-input.tsx` can only show one shared
      label and only lays out horizontally. gfeai `03d2411`.
- [ ] **Scroll hint on listing tables.** Every listing is a fixed 1120px table; on a laptop the
      Actions column (Edit/Delete) is off-screen with nothing indicating it. **46 components import
      `ListingTable`** — one edit fixes all of them. gfeai `cd0a861`.
- [ ] **Show consumed uses / limit** on the invitations listing instead of a raw send count.
      gfeai `5520a33`.
- [ ] **Bulk guest hard-delete by email list**, with preview and a workbook export. Operator tool.
      gfeai `e1273d4`, `c4dfcbc`, `382b13d`.
- [ ] **Tests we do not have:** first coverage for the invitation store endpoint and the guest
      listing filters. gfeai `4d4fed0`, `6622fcc`.

### Public registration form (frontend)

- [x] **The complete-data link creates a duplicate guest.**
      `complete-data/[category]/[token].tsx:57` passes `onCustomSubmit`, and
      `renderFormSteps.tsx` forwards it at `:155` and `:217` — but only
      `one-step-rsvp/step-1.tsx` actually uses it. `one-step/step-1.tsx:204` and
      `fours-steps/step-4.tsx:90` just POST to `guests`. **2 of our 3 form shapes.**
      From gfeai `6b15463`.
      **DONE — frontend `1d542c4`.** Both now branch on `onCustomSubmit` before the create. Passing
      raw `getValues()` was checked rather than assumed — `fours-steps` strips `*_url` and
      Saudi-conditional fields before its own POST, but the backend's `updateGuestMissingData` uses
      `$request->only([...])` with a **12-field allow-list**, so extra keys are dropped server-side.
      gfeai's commit fixes `default/one-step` plus its own project shapes; it does not carry
      `fours-steps`, so that half is ours.
      ⚠️ **Pre-existing, NOT fixed:** that 12-field allow-list means complete-data only ever updates
      `gender, title_id, first_name, last_name, company, job_title, email, phone, personal_image,
      will_attend, dietary_requirements`. A guest completing the **four-step** form via a token link
      fills in visa/accommodation/flight fields that are then **silently discarded** — previously
      lost to a duplicate record, now lost to the allow-list. See `FORK_PORT_BACK_LEFTOVERS.md`.

- [x] **Prefilled forms reject the guest's own email.**
      `isUniqueAttribute(...)` is called with the mode hardcoded to `'create'`, so the check matches
      the guest's own row and blocks submit. The guest id is already being passed — only the mode is
      wrong. 3 call sites. From gfeai `3499a55`.
      **DONE — frontend `15caa95`.** Chain verified end to end:
      `GuestsCompleteMissingDataResources` exposes `id` → the page passes `defaultFormValues={guest}`
      → `getValues('id')` is populated. `checkUnique` forgives a row against itself only when
      **`mode === 'edit'` AND the id matches** — the literal is `'edit'`, not `'update'`. Mode is now
      derived from whether an id exists, so a fresh registration still sends `'create'`.
      **This item's description was accurate** — the first one this session that needed no
      correction.
      ⚠️ **But "3 call sites" is wrong: only TWO should change.** The third is the **plus-one** email
      validator, where `getValues('id')` is the *primary* guest's row — passing `'edit'` would ask
      the backend to forgive a collision with the primary, letting a plus-one reuse the main
      registrant's address. A plus-one has no own row to forgive, so it stays `'create'` with a
      comment saying why. A local check above it already blocked that case, so it was never
      exploitable; the change would simply have been wrong.

- [x] **The completion page marks every field required**, because it passes no optional/mandatory
      lists. From gfeai `3499a55`.
      ⚠️ **Half wrong, and the half it misses is worse.** The two lists work in OPPOSITE directions,
      so passing neither breaks the form both ways at once:
      `mandatoryFields?.includes(x) ?? false` → missing means **optional**;
      `!optionalFields || !optionalFields.includes(x)` → missing means **required**.
      On the **one-step shape** (21 `mandatoryFields` uses) the completion form asked for
      **nothing** — a guest could submit it blank, which is the opposite of the reported bug and
      arguably worse, since collecting missing data is the entire point of the link. Only
      four-steps step-2 matched the description.
      **DONE — backend `0dab0d1`, frontend `b0d33cf`.** `verifyStatusAndToken` returned only
      `status`, `guest`, `form_shape` — **no field rules at all** — so the form rendered with
      nothing configured. It now returns `mandatory_fields`, `extra_guest_mandatory_fields`,
      `with_extra_guests` and `number_of_extra_guests`, mirroring `verifyInvitation`; the page
      threads them into the same `JoinFormWrapper`. Additive only, and the endpoint is absent from
      `docs/mobile/` (hard rule 4 checked).
      ⚠️ **The real gap was 4 props, not the 9 first identified.** `guestEmail`/`FirstName`/
      `LastName` are already covered by `defaultFormValues`, and `residencyStatusTypes` is **not**
      needed — step-2 falls back to the full `ResidencyStatusTypesSelect` when it is empty, so that
      dropdown was never broken. See the new leftovers entry: `visa_status_types` is never returned
      by the API at all, so the **join** page passes `null` too.

- [x] **`optionalFields` is dead input and inverts the required/optional default.** *(Raised while
      doing the item above, 2026-08-24.)*
      **DONE — frontend `e0d049f`, admin `f91fcc5`, backend `c6bf5d2`.**
      `fours-steps/step-2` gated 14 ID/visa fields on the category's `optional_fields` — required
      *unless* listed — while every other step uses `mandatory_fields`, required *when* listed.
      `optional_fields` is a column **the admin form has never written**, so it was permanently
      null: all 14 fields were unconditionally required for every category, unchangeable. The list
      existed only to be empty.
      All 29 logic sites migrated to `mandatoryFields`, then the whole prop chain deleted
      (`renderFormSteps`, `form-wrapper`, three step-1 declarations, both join pages) — **zero
      references remain**. Admin gained `full_name_on_document`, the one field of the 14 its picker
      did not already offer. A data migration backfills the 14 names into existing
      `default-four-steps` categories, verified against a real DB (de-dupes, leaves other shapes
      alone, `down()` reverses).
      ⚠️ **DEPLOY ORDER: run the migration BEFORE shipping the frontend**, or all 14 ID/visa fields
      silently become optional. Not yet run on the dev DB.
      ⚠️ **A claim I made mid-investigation was wrong:** I reported the admin picker offered none of
      these fields and that a category save would wipe the backfill. I had read
      `data/mandatory-fields-select.tsx`, which is marked `@deprecated`; the picker actually uses
      `getMandatoryFieldsByFormShape()`, which already had 13 of 14. No such danger existed.

- [x] **The country dropdown silently wipes a saved country** that is no longer in the list — on
      the prefilled complete-data form, which is exactly where it hurts.
      **NOT REPRODUCIBLE — closed 2026-08-24, no code change.** Every condition that could fire the
      clearing effect was checked and none can:
      1. **Not on the complete-data page at all.** `GuestsCompleteMissingDataResources` does not
         return `nationality` or `country_of_residence`, so the select receives
         `selected_id={undefined}` — and the effect is guarded on `selected_id &&
         !selectedOption && !fetchLoading && countries.length > 0`. It short-circuits on the exact
         page the item calls out.
      2. **The list is never filtered.** `CountryController::selectList()` is a bare
         `Country::orderBy('name_en')->get()` — no `is_active`, no scope — so every country is
         always returned and any stored id always matches.
      3. **Countries cannot be deleted.** The admin routes are `index`, `store`, `show`, `update`;
         there is **no destroy route**. A country cannot vanish from the list.
      The effect is defensive code that, on this codebase, never runs.
      ⚠️ **It did surface a real gap, logged separately:** that resource returns only **13 fields**,
      so on a four-step completion form nationality, residency, document, visa and birth fields all
      render **empty** despite being stored — the "prefilled" form is not prefilled past step 1.
      That is the mirror of LEFTOVERS 2.5, where the *submit* path drops the same fields via a
      12-field allow-list. Both ends of the flow are narrowed to the same subset; the decision lives
      in 2.5.

- [ ] **Two-step form shapes strand the user.** `renderFormSteps` step-2 branch is not
      last-step aware, so the guest submits, the record is created, and nothing happens next.
      **Fixed independently in both gfeai and ewc** — two forks hitting it is a strong signal.

- [ ] **Admin-controllable privacy consent** — today the consent checkbox is hardcoded per shape,
      one shape has none, and one records consent silently. gfeai `bfebdfc` / `0bef709`.

### Big jobs — decide before starting

- [ ] **DB-driven mandatory-field catalogue (`form_shape_fields`).**
      Today the field lists are hardcoded in the admin and frontend bundles. gfeai moves them into a
      table, serves them over the API, and enforces them server-side.
      **This sits right on the "no forms business logic" line.** The catalogue and the enforcement
      are architecture; the per-shape field lists are project data. Worth its own task.
- [ ] **Split the completion flow into purpose-scoped tokens (`completion_tokens`).**
      Replaces one-row-per-email token tables. Only relevant if we keep logistics.
      ⚠️ Related bug found on the way: sending a test email **silently invalidates the completion
      link already sitting in a guest's inbox**.

---

## 6. TODO — 123-pif-pep-v2

- [x] **Fix the always-500 offline scanner endpoint** and the dead dinner filter — see section 4.
- [x] **Write `checked_in_at` on gate check-in** — see section 4.
- [ ] **Fix the seating chair overlap.** The seating app draws chairs on top of each other at every
      table corner — 0px apart on U-shape tables, 55.1px apart for a 56px chair on rectangles once
      `headSeats >= 2`. pep `8c39930`.
- [ ] **`GuestSeatingObserver`** — sends seat-assigned, seat-changed and check-in notifications
      across all three channels, with bulk-send suppression and previous-seat capture in the same
      UPDATE. Needs a `guests.previous_seat` column. pep `c941802`. *(Large.)*
- [ ] **Per-admin filter visibility.** One button on the filter row opens a picker; each admin hides
      the filters they never use. Stored on `admins.preferences`, keyed by route, so it follows them
      to another device. Saves *hidden* keys, so filters added later stay visible by default.
      **`ListingFilters` is used by 44 listings**, so they all get it with no call-site changes.
      The guests listing needed extra work — `FilterSlot` self-registers so its 24 hand-written
      filters appear in the picker too. pep `fd41aed`. *(Large.)*
- [ ] **`admins.preferences` column + `GET/PUT /admin/me/preferences`.**
      ⚠️ **pep's routes have no `auth.admin` middleware**, so a mobile guest token can reach an
      admin write. Add it when porting.
- [x] **Fix the offline-sync foreign key bug.** The round-trip writes an email string back into
      `scanned_by`, which is a UUID foreign key. pep touches this method without fixing it.
      **DONE — backend `dc6eeb9`.** ⚠️ **Not a data-integrity nit — a hard 500.** Root cause
      captured: `SQLSTATE[23000]: Integrity constraint violation: FOREIGN KEY constraint failed —
      update "guests" set … "scanned_by" = gate@example.com`. A first-ever scan passes because
      `scannedBy` is null; **every subsequent sync fails**, and draining repeat scans is the whole
      point of the endpoint.
      Four fixes in one: `scanned_by` now comes from the **token**, never the payload (the scanner
      *is* the authenticated admin — `attend()` already did it this way); **validation added**, there
      was none at all; `checked_in_at` mirrored as `attend()` does since `P033.5`, **only ever set,
      never cleared** so a sync reporting no check-in cannot erase a real one; and
      `check_in_count` stays client-supplied *on purpose* — the device counted scans while offline —
      now validated `integer|min:0`.
      First coverage for the endpoint (5 tests); without the fix 4 of 5 fail, three with 500s.
      Mobile contract checked: absent from `docs/mobile/`, response shape unchanged.
- [ ] **`withoutNotifications` wrapper** for bulk drains — generic suppression so a bulk sync does
      not fire a notification per row. pep `c941802`.
- [ ] **WhatsApp: full set — see section 7.**
- [ ] **`NotificationEventFields`** — renders one notification event's three channels (email, SMS,
      WhatsApp) as a reusable block. Today `categories-form.tsx` hand-writes ~500 lines for three
      events; adding a fourth means copying it all again. pep `fd41aed`.
- [ ] **`admins.is_vip` flag** — independent of role. Small migration + form toggle + a live-resolved
      `actor_is_vip` on the seating audit log, highlighted in the seating app's notification panel.
- [ ] **Seat/table number in the default see-more panel** — present only on the RSVP shape today.
- [ ] **Favicon components reference three assets that are not in `public/`**, while the real ones
      go unreferenced.

### ⚠️ One pep change is NOT safe to port as-is

**`PUT /admin/guests/attend` single check-in rewrite.** `attend()` returns the whole Guest model, so
changing it changes what the **out-of-repo iPad scanner** receives. This is not the Flutter app.
Check `docs/mobile/BACKEND_INCOMING_CHANGES_FOR_MOBILE.pdf` and get owner sign-off first.
(Adding `checked_in_at` is additive and safe; the wider rewrite is not.)

---

## 7. WhatsApp — the full pep list

pep did **not** change the WhatsApp integration. The sender, webhook, OTP, listeners, logs, provider
configs and routes are byte-identical to ours. What pep improved is the **template layer**.

That is because pep forked *after* we built the whole WhatsApp stack ourselves (P24.1–P24.27). It
inherited a complete integration and had nothing to fix in it.

**11 files across 5 commits** (`9b71528`, `976ae89`, `c941802`, `4633938`, `57218c4`, `fd41aed`):

> ### ⚠️ This section was written 2026-08-06 and pep has since redesigned it
>
> **pep replaced the conferences coupling on 2026-08-18** (`f289299` backend, `152d8a0` admin):
> a new **`whatsapp_configs` singleton** owns the event copy, and the renderer resolves `app_name`,
> `event_name`, `event_date`, `event_datetime`, `venue` and `venue_url` from **that** table —
> explicitly so *"WhatsApp owns its event copy rather than borrowing the mobile CMS's."*
>
> **We ported the new design, not the one described below** (`P033.39`). Consequences:
> - The renderer item below is **done differently** — see its entry.
> - **The `/conference` sidebar item is now moot.** It was listed only because the conferences page
>   fed `app_name`/`event_date`/`venue`. It no longer does. The owner had already closed that item
>   as won't-do (section 10), and pep independently did the same thing (admin `eaa0fb8`,
>   *"Hide the conference link from the sidebar"*).
> - The **admin screen** item changes shape: it is now a WhatsApp → Config screen for the singleton,
>   not a conferences link.
>
> **Also found while checking: pep shipped a security batch on 2026-08-16 that this audit predates**
> — four pentest fixes (`SEC-P4/H01–H04`: forgot-password timing, per-address rate limiting, a
> forced-500 debug hook, health-endpoint disclosure). **H02 we had already fixed independently**
> (`afd4eee`). **Owner decision 2026-08-24: hold the batch until pep's remediation is fully closed**
> before porting. Not tracked as items here yet.

**Backend**

- [x] **`WhatsAppTemplateRenderer.php` (+111 lines)** — adds 11 new variables:
      **DONE — backend `959010c`, against pep's CURRENT design.** Added a `whatsapp_configs`
      singleton + `GET`/`PUT admin/whatsapp-config` (gated on the existing `whatsapp_config`
      feature) and resolved **12** variables from it: `app_name`, `event_name`, `event_date`,
      `event_start_date`, `event_end_date`, `event_datetime`, `venue`, `venue_url`, `seat_number`,
      `table_number`, `seat_label`, `checkin_time`.
      **Not ported: `previous_seat_number` / `previous_table_number` / `previous_seat_label`** —
      they read a `guests.previous_seat` column we do not have, so they could only render blank.
      They belong with `GuestSeatingObserver` (section 6), which adds that column.
      pep's **241-line test suite ported and passes unmodified** (11 tests). Design points kept: the
      table ships empty, `show()` creates the row so the form has an id to PUT to, the send path uses
      a plain `first()` so resolving a variable never writes to the DB, and an unset field renders as
      a blank slot rather than the literal variable name. 532 tests pass.
      ⚠️ `whatsapp_configs` (event copy) is one word from `whatsapp_provider_configs` (Meta
      credentials) — warned about in both the migration and the routes.
      - `app_name` — reads `app_name_en` / `app_name_ar` from the `conferences` row, falls back to
        `config('app.name')`. Memoised. Resolves for real even in a test send, so an admin checking
        credentials sees the actual name.
      - `event_name`, `event_date`, `event_start_date`, `event_end_date`, `event_datetime`,
        `venue`, `venue_url` — from the same `conferences` row. Dates render in the target language,
        so Arabic sends get Arabic month names. Multi-day collapses to a `start - end` range.
        **An empty row gives empty strings, not the raw variable name** — a blank slot reads better
        than "event_date" in a live message.
      - `seat_number`, `table_number`, `checkin_time` — from the guest row. `checkin_time` prefers
        `checked_in_at` and falls back to `check_in_time`.
      - All merged into **both** the invitation and guest send paths.
- [x] **`WhatsAppTemplatesSeeder.php` (new, 245 lines)** — 8 templates keyed on `purpose` via
      `updateOrCreate`, so re-running updates in place. **`is_active` mirrors the template's real
      status at Meta**, so nothing that would fail on send ships switched on. Its docblock also
      warns about a real trap: the webhook picks the post-RSVP follow-up by `purpose='qr'` +
      `latest()`, so two active rows make that a coin flip.
      *Port the structure; drop the `pep_*` rows, they are project content.*
- [x] **`DatabaseSeeder.php`** — register the seeder.
      **Both DONE — backend `366a521`.** ⚠️ **Seeds ONE row, not nine.** pep's nine include eight
      naming Meta templates owned by specific projects — `pep_*`, `intvitation_ar`, `ebadge_en`, and
      **`qr_hci_en`, another ALT client's**. A Meta template name only resolves inside the WABA it
      was approved in, so those rows are dead in any clone *and* carry another customer's naming
      into the baseline — the same problem as section 9's hardcoded buckets. Three of them
      (`seat_assigned`, `seat_changed`, `checkin`) also serve `GuestSeatingObserver`, which we do not
      have. This item already said "drop the `pep_*` rows"; the same reasoning covers the rest.
      What is seeded is the one universally true row: **`hello_world`**, which Meta ships in every
      WABA — no variables, no buttons, no approval wait, the fastest proof that token + phone number
      id + queue are wired. Projects add their own approved templates via WhatsApp → Templates.
      ⚠️ **The coin-flip trap was verified against OUR code**, not inherited:
      `WhatsAppWebhookController::qrTemplate()` uses the identical
      `is_active` + `purpose='qr'` + `latest('created_at')` lookup, so the seeder deliberately
      creates **no `qr` row**. Proven: seeding twice yields 1 row and 0 active `qr` rows.

**Admin**

- [x] **`whatsapp-variables-select-ordered.tsx` (new, 243 lines)** — an ordered picker. Position
      **is** the Meta `{{n}}` placeholder, which is why it is deliberately not the existing
      `CheckboxDropdown` (that returns values in option order, not selection order, and would
      silently remap the template). Unknown saved names are kept and flagged amber, never dropped.
- [x] **`data/whatsapp-body-variables.ts` (new)** — catalogue of the 18 names the backend can
      actually resolve, each tagged `scope: both | invitation | guest`.
- [x] **`whatsapp-templates-form.tsx` (+73)** — replaces the free-text comma list. Today a typo
      ships a literal variable name to Meta with nothing flagging it.
- [x] **WhatsApp → Config admin screen** *(replaces the `/conference` link item — see the section
      header).* **DONE — admin `edfa1f7`.** The screen for the `whatsapp_configs` singleton: form,
      types, route, sidebar entry above provider-configs (content before credentials, no
      `superOnly`), and 12 EN + 12 AR keys at 1790/1790 parity. Without it the table stays empty by
      design and all 12 variables render blank.
      Two deviations from pep `152d8a0`: the page wrapper follows our
      `checkFeaturePermission` convention rather than pep's bare `withAuth`, and the component's
      `mode: 'config'` prop — a single-valued union guarding an always-true branch — was dropped
      rather than imported as fresh dead input. `check:rbac` green at 33 links.

- [ ] ⛔ **`notification-event-fields.tsx` (new, 128 lines)** — WhatsApp as one of the three channels.
      **BLOCKED on `GuestSeatingObserver` (section 6).**
- [ ] **`categories-form.tsx` (+118)** — seating events (`on_check_in`, `on_seat_assigned`,
      `on_seat_changed`), each with `whatsapp: { enabled, whatsapp_template_id }`.
- [ ] ⛔ **`interfaces/category.tsx` (+38)** — the types. **BLOCKED on `GuestSeatingObserver`.**

> ### ⛔ The last three section 7 items are blocked on section 6, not merely unstarted
>
> Checked 2026-08-24. `NotificationEventFields` has **exactly one call site** in pep —
> `categories-form.tsx` inside `{SEATING_EVENTS.map(...)}`. pep did **not** refactor the existing
> `on_register_complete` / `on_accept` / `on_reject` events; those stay hand-written. Its own comment
> says so: *"seating trio is rendered by `<NotificationEventFields />`"*.
>
> So the component exists solely to render `on_check_in` / `on_seat_assigned` / `on_seat_changed` —
> and those three events are fired **only** by `GuestSeatingObserver`, which this baseline does not
> have (it needs a `guests.previous_seat` column). Porting them now would give an admin three
> configurable notification events that **nothing can ever fire**.
>
> ⚠️ **This corrects the pitch made for these items**: the value was described as "removes ~500 lines
> of duplication in `categories-form.tsx`". It does not — the duplication stays either way.
>
> **Do `GuestSeatingObserver` first, or leave these.**
- [x] ~~**`data/sidebar-links.tsx` (+20)** — the `/conference` link.~~ **MOOT — not needed.** It was
      listed only because the conferences page fed `app_name` / `event_date` / `venue`. After
      `P033.39` it does not: WhatsApp reads its own `whatsapp_configs` singleton. The owner closed
      the sidebar item as won't-do (section 10) and pep independently hid the same link
      (admin `eaa0fb8`). The conferences row remains the **mobile app's** config, edited via
      `/conference` by direct URL.
- [x] **Translations EN + AR** — 4 keys: `body_variables_add`, `_empty`, `_custom`, `_unsupported`.
      **All four items above DONE — admin `6f852af`** (picker + catalogue + form + keys land
      together; the form is unusable without the picker, the picker without the catalogue).
      ⚠️ **It is 6 keys, not 4** — `move_up` / `move_down` are absent from our translations too.
      EN/AR parity 1796/1796.
      ⚠️ **The catalogue was verified against OUR renderer, not copied.** Its own docblock warns an
      out-of-sync name ships to Meta as a literal. Two mismatches: **`seat_label`** is resolvable
      here but missing from pep's list (added), and **`reconfirmation_url`** looked missing from our
      renderer but is resolvable — minted conditionally in `forGuest()`, outside the maps so a token
      is only created when a template uses it. Final: **26 entries, every one resolvable**, checked
      programmatically.
      The catalogue count in this item ("18 names") is also stale.

---

## 8. TODO — 124-ewc-2026-v2

- [x] **Invitation link fix + the 127-line `InvitationLinkTest`** — `e647bb9`. **Take this version,
      not gfeai's.** It keys purely on `remaining`, and it comes with tests. It also documents why
      gfeai's first attempt (`21037e6`) was wrong: gating on `usage_type` still blocked
      number-of-use invitations whose `usage_type` was `single` or unset.
- [x] **Plain `build` script** — see section 4.
- [x] **Derive `is_saudi` from the ISO country code**, not the country's display name. Today it
      matches a localized label, so it breaks in Arabic. The shared country select cannot even
      expose the code — that needs adding. `dcee22e` / `38b4209`.
      **DONE — frontend `eb12414`, admin `d5c2e0d`.**
      ⚠️ **"Breaks in Arabic" is wrong.** The code checks *both* labels, and the hardcoded Arabic
      string is **byte-identical** to the seeded `name_ar` — verified before changing anything. It
      works today in both languages.
      **The real defect is fragility, and the blast radius is why it was worth doing:** countries
      are editable from the admin, so a rename, a spacing fix or appending "(KSA)" silently flips
      `is_saudi` to false — which then wrongly sets `document_type`, `religion`,
      `residency_status`, `valid_visa` and `document_number`. Silent, surfacing much later in a visa
      workflow. Second trigger: `label` gains a phone code in `isPhoneCode` mode, so the match
      already failed on those variants.
      **Fixed in BOTH repos.** The admin guest form carried the identical two hardcoded strings;
      fixing only the frontend would have left the half-fix pattern this backlog keeps turning up.
      Neither select exposed `code` — though the backend already returned it
      (`CountriesSelectResources`) and the frontend select carried it in `raw: c`, just untyped.
- [ ] **Two-step form dead end** — see the gfeai frontend list; ewc fixed it too.
- [ ] **`DATE_COLS` in the guest export point at the wrong columns** — folded into the export
      mechanism job above.
- [ ] **Favicon tags reference three assets that do not exist** in the repo.

---

## 9. Problems in our own baseline — no fork fixes these

Found by auditing our code directly. No fork comparison could have surfaced them.

- [x] **Two forced-500 test hooks on public routes** — see section 4.
- [x] **A `dd()` in committed controller code** — see section 4.
- [x] **Public, unauthenticated, unthrottled `/pdf/{id}` routes.** `routes/web.php:24-25`. The `web`
      middleware group has no throttle and no auth. Each request runs Imagick + PDF rendering and
      **writes a new file to public storage**. Anyone can call it in a loop.
      **DONE — backend `f2976e1`, deleted rather than throttled.** Worse than described on reading:
      each request wrote **two** permanent files to public storage (the PDF *and* the Imagick PNG),
      so throttling would only slow the disk fill. The card also embeds `personal_img_url` from the
      **private** disk (a D14-protected guest photo) into a **public**-disk file, keyed on a
      guessable `registration_number` — so it could materialise any guest's photo and details on
      demand. Safe to delete because the handlers were named `getSocialCardWebTest` / `…Ar` with
      **zero callers in any of the five repos**, and a working twin already exists:
      `GET /api/get-social-card{,-img}/{id}` → `getSocialCard` / `getSocialCardImage`, which the
      frontend actually calls and which sit under `throttle:public-api` (30/min by IP).
      134 controller lines + 2 routes removed.
- [x] **Five unrouted OTP endpoints that skip reCAPTCHA.** In `AuthController` — one has the
      reCAPTCHA rule commented out. Each mints an email-verification token. Not routed today, so it
      is a latent bypass, not a live one. Delete them or gate them.
      **DONE — backend `ef17898`, deleted.** `sendGuestOtpNotification`, `sendGuestOtpMailable`,
      `emailVerificationTemp`, `emailVerificationTempEmpty`, `emailVerificationTempWithMailGun` —
      all unrouted, zero callers. The live path (`emailVerification`, `routes/api.php:171`, under
      `throttle:sensitive-api`) enforces reCAPTCHA and is untouched. Route count 476 → 474, entirely
      accounted for by the `/pdf` pair above. **Bonus:** the dead code held **7 of AuthController's
      11 `env()`-outside-config calls**, so its baseline count dropped 11 → 4. 288 lines removed.
- [ ] **Hardcoded buckets belonging to other ALT clients**, on live render paths:
      - `GuestsController.php:3256` — a **devego** bucket, on `POST /generate-static-badges`
      - an **ims** bucket sticker URL on three badge-PDF paths
      - plus tourise, hci-2026, glmc, faf and `sec-emails` in PDF and email templates
      Every clone renders client-facing PDFs pulling images from a different customer's storage.
      **The fix mechanism exists in gfeai `60ba649`** — `config/pdf.php` keys, env-backed,
      defaulting to null. Port the mechanism, then extend it to the buckets gfeai did not cover.
- [x] **`X-Mailer` uses `env()` outside a config file**, so it sends **empty under `config:cache`**
      — i.e. in production.
      **DONE — backend `c1b118c`.** Scoped to the whole defect, not the one named line: the same
      three-line `replyTo` + `X-Mailer` block is copy-pasted across **four** mail-sending files, so
      **11 calls in 6 files** were swapped to their existing config keys
      (`config('mail.from.address')` / `config('mail.from.name')` / `config('app.frontend_url')`).
      Two real production consequences: guest OTP, admin login OTP and template test-sends went out
      with **no Reply-To and an empty `X-Mailer`**; and invitation exports wrote join links starting
      `/en/join/…` **with no domain**, making the workbook handed to a client unusable. Verified
      under a real config cache (`env()` → `NULL`, `config()` → real value).
      ⚠️ **Not finished — two groups deliberately left, each needing its own item:**
      - ~~**`DynamicSmtpService` (4 calls).**~~ **DONE — backend `ee002b0`.** `applyFallbackConfig()`
        exists to *restore* the `.env` defaults after a dynamic SMTP override, so `config()` there
        would read the value it is about to overwrite — which is why `P033.24` skipped it. It now
        **snapshots the pristine `config/mail.php` values once per process on first construction**,
        necessarily before any override, since every mutation of those keys happens inside this
        class. Under `config:cache` the old `env()` reads returned `NULL`, so the "restore" was
        **wiping the From address** — the incident ledger D971 records.
        ⚠️ **Test honesty:** the new test pins the contract but passes **with or without** the fix,
        because the suite never runs with a cached config and that is the only condition where the
        bug appears. The cached-config demonstration is the real evidence.
        ⚠️ The snapshot is `static`, so a long-lived queue worker holds it for its whole life —
        **`queue:restart` after a deploy is now load-bearing for mail config too** (already required
        by the prod runbook).
      - **No config key exists yet (10 calls, 5 files):** `FRONTEND_BASE_URL` +
        `REVALIDATE_SECRET_TOKEN` (Speakers / Sponsors / SpeakerLabels / SponsorLabels controllers),
        `SECURITY_ALERT_EMAILS`, `MAIL_HOST_BULK`, `MAIL_EHLO_DOMAIN`. Mechanical, but each needs a
        key added.
      **Debt: 12 files → 6.** `phpstan-baseline.neon` shrank 27 lines. Note the larastan rule
      `noEnvCallsOutsideOfConfig` catches this class automatically — it is what surfaced the
      reCAPTCHA outage, so **do not re-add baseline entries for it.**
- [x] **⚠️ `GOOGLE_RECAPTCHA_SECRET` is read with `env()` outside a config file** —
      `app/Rules/ReCaptcha.php:31`. Same defect class as `X-Mailer` above, but the consequence is far
      worse: Laravel does not load `.env` when the config is cached, so under `php artisan
      config:cache` — the normal production step — **the secret is `null`**. Google then replies
      `success: false, invalid-input-secret`, and **every login and every registration is rejected**,
      at all **13** call sites. There is no `recaptcha` key in `config/` at all. *Found 2026-08-08
      while doing the section 4 guard; not surfaced by the original audit.* **Verify against the real
      deploy before anything else — if it runs `config:cache`, this is a live outage waiting on the
      next cache clear.** Fix: a `config/services.php` entry + `config('services.recaptcha.secret')`.
      **DONE — backend `bf31b85`.** Proven on a real config cache before and after:
      `env('GOOGLE_RECAPTCHA_SECRET')` → `NULL`, `config('services.recaptcha.secret')` → present.
      **larastan was already catching this** (`noEnvCallsOutsideOfConfig`) — the finding sat
      suppressed in `phpstan-baseline.neon`; fixing the code made the ignore unmatched, which turned
      the gate red and forced its removal. **12 baseline entries for the same rule remain across 13
      files** — `DynamicSmtpService` (4 calls) is the next worth looking at, and ledger D971 records
      that it has already caused a real incident.
- [ ] **`check_in_time` and `checked_in_at` hold the same fact.** Two generations of attendance
      columns: the 2023 `check_in` / `check_in_time` / `check_in_count` set, and the July-2026
      seating set (`checked_in_at` + `checked_in_by`, plus check-out and food pairs). A
      `Guest::booted()` hook copies between them so whichever client writes, both stay true. The
      legacy columns cannot simply be dropped — admin filters, reports and the out-of-repo iPad
      scanner read them. *Raised 2026-08-08 while doing section 4's `checked_in_at` item.* Not a
      quick win: needs a migration, an admin/reports sweep and scanner sign-off.
- [ ] **20 export classes duplicate the same row-number binder** with no shared trait.
- [ ] **`composer.lock` is untracked in all four backend repos.** Every clone's `composer install`
      resolves fresh versions. Non-reproducible builds, and no supply-chain pinning — which sits
      awkwardly next to the OWASP hardening this baseline advertises. *(No fork fixes this either;
      noting it as a decision to make.)*
- [ ] **Admin, frontend and seating have zero automated tests, and there is no CI in any of the 18
      repos.** The backend has 24 feature test files. Most of the bugs in this document are
      client-side, where we have no coverage at all.

---

## 10. What we decided NOT to port

Each of these was checked and rejected with evidence.

| Item | Why not |
|---|---|
| **SMS non-production guard** | We removed this **on purpose** in P23.2 (`28eca40`) on 2026-07-23, *after* gfeai forked. gfeai never added it — it just never got our removal. Porting it would reverse a deliberate decision. **Raise as an owner question, not a port.** |
| **CSP `connect-src` for reCAPTCHA** | Contested — see section 11. |
| **Route shadowing fixes** (`2f21800`, `d4ed8cd`) | Fork-native. An agent wrote a route-table parser, proved it against gfeai's pre-fix file (found exactly the 4 routes gfeai later fixed), then ran it on ours: **0 shadowed across 467 routes.** Note the reason is *declaration order*, not `whereUuid` — 65 of our routes have unconstrained params and are safe only because their literal siblings come first. |
| **Duplicate PUT / guest-edit 500** (`b0f275a`) | **0 duplicate routes** and **0 routes pointing at a missing handler**, across all 467. |
| **Email-log boolean filters, backend half** | We fixed this independently in P23.11 (`7eb1e54`) — five days *before* gfeai did, in both controllers, for all four filters. **The admin half is still broken** (section 5). |
| **`/join` landing page** | The fork's version is a 35-line empty shell. |
| **Most gfeai work before 2026-07-22** | We built the same things in parallel. Reading all 166 unread commits found only 8 real gaps, all listed above. |
| **Un-commenting the mobile-CMS sidebar links** (section 4) | **Owner decision, 2026-08-08.** The block is not the three links section 4 described — it is a nine-module "Mobile App Section", hidden on purpose in admin `1e449dc` during a sidebar restyle. The pages stay reachable by direct URL and every module is still RBAC-gated, so nothing is broken by leaving the nav as it is. Restoring all nine would change the admin nav in every clone off the back of a miscounted audit item. **Do not re-raise on the next audit.** If a project needs the mobile CMS surfaced, restore the block in that clone. |

---

## 11. Still unanswered

- **Does our CSP block reCAPTCHA on admin login?** Two agents, opposite conclusions, both citing
  source. One proved the wrapper only injects a script (which `script-src` already allows) and makes
  zero `fetch`/`XHR` calls. The other says `connect-src` at `next.config.js:55` cannot contain
  `google.com`, and points at three submit handlers that `await executeRecaptcha`.
  **Settle it in a browser, not from source.** Open admin login with devtools and watch for a CSP
  violation. Adding the two Google hosts is cheap insurance either way.
- **Is the backend's Google reCAPTCHA secret the test one?** We do not read `.env`. Please check.
- **Some scope numbers are single-agent claims.** The audit was stopped before the verifying critic
  ran. The counts of 14 sites / 46 components / 44 listings / 7 blob call sites are probably right
  but unchecked. The findings themselves were all verified.

---

## 12. Notes for whoever does this

- **Process:** see [How we work through this list](#how-we-work-through-this-list) at the top —
  **one item at a time, and no commit until the owner has reviewed the diff.**
- **Order:** section 4 first. They are cheap, independent, and cover most of the high-severity list.
  Then the broken-things lists. Save the five large jobs for last — they need a decision, not a port.
- **Do not copy commits blindly.** Several fork fixes are incomplete or wrong for us:
  the invitation-collections fix misses three methods; gfeai's upload fix covers one of two files;
  gfeai's CSP fix would undo our env-derived design; gfeai's first invitation fix is superseded by
  its own later one.
- **Two ports need a mobile-contract check** before they land — see the warning in section 6 and
  hard rule 4 in `CLAUDE.md`.
- **Prefer porting mechanisms over values.** The export job is the clearest case: fixing the column
  letters leaves 28 classes ready to break again on the next column insert.
- Every claim here has a file and line number. If something looks wrong, open the file — that is
  faster than re-running the analysis.

---

*Analysis: three passes, 34 of 36 agents completed (2 stopped early — see section 11).
Nothing in this document has been ported yet.*
