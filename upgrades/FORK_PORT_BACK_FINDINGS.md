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

- [x] **Export column letters point at the wrong columns.** **Done (letters only)** — `P033.33`
      (`fa4f4a8`, the `InvitationsExport` crash) and `P033.34` (`817742c`, the guest-export date
      columns), plus `ExportColumnAlignmentTest` — our first export test. **The headings-lookup
      mechanism was explicitly declined**, not forgotten: it would rewrite all 28 classes in
      `app/Exports/` to close a fault that only ever affected 4. Parked in `LEFTOVERS §2.9`.
      Original item text below, kept for the record — note its "port the mechanism, not corrected
      letters" instruction is the part that was overruled.
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

- [x] **Admin CSP blocks 8 social icons.** `soical-media-modal.tsx:43+` hardcodes them on
      **another ALT client's bucket** (`emails-alt.eu-central-1.linodeobjects.com`). It is never
      derived from env, so the CSP at `next.config.js:54` cannot cover it and all 8 render broken.
      **Do not copy gfeai's fix** — it widens the CSP to a wildcard bucket host, which undoes our
      env-derived design. Move the icons to the project's own storage instead.
      **Done** — `P033.69`, backend `887d324` + admin `3de202f`. Owner decision: serve them from our
      own backend. The 8 PNGs are tracked in the backend's `public/social/` and resolved through a
      new `socialIconUrl()` in `utils/storage.ts`.
      ⚠️ **Worse than the item says.** The broken preview is the visible half; the chosen URL is
      stored on `SocialMediaLinksPerTemp` and **embedded in outgoing email**, so every clone was
      sending guests images served from a customer's bucket it does not control.
      ✅ **NO CSP CHANGE WAS NEEDED** — the warning above is satisfied by construction, not by
      restraint. `next.config.js` already derives `envOrigins` from `NEXT_PUBLIC_STORAGE_URL` via
      `new URL(url).origin`, and the icons now sit on **that same origin**. Verified, not assumed:
      `https://api.example.com/social/linkedin.png` against a CSP origin of `https://api.example.com`.
      **Two constraints that shaped it:** `storage/app/public/.gitignore` is `*`, so nothing there
      can be tracked and the icons could not live behind `/storage` and still ship with the repo —
      hence `public/`, served at the app origin. And `socialIconUrl()` strips the trailing
      `/storage` rather than adding a second env var that could drift from the first; the reasoning
      is in its docblock so it is not "tidied" into a new variable later.
      The URL must stay **absolute** — it is embedded in mail, where a relative admin path would not
      resolve.

### Missing features

- [x] **Add an `is_sent` filter** to the invitations listing (backend + admin). gfeai `12f6a62` /
      `7effa14`. **Done** — `P033.50`, backend `dc42aaa` + admin `8f79f11`, with
      `InvitationIsSentFilterTest` (3 tests). Gated on **`filled()`, not `has()`**: the admin sends
      every filter key on each search, so an untouched dropdown arrives as `?is_sent=` and
      `boolean('')` is `false` — under `has()` the listing would quietly show unsent invitations
      only, a working-looking filter nobody set. gfeai flagged this in their own commit; it is the
      same trap as `LEFTOVERS §2.10`. **The test was verified to catch it** — flipping the line to
      `has()` fails the cleared-dropdown case while the other two pass.
      ⚠️ The neighbours `is_active` and `usage_status` **still use `has()` and still carry the
      defect** (`?is_active=` filters to inactive only, `?usage_status=` returns nothing).
      Deliberately not fixed here — that is the `LEFTOVERS §2.10` batch, which wants deciding
      alongside the admin's `String(value?.value)` bug so it can be verified end to end.
      No new translation keys: `web:is_sent`, `web:yes`, `web:no` already exist in EN **and** AR.
      `routes/api.php` untouched — a query parameter on an existing endpoint, so no mobile-contract
      impact.
- [x] **E-visa bulk download** — selected / filtered / all pages / in-progress only, as one zip.
      Today "Download all" fires one browser download per guest and browsers block the burst.
      Paired backend + admin job. gfeai `01582a2`, `2f917ec`, `48b8578`, `528c977`, `b6b84b9`.
      **Done** — `P033.66` backend `054903e` + `P033.67` admin `993ef97`, with
      `EVisaBulkDownloadTest` (10 tests). `POST /admin/guests/e-visa/download-bulk`, gated
      `admin.can:e_visa`. Each guest's package is already a zip on the private disk, so each is
      extracted and re-added under a folder named by registration number rather than nesting zips.
      Chunked at 100, streamed in 512 KB reads, scratch swept, 1000-package ceiling.
      🔴 **THE ADAPTATION THAT MATTERED — do not "restore" gfeai's gate.** They gate on
      `require_visa === 'yes' AND valid_visa === false`. **We have no `require_visa` column** (zero
      references in migrations, model or controller), so their gate is an unknown-column error here.
      More importantly `EVisaListing` gates on `valid_visa === false` alone and its docblock forbids
      widening it — the rows carry passport-grade PII. The bulk endpoint uses **our listing's gate
      exactly**: a bundle that could include a guest the console never showed is a leak.
      **Structural improvement taken from gfeai:** `applyEVisaConsoleFilters` is extracted and called
      by **both** the listing and the bulk download, so the `filtered` scope cannot drift from what
      the console shows. Our `EVisaListing` was refactored to share it.
      ⚠️ **Third fork post-image that would have reverted our work:** our `e-visa-listing.tsx`
      differed from gfeai's pre-image by 25 lines, all of them our `messageOf` /
      `readApiErrorMessage` blob-error utilities; their version reverts to `getApiError`, which
      cannot read a message out of a Blob. Applied their change onto ours instead. (See also
      `ListingFilters` `modal={false}` and `guestsSyncOffline`.)
      Ported the **refined** final state (`b6b84b9`, net −62 lines: one server-scoped action) rather
      than the intermediate `528c977`. **Not ported:** `01582a2` also edits
      `visa-application-form.blade.php` — a blade we do not have (ours is `visa-document.blade.php`),
      and the "enhance form rendering" half of that commit's title.
- [x] **Clone an email template** (with its attachments). We already clone categories, SMTP configs
      and badges — templates are the gap. Paired backend + admin. gfeai `1a423f5` / `3f97f70`.
      **Done** — `P033.62` backend `0b5d093` + `P033.63` admin `56c1593`, with
      `EmailTemplateCloneTest` (7 tests). Mirrors `BadgesController::clone` and additionally copies
      the attachment rows, **the files behind them**, and the social links.
      **Two departures from gfeai:** their blanket `catch (\Exception)` returning
      `$e->getMessage()` with a 500 was dropped — echoing raw exception text leaks table names and
      paths, and Laravel already returns a clean 500; only the `ModelNotFoundException` → 404 is
      kept. And **gfeai shipped this untested by their own admission** (*"the full test suite and
      phpstan were NOT re-run … feature tests are still outstanding"*), so the tests are new.
      The tests pin the silent-failure cases: a `blocked` row (this module's delete gesture) is not
      copied, a row whose file is already **missing from disk** is skipped rather than fatal and does
      not leave a row pointing at nothing, and files are asserted **on disk** at the copy's path.
      Confirmed our layout is `attachments/{template_id}/{lang}/{file}`, identical to gfeai's — had
      it differed, their `continue`-on-missing would have produced attachment-less clones with **no
      error at all**.
      🔎 Fixed in passing: `body_en`/`body_ar` are **not columns** on `email_templates` (they are
      `content_en`/`content_ar`), so `fill()` silently dropped them. The same wrong field had been
      introduced into `GuestSeatingObserverTest` and `GuestOfflineSyncTest` earlier in this task.
- [~] **Bulk missing-data notify** — `POST /admin/guests/notify-bulk` with a sent/skipped tally.
      Hidden inside a commit titled "feat: rebrand". gfeai `4c6187e`.
      **BACKEND DONE** — `P033.64`, backend `db69a10`, with `GuestNotifyBulkTest` (8 tests). Per
      guest the work is identical to our single-guest `NotifyMissingData`; the difference is that a
      guest who cannot be sent to is **skipped and returned by id** rather than 422-ing the whole
      run. Two skip reasons: no `missing_data_email` on the category, or no email on the guest.
      Route sits inside our `admin/guests` group with `admin.can:guests_listing,notify` —
      **gfeai's route has no permission gate at all**. `ids.*` validated as `uuid`, not their
      `string`.
      ⛔ **ADMIN HALF DELIBERATELY NOT PORTED — it is dead code in gfeai too.**
      `SendMissingDataBulkModal` has **no caller anywhere in gfeai's tree**: every reference to it is
      inside the modal file itself (declaration, definition, export). They built it and never wired
      it up. Separately, **our guests listing has no multi-select** — no checkbox, no `selectedIds`,
      no selection state — and the modal's whole contract is `selectedIds: string[]`. Porting it
      would reproduce their dead code exactly.
      **What is actually missing is multi-select on the guests listing**, which is its own piece of
      work and is not what this item describes. The endpoint is live, gated and tested, so it is
      callable the moment a selection UI exists — or from an ops script today.
      ⚠️ Also needs `res:notify_bulk` / `res:notify_bulk_skipped`, absent from our translations.
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
- [x] **Skip reCAPTCHA outside production** (opt back in with `RECAPTCHA_ENFORCE`). Local dev
      currently cannot log in or register. gfeai `eafd3e4`. **Done** — `P033.49`, backend `492eb7a`,
      with `ReCaptchaEnvironmentTest` (5 tests). **Two deliberate departures from gfeai:**
      **(1) `local` only, not `! production`.** gfeai skips in every non-production environment,
      `testing` included — that turns `ReconfirmationTest`'s `fakeRecaptchaPass()` into a no-op and
      leaves the suite never exercising the verification path: green tests proving nothing. Their
      commit message lists tests as a *reason* to skip. Scoping to `local` fixes the real complaint
      without paying in coverage, and keeps **staging** enforcing.
      **(2) `config()`, not `env()`.** gfeai put `env('RECAPTCHA_ENFORCE', false)` in the rule —
      null under `config:cache`, and our PHPStan `noEnvCallsOutsideOfConfig` fails the build on it.
      The key now sits in `config/services.php` beside `secret`. gfeai's other half — the
      `?? false` guard — we already had.
      The test pins what must not follow: `testing` still calls Google and honours a `false`, `local`
      never reaches Google (`assertNothingSent`), `RECAPTCHA_ENFORCE=true` restores the challenge,
      and **production and staging both still verify**.
      ⚠️ **The secret warning is NOT closed and is prod-affecting.** gfeai reports the backend's
      `GOOGLE_RECAPTCHA_SECRET` is Google's **test** secret (`6LeIxAcT…`), not matching the site key
      the apps use (`6LeW02kq…`); siblings pair that site key with
      `6LeW02kqAAAAAL3GDbr_MFN3y0rkb_uZ_LhjyppG`. If it holds here, **production reCAPTCHA accepts
      everything** — the test secret passes any token. `P033.49` does not affect this either way.
      We do not read `.env`; check it on the real deploy. See `LEFTOVERS §4`.
- [ ] **Multi-file upload input** with in-browser image downscaling. gfeai `df4c152`.
      ⛔ **AUDIT 2026-08-25 — NOT PORTED. The item overstates what this is.** Three reasons, any one
      of which is disqualifying:
      **(1)** it posts to **`/upload-media-work`**, which does not exist here — that is
      `uploadMediaWorkAdmin`, added in gfeai's rebrand commit for their GFEAI-26 media form. Porting
      the component means also porting a backend endpoint this item never mentions.
      **(2)** it is **orphaned in gfeai's own tree** — `git grep` for `custom-media-input` /
      `CustomMediaInput` across their HEAD returns only the file itself. Nobody imports it. That is
      the **third** orphaned component found in the forks (see also `SendMissingDataBulkModal` under
      bulk notify, and `resetFilterPreferencesCache` under filter visibility).
      **(3)** it is not a generic input. Its own header reads *"GFEAI 26 — Media (media_gfai): admin
      multi-file 'previous work' upload"*, with `MAX_FILES = 5`, a 10 MB video cap and a shape bound
      to their `media_gfai` form — **a form shape this baseline does not have**.
      Porting it would yield a component nothing renders, posting to a route that 404s, for a shape
      that does not exist — worse than dead code, since it would also need a new endpoint to become
      anything.
      **The salvageable part is ~30 lines:** the canvas downscale to `IMAGE_MAX_DIM = 1920`
      (`custom-media-input.tsx:60-75`). If in-browser downscaling is wanted, lift that into the
      existing `custom-file-input-2` components as its own task — that is building a capability, not
      porting a fork feature.
- [x] **Searchable guest-type picker** instead of an unbounded card list. gfeai `c127e36`.
      **Done — with a caveat: it is a bounded dropdown, NOT searchable.** `P033.53`, admin `bd4bc2a`.
      Our `CustomSelect` is a re-export of the Headless UI `ui-select` primitive (migrated off
      react-select), and its interface marks `isSearch` / `isLoading` as *"accepted and ignored —
      no-ops on Headless UI"*: both are declared in the type and **never referenced in the component
      body**. gfeai passes both; here they would be dead props, so they are not passed. What ships is
      a bounded dropdown with Listbox **typeahead** (type letters to jump) — it solves the real
      problem, the unbounded list, but is not a filter box. **Real search would mean adding a
      Combobox path to `ui-select`, which affects every select in the admin** — that is the follow-up
      if search is actually wanted.
      **Kept our `useFetch`:** our file is *ahead* of gfeai's pre-image (they hand-roll
      `useState`/`useEffect`/`Axios` against `/categories-select`), so only the UI change was applied
      — taking their post-image would have regressed the refactor and changed the endpoint.
      The `/${lang}` prefix is load-bearing: `i18n/link.tsx` adds it inside `NextLink`, and
      `router.push` does not, so dropping the link means writing it out or every navigation 404s.
      No new keys — reused `web:select`. **Not ported** from the same commit: a Questionnaire heading
      across six gfeai-only form shapes, and their religion-key trimming (project data).
- [x] **Per-option help text on radio groups** — `custom-radio-input.tsx` can only show one shared
      label and only lays out horizontally. gfeai `03d2411`. **Done** — `P033.52`, admin `f897b89`.
      `CustomRadioInput` takes an optional per-option `description` and stacks vertically **only when
      descriptions are present**, so every other radio group is untouched. Our file was verified
      byte-identical to gfeai's pre-image first, so it is a clean port rather than a merge.
      **Translations:** the two new keys are a decomposition of our *existing* `usage_type_help`
      text — the AR `single` string is a verbatim match of our AR's first clause. gfeai's `multiple`
      string adds "up to the number of uses you set" in both languages, which the old copy never
      said. `usage_type_help` was **removed rather than orphaned** (gfeai left it behind unused; it
      had exactly one consumer here). EN/AR key sets verified identical in both directions.
- [ ] **Scroll hint on listing tables.** ~~Every listing is a fixed 1120px table; on a laptop the
      Actions column (Edit/Delete) is off-screen with nothing indicating it.~~ **47 components import
      `ListingTable`** — one edit fixes all of them. gfeai `cd0a861`.
      🔎 **AUDIT 2026-08-24 — the premise is wrong; NOT ported. Needs a browser before anyone tries
      again.** `ListingTable` defines `STICKY_COL = 'sticky ltr:right-0 rtl:left-0 bg-white z-10'`
      and applies it via `isActionsColumn(col.key) && STICKY_COL` in **both** render paths, so
      **Actions is pinned to the trailing edge and never scrolls off** — Edit/Delete are always
      reachable. What remains true is milder: the *middle* columns scroll with no affordance.
      **gfeai's fix collides with that sticky column.** Their overlay is a 48px (`w-12`) fade pinned
      to the same trailing edge at the same `z-10`, so it either washes out the Edit/Delete buttons
      or paints behind them and is invisible — which one depends on stacking-context resolution
      between a `z-10` absolute sibling and a `z-10` sticky descendant, and **that cannot be settled
      from source**. gfeai carries the identical conflict.
      Two further defects in their version: it is **not RTL-aware** (hardcoded `right-0` /
      `bg-gradient-to-l`, while our table uses `ltr:`/`rtl:` variants throughout, so the hint points
      the wrong way in Arabic), and it is **always on** (no overflow detection, so a wide monitor
      where the 1120px table fits still shows a permanent arrow that lies).
      **If picking this up:** a corrected version is ~20 lines — detect overflow via
      `scrollWidth > clientWidth` with a `ResizeObserver`, and place with `ltr:`/`rtl:` variants plus
      `rtl:rotate-180` on the chevron. Avoid tracking scroll *position*: RTL `scrollLeft` sign is
      inconsistent across browsers. **Check the sticky-column overlap in a browser first** — if the
      fade is invisible behind Actions, the whole affordance needs to sit inside the scroll container
      instead. This is a visual change across 47 screens; it should not ship unseen.
- [x] **Show consumed uses / limit** on the invitations listing instead of a raw send count.
      gfeai `5520a33`. **Done** — `P033.51`, admin `3688054`. Renders `uses / valid_up_to`.
      **Admin-only — no backend change:** `InvitationResource` already exposes `uses`,
      `valid_up_to` and `send_count`, `index()` genuinely returns that resource, and
      `web:number_of_use` already exists in EN and AR. Dropped gfeai's `?? 0` / `?? '-'` fallbacks —
      both columns are `unsignedInteger NOT NULL DEFAULT 0`, so neither can fire.
      ⚠️ **Consequence:** this removes the **only** place invitation `send_count` is visible in the
      admin — every other `send_count` in the codebase is `issued_visa_send_count`, an unrelated
      field on guests, and no see-more panel shows it. `send_count` and `uses` are different facts:
      how many times someone was *chased* versus how many times the invitation was *consumed*. The
      item's wording ("instead of") and gfeai both call for the replacement, so that is what shipped;
      the field remains in the API and the TS interface, so restoring the column is two lines if the
      chase count matters operationally.
- [ ] **Bulk guest hard-delete by email list**, with preview and a workbook export. Operator tool.
      gfeai `e1273d4`, `c4dfcbc`, `382b13d`.
- [x] **Tests we do not have:** first coverage for the invitation store endpoint and the guest
      listing filters. gfeai `4d4fed0`, `6622fcc`. **Done in two halves** — the invitation store
      endpoint by `InvitationStoreTest` (`P033.27`), the listing filters by
      `GuestListingFiltersTest` (`P033.47`, backend `1be7d3e`): 10 tests / 51 assertions over
      `will_attend`, `reconfirmed_will_attend`, `require_flights`, `require_accommodation`,
      absent-and-empty handling, and AND-combination. `applyFilters()` is also what `selectIds()`
      runs, so the automation picker's "select all N matching" is covered too.
      **Not ported:** about half of gfeai's file tests `missing_data_sent` / `logistics_completed`,
      which read `CompletionToken` and `CompletionTokenService` — their `completion_tokens` port, a
      §5 big job we have not taken, so those tests have nothing to run against here.
      ⚠️ **Two defects found while writing it**, documented not fixed — `LEFTOVERS §2.10` (the ~16
      `has()`-guarded filters return zero guests on an empty parameter, and the admin can send one)
      and `LEFTOVERS §2.11` (`where_null` is a duplicated block whose ungrouped `orWhere` discards
      every other filter, `selectIds()` included).

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

- [x] **Two-step form shapes strand the user.** `renderFormSteps` step-2 branch is not
      last-step aware, so the guest submits, the record is created, and nothing happens next.
      **Fixed independently in both gfeai and ewc** — two forks hitting it is a strong signal.
      **Done** — `P033.46`, frontend `f4cb529`. `isLastStep` was computed at line 110 but only used
      to derive `nextStepId`, never to change behaviour.
      ⚠️ **Cannot fire in this baseline today** — this is why the earlier liveness check on this item
      came back negative, and the item does not say so. Our shapes are `default` (1),
      `default-one-step-rsvp` (1) and `default-four-steps` (4), so `personal-info-2` is only ever
      index 1 and `flights-info` only ever index 2. The commit changes nothing observable here; it
      matters because this repo is the **clone baseline** — gfeai added six 2-step and two 3-step
      shapes, ewc added `ewc-two-steps`, and both hit it immediately.
      **Fixed both branches**, not just step 2: the root cause is a branch that is not last-step
      aware, and `flights-info` has the identical defect for a 3-step shape (gfeai, which has 3-step
      shapes, fixed both). **Not ported:** gfeai also threads `withToken` / `token` /
      `onCustomSubmit` through these branches — `step-3.tsx` consumes none of them and `step-2.tsx`
      only `token`, which it already had, so porting them would be dead plumbing. `optionalFields`
      appears throughout both forks' copies of this file and is not ported either.
      **Limitation:** this fixes navigation only. A clone building a real 2-step shape still needs a
      `step-2` that POSTs — ours is a middle step and does not submit.

- [x] **Admin-controllable privacy consent** — today the consent checkbox is hardcoded per shape,
      one shape has none, and one records consent silently. gfeai `bfebdfc` / `0bef709`.
      **Done** — `P033.68`, frontend `f1e0b74` + backend `d53b4d8`. **Owner decisions:** required in
      **all** shapes, and **text only, no policy link** (this baseline ships no policy document, and
      a placeholder a clone forgets to replace is worse than no link).
      **Not "admin-controllable" in the end** — gfeai's `0bef709` makes `isRequired` configurable;
      the owner chose always-required, which is the stronger position and removes the toggle.
      The item's description held exactly for our three shapes: `one-step` had a hardcoded
      **paragraph** asserting agreement with nothing to tick; `one-step-rsvp` called
      `setValue('terms', 'yes')` on submit; `fours-steps` was already correct and is **untouched**.
      🔴 **A second silent-consent site the item does not mention:** `GuestsController:1098` built the
      **decline** path with `'terms' => $request['terms'] ?? 'yes'`, recording consent for a guest who
      declined and may never have seen a checkbox. Now `?? null`.
      ⚠️ **A runtime crash caught before shipping:** gfeai's component reads `useFormContext()`, and
      **none of our shapes wrap their form in a `FormProvider`** — the context would be null and the
      component would throw, while `tsc` passed clean. Rewritten to take `register` / `errors` as
      explicit props, with the reason in the docblock.
      Only **2 of gfeai's 16 files** were relevant; the other 14 are form shapes we do not have plus
      their own policy PDF.
      🔎 **FINDING, not fixed:** the frontend carries **103 `privacy_policy_*` keys in EN with no
      Arabic** — the whole privacy-policy page renders only in English, so Arabic guests now tick a
      consent referring to a policy they cannot read in their language. Pre-existing (parity was
      606/503 before this change), out of scope, **wants its own item**.

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
- [x] **Fix the seating chair overlap.** ~~The seating app draws chairs on top of each other at every
      table corner — 0px apart on U-shape tables, 55.1px apart for a 56px chair on rectangles once
      `headSeats >= 2`.~~ pep `8c39930`. **Done** — `P033.44`, seating `9823552`.
      Measured before/after against the real chair size: U-shape `head >= 2` **0.0px → 45.3px**,
      U-shape `head = 1` 50.0 → 59.4, rect `headSeats 2-3` 55.1 → 70.0, rect `headSeats 1/4`
      60.1 / 67.4 → 70.0. Two corrections to this item as written:
      **(1)** the rectangle was never overlapping — `ELEMENT_DEFAULTS.chair.width` is **46**, not 56,
      in both this repo and pep's (pep's own code comment repeats the wrong 56px figure), so at
      55.1px there was ~9px of clearance. That half is a spacing improvement, not a bug fix.
      **(2)** the U-shape bug is real and exactly as described — two chairs computed on the identical
      point. Residual, deliberately left: U-shape corners now sit at 45.3px against a 46px chair, so
      adjacent chairs *touch* rather than stack; closing that means changing `seatGap` or the corner
      rule, a visual-design call rather than a bug fix. Only `src/utils/layout.js` was taken —
      `8c39930` bundles the unrelated `admins.is_vip` flag, which stays a separate item below.
- [x] **`GuestSeatingObserver`** — sends seat-assigned, seat-changed and check-in notifications
      across all three channels, with bulk-send suppression and previous-seat capture in the same
      UPDATE. Needs a `guests.previous_seat` column. pep `c941802`. *(Large.)* **Done** — `P033.58`,
      backend `85c54ff`, with `GuestSeatingObserverTest` (15 tests) + an end-to-end drain test.
      ✅ **IT SHIPS INERT — the finding that changes the risk picture.** Our categories know only
      four events (`on_accept`, `on_register`, `on_register_complete`, `on_reject`); the three
      seating events are **new**, so `getNotificationTemplate` returns `null` on every channel and
      **nothing dispatches until an admin configures them** — i.e. until the §7 `categories-form.tsx`
      item lands. The "bulk import messages 400 guests" failure mode cannot happen on day one.
      **Reach traced, not assumed:** `checked_in_at` has exactly two writers — `attend()` (live scan,
      *should* notify) and `guestsSyncOffline` (drain, suppressed). The Excel import calls
      `Guest::create`, and the observer hooks `updating`/`updated` only, so creates cannot trigger it
      (pinned by a test). `Guest::query()->update(...)` bypasses Eloquent events entirely.
      **pep's diff did not apply** — they wrap `guestsSyncOffline`, which `P033.43` had already
      rewritten to fix the FK bug, so the concept was applied to our version.
      Both suppression paths are **proven**: removing the wrapper fails the drain test; suppression
      survives a throwing callback and nests without an inner call re-enabling an outer one.
      Beyond pep — whose observer has **no test at all** — the suite covers the "must not fire" half:
      inert by default, master channel gate, clearing a seat, editing an unrelated field, a second
      check-in, and creating an already-seated guest.
- [x] **Per-admin filter visibility.** One button on the filter row opens a picker; each admin hides
      the filters they never use. Stored on `admins.preferences`, keyed by route, so it follows them
      to another device. Saves *hidden* keys, so filters added later stay visible by default.
      **DONE IN FULL** — Part A `P033.60` (admin `d1ffa7b`): the `use-filter-visibility` hook and the
      `ListingFilters` picker, covering all **45** generic listings (not 44). Part B `P033.65`
      (admin `b3d9483`): `FilterVisibilityProvider` / `FilterSlot` plus self-registration of the
      guests listing's **21** hand-written filters, sharing the same hook, storage and route key.
      **The 977-line Part B diff is re-indentation, not rewriting** — our file was byte-identical to
      pep's pre-image except 8 lines, and those 8 were the dinner params pep had already removed. All
      11 imports were verified to resolve before copying.
      **Three departures from pep, all deliberate:**
      **(1)** our `ListingFilters` was pep's pre-image *plus* the `modal={false}` scroll-lock fix from
      `P033.12`; taking their post-image silently reverts it. Re-applied and verified.
      **(2)** `resetFilterPreferencesCache` is **exported but never called anywhere in pep's HEAD**,
      so in their build one admin inherits another's hidden-filter choices on a shared tab until a
      reload. Wired into our `logout()`.
      **(3)** `data/feature-flags.ts` from the same commit was not ported — a `SOCIAL_POSTER_ENABLED`
      flag for an unrelated client request, and against `CLAUDE.md` rule 3.
      🔎 **Side-effect on `LEFTOVERS §2.1`:** Part B removed the dead `dinner_invite` /
      `with_dinner_badge` params **from the filter form** (verified dead: no `dinner` column in any
      migration, `with_dinner_badge` commented out at `GuestsController:1410`). **The chain is only
      half removed** — `guests-listing.tsx` and `pages/[lang]/guests/index.tsx` still declare and
      forward both. They are now always empty and the backend has no such filter, so it is harmless,
      but it is a half-state left on purpose: §2.1 is parked pending an owner decision. ~8 lines
      across 2 files whenever that lands.
- [x] **`admins.preferences` column + `GET/PUT /admin/me/preferences`.**
      ⚠️ **pep's routes have no `auth.admin` middleware**, so a mobile guest token can reach an
      admin write. Add it when porting. **Done** — `P033.57`, backend `61a62a6`, with
      `AdminPreferencesTest` (7 tests).
      **The warning was correct and the guard was added.** Confirmed the exposure is real, not
      theoretical: `Guest` uses `HasApiTokens` and the mobile app calls `createToken('mobile', …)`,
      so every registered guest holds a token that satisfies our plain `auth:sanctum` group. Both
      routes are wrapped in `auth.admin`, and **the guard is proven to be what blocks them** —
      removing it fails `a_guest_token_cannot_reach_it`, restoring it passes.
      This follows an existing convention rather than inventing one: `routes/api.php:960` already
      guards the ported mobile-feature modules with `auth.admin` for the same reason.
      **Rule 4 checked:** `routes/api.php` is touched, and `docs/mobile` has no mention of
      `/admin/me` or `preferences` — new additive admin-guarded endpoints, no mobile impact.
      Payload validated only as `['present','array']`: display choices with no security meaning, and
      a fixed schema would mean a backend change per new preference.
      🔎 A near-miss: the scoping test first appeared to show one admin reading another's bag. The
      **database was probed directly before changing any code and was correct** — Laravel memoises
      the resolved guard across requests inside one test method, so the second request reused the
      first identity. Fixed with `forgetGuards()` in the test. The code was never wrong.
      **pep `c941802` is now two-thirds consumed** (`is_vip` `P033.54`, `preferences` here);
      `GuestSeatingObserver` + `guests.previous_seat` remain.
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
- [x] **`withoutNotifications` wrapper** for bulk drains — **Done** as part of `P033.58`
      (`85c54ff`), since shipping the observer without it was the genuinely risky ordering. Static
      flag on `GuestSeatingObserver`, restores the previous state in a `finally` so a throwing
      callback cannot leave notifications off, and nests safely. Applied to `guestsSyncOffline`; the
      import needs no wrapper because it creates rather than updates.
- [x] **WhatsApp: full set — see section 7.** **Done — §7 is now closed in full** (12/12). This was
      a pointer, not work of its own: the last three §7 items landed as `P033.59` (admin `53ea733`),
      unblocked by `P033.58`.
- [x] **`NotificationEventFields`** — **Done** as part of `P033.59` (admin `53ea733`); same
      component as the §7 item, one change closes both.
- [x] **`admins.is_vip` flag** — independent of role. Small migration + form toggle + a live-resolved
      `actor_is_vip` on the seating audit log, highlighted in the seating app's notification panel.
      **Done across all three repos** — backend `P033.54` (`0e8b019`), admin `P033.55` (`19bf242`),
      seating `P033.56` (`793e244`), with `AdminIsVipTest` (4 tests).
      ⚠️ **Unbundled from pep `c941802`, which is the SAME commit as `GuestSeatingObserver` and
      `admins.preferences`** — three separate items in one commit, 15 files, +438, and it touches
      `routes/api.php` for the preferences endpoints. Only the `is_vip` slice was taken; the observer,
      `previous_seat` and `preferences` remain their own items below. The seating half came from
      `8c39930`, the same commit whose `layout.js` shipped as `P033.44` — **that commit is now fully
      accounted for.**
      **`has()`, not `filled()`** on the update branch — deliberately the opposite of `P033.50`: for
      a boolean *body* field `filled()` is false when the value is `false`, which would make the flag
      impossible to turn off.
      **`actor_is_vip` is resolved live, not snapshotted** (pep's design): `actor_name` / `actor_role`
      are frozen so an entry keeps naming the right person after a rename, but VIP is a *current*
      property, so flagging someone lights up work they already did. The test pins exactly that.
      `->with('admin:id,is_vip')` added so the listing does not fire a query per row.
      Two near-misses caught by verifying rather than assuming: `reset(d)` in the admin form would
      have **silently cleared the flag on every save of every admin** had the resource not been
      updated; and the seating classnames were checked in the **built CSS**, because Tailwind purges
      what it cannot statically see and a dynamic classname would have rendered nothing.
- [x] **Seat/table number in the default see-more panel** — present only on the RSVP shape today.
      **Done** — `P033.61`, admin `760d5f6`. **One edit covers two shapes:**
      `fours-steps/step-1.tsx` is a 13-line re-export of `one-step/step-1.tsx`, so the four-step
      shape picks the rows up for free. pep does not touch their `fours-steps` panel, which reads as
      an oversight until you notice it is a re-export in their tree too — a second edit would have
      been redundant, not thorough. All three shapes now show seat and table.
      No new keys: `web:table_number` / `web:seat_number` already existed in EN and AR.
      Kept pep's *"read-only, the Seating Plan Manager owns these columns"* comment, which matters
      more here than in their tree: after `P033.58`/`P033.59` a seat change can fire guest
      notifications, so this panel must not be read as an edit surface.
- [x] **Favicon components reference three assets that are not in `public/`**, while the real ones
      go unreferenced. **Done** — `P033.45`, frontend `97700e7`. `favicon-32x32.png`,
      `favicon-16x16.png` and `safari-pinned-tab.svg` all 404'd; `favicon.svg` and
      `favicon-96x96.png` sat in `public/` unreferenced. Ported **ewc's** approach, not pep's: pep
      and gfeai added the three missing binaries, ewc made the tags follow the assets — and our
      `public/` is filename-identical to ewc's, so ewc's fix needs no new binaries. Took ewc's
      structure only, not its branding. Also relative-ised the `shortcut icon`, which rendered
      `undefined/favicon.ico` under `yarn build` (see `LEFTOVERS §1.2`). Verified in the compiled
      bundle. **Same item as §8 below — one fix closes both.**

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

- [x] **`notification-event-fields.tsx` (new, 128 lines)** — WhatsApp as one of the three channels.
      ~~**BLOCKED on `GuestSeatingObserver` (section 6).**~~ **Unblocked by `P033.58` and done** —
      `P033.59`, admin `53ea733`.
- [x] **`categories-form.tsx` (+118)** — seating events (`on_check_in`, `on_seat_assigned`,
      `on_seat_changed`), each with `whatsapp: { enabled, whatsapp_template_id }`. **Done** —
      `P033.59`, admin `53ea733`.
      ⚠️ **Easy to miss:** pep extends `NOTIFICATION_EVENTS` to all six **and** adds `SEATING_EVENTS`.
      That is not redundancy — `NOTIFICATION_EVENTS` drives the **normaliser**, so adding only
      `SEATING_EVENTS` renders the fields but silently drops them on load and save. Both applied.
      🔴 **This commit is the switch.** Everything in `P033.58` is dormant until a category
      configures one of these events; from here, ticking a box and picking a template sends real
      messages.
- [x] **`interfaces/category.tsx` (+38)** — the types. ~~**BLOCKED on `GuestSeatingObserver`.**~~
      **Done** — `P033.59`, admin `53ea733`. Added as a shared `NotificationEventChannels` alias
      rather than a fourth, fifth and sixth copy of the same inline shape.
      ⚠️ **SCOPE CALL — pep's *current* design was deliberately NOT taken.** pep has two commits on
      this component: `fd41aed` (128 lines, guest-only) and the later `0c7f788` / `5ab0380`, which
      grew an **admin-notifications** half — admin copies per event,
      `Category::getAdminNotificationTemplate()`, per-admin subscriptions, attendance preference,
      guest change requests. The standing instruction for §7 was to port pep's current design; this
      does not, because our backend has no `getAdminNotificationTemplate` and `P033.58` is pep's
      guest-only observer. Porting their HEAD would put admin switches in the UI that **nothing
      could ever act on** — the trap this section's own blocking note raised. **pep's admin
      notification layer remains an unported, separate feature.**

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
- [x] **Two-step form dead end** — see the gfeai frontend list; ewc fixed it too. **Done** —
      `P033.46`, frontend `f4cb529`. Same item as §5; one fix closes both.
- [x] **`DATE_COLS` in the guest export point at the wrong columns** — folded into the export
      mechanism job above. **Done** — `P033.34`, backend `817742c`: `DATE_COLS` is now `M`, `Q`, `X`
      (Printed At · Created At · Reconfirmed At) in both `GuestsExport` and `GuestsExportGuestView`,
      pinned by `ExportColumnAlignmentTest`. Same fix as the §1 export item.
- [x] **Favicon tags reference three assets that do not exist** in the repo. **Done** — `P033.45`,
      frontend `97700e7`. Duplicate of the pep favicon item in §6; ewc's own fix is the one that was
      ported, because ewc's asset set matches ours exactly.

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
- [x] **`composer.lock` is untracked in all four backend repos.** Every clone's `composer install`
      resolves fresh versions. Non-reproducible builds, and no supply-chain pinning — which sits
      awkwardly next to the OWASP hardening this baseline advertises. *(No fork fixes this either;
      noting it as a decision to make.)* **Done for this repo** — `P033.48`, backend `c32fe3b`.
      Removed from `.gitignore` and committed (182 packages). The three clone repos are separate git
      repos outside this baseline and are untouched.
      ⚠️ **An update had to come first, and it inverts the obvious answer:** with no lock,
      `composer install` behaves like `composer update` and resolves fresh — so committing the
      Jul-8 lock as-is would have *downgraded* every future clone into vulnerabilities that fresh
      resolution already avoids. `composer audit` found **17 advisories across 4 packages**; three
      were fixable inside the existing constraints — guzzle `7.12.1→7.15.5` (7), psr7
      `2.12.1→2.13.1` (1), commonmark `2.8.2→2.10.0` (6), plus 4 transitive. **17 → 3.**
      `composer.json` is byte-unchanged, so this is minor-within-major per `CLAUDE.md` rule 6; no
      constraint was widened. Note `composer outdated` misleads here — it reported guzzle as
      `~ 8.1.0` and never revealed 7.15.5 inside `^7.12.0`; only the audit's affected-version ranges
      were reliable. Lock checked for credentials before committing: all dist URLs on
      `api.github.com`, no private sources, no token-like strings.
      **Consistency:** admin/frontend already track `yarn.lock` and seating `package-lock.json` —
      the backend was the only one ignoring its own.
- [ ] **Bump `phpoffice/phpspreadsheet` off the 1.x line — 3 high advisories, needs its own branch.**
      Left behind by `P033.48`. `1.30.5` is the ceiling of `^1.30.5` and the whole 1.x line is
      affected: SSRF via an HTTP redirect bypassing the `WEBSERVICE()` domain whitelist
      (`CVE-2026-59931`), plus two memory-exhaustion advisories. Fixes start at **2.1.18**, a major
      bump — gated by `CLAUDE.md` rule 6. It is also pulled by `maatwebsite/excel ^3.1`, so the bump
      likely drags Excel with it, and **28 classes in `app/Exports/` ride on that** (see the export
      column-letters item in §1 and `LEFTOVERS §2.9`). Sequence it after, or together with, any
      export work.
      🔴 **BLOCKED ON A PHP RUNTIME UPGRADE — established 2026-08-25, proven by Composer's resolver,
      not inferred.** The chain dead-ends:
      `phpspreadsheet 1.30.5` (where we are, whole 1.x line affected) is pinned by
      `maatwebsite/excel 3.1.70` — **the newest 3.x, which still requires `phpspreadsheet ^1.30.5`**.
      The only line that allows the patched `phpspreadsheet ^5.8` is `maatwebsite/excel 4.0.x`, and
      **that requires `php ^8.3`** against our runtime of **8.2.26** (`composer.json` requires `^8.2`).
      Composer's verdict verbatim: *"maatwebsite/excel[4.0.0, ..., 4.0.2] require php ^8.3 -> your
      php version (8.2.26) does not satisfy that requirement."* **There is no Excel release that
      reaches the patched phpspreadsheet on PHP 8.2.**
      **So this is not a dependency bump — it is a PHP 8.2 → 8.3 runtime upgrade**, gated by
      `CLAUDE.md` rule 6 by name and changing the production server, not just the lockfile. The only
      alternative is dropping `maatwebsite/excel` and driving `phpspreadsheet` directly, which is a
      rewrite of **28 export classes**. Both are materially bigger than this item describes and need
      an owner decision.
      A branch (`task-033-phpspreadsheet`) was created, the resolution proven, `composer.json`
      restored unchanged, and the branch deleted — nothing was left behind.
      ⏸️ **PARKED BY THE OWNER, 2026-08-25 — waiting on a PHP 8.3+ upgrade of the dev machine, not
      abandoned.** Revisit once `php -v` reports 8.3 or later.
      **Recipe when that lands** (all of this was verified, only the PHP version blocked it):
      1. branch off `dev` — rule 6 requires a dedicated branch for a PHP-major move;
      2. raise `composer.json` `"php"` to `^8.3`;
      3. set `phpoffice/phpspreadsheet` to `^5.8` and `maatwebsite/excel` to `^4.0`;
      4. `composer update phpoffice/phpspreadsheet maatwebsite/excel --with-all-dependencies`;
      5. `composer audit` — expect **3 advisories → 0**, closing `CVE-2026-59931` (SSRF via an HTTP
         redirect bypassing the `WEBSERVICE()` domain whitelist) plus two memory-exhaustion highs;
      6. run `composer qa`. **The blast radius is the 28 classes in `app/Exports/`** — Excel 3.x→4.x
         is a major, so expect signature changes. `ExportColumnAlignmentTest` and
         `EmailTemplateCloneTest` (which asserts a real workbook is produced) are the first things
         that will catch breakage.
      ⚠️ **Also check the deploy target's PHP, not just the dev machine** — the runtime has to move
      too, and `composer.json` requiring `^8.3` will refuse to install on an 8.2 server.
- [ ] **Admin, frontend and seating have zero automated tests, and there is no CI in any of the 18
      repos.** The backend has 24 feature test files. Most of the bugs in this document are
      client-side, where we have no coverage at all.
      ⛔ **GitHub Actions CI declined by the owner, 2026-08-25.** Workflows were written for all four
      repos and then removed unbuilt — do not re-propose them. Recorded because the groundwork is
      real and cheap to redo if the decision changes: all four repos are on GitHub with **no CI**,
      all four now carry lockfiles (`composer.lock` since `P033.48`), the backend suite needs **no
      database service** (`phpunit.xml` pins sqlite `:memory:`), and `.env.example` +
      `key:generate` is all the environment it wants. The three backend gate commands were verified
      to pass verbatim from the command line — `pint --test`, `phpstan analyse --no-progress`,
      `php artisan test` (607 passing).
      **The other half of this item still stands and is untouched by that decision:** admin,
      frontend and seating have **zero tests of their own**. Whatever runs them, there is nothing
      to run.

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
