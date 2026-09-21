# Decision Ledger

Append-only record of **durable, cross-task locked decisions** for alt-static-basecode. Numbered
`D1…Dn`, never renumbered. A task-local choice lives in that task's `TASK.md`; a decision that
outlives the task gets promoted here. This is the source of truth when a decision and the code/docs disagree.

> Format: `D<n> — <date> — <one-line decision>` + a short why + where the detail lives.

---

## D1 — 2026-06-21 — cloned from pif-directors-gathering baseline

`alt-static-basecode` was cloned from the `pif-directors-gathering` upgrade-then-clone baseline
(source SHAs: backend `8745150` · admin `0f65026` · frontend `5e3fee2` · landing `55fd6b7` · docs `3cbca75`).
Inherited stack: **Next 15 / React 18.3.1 / Headless UI v2 / Tailwind v4 / Laravel 12** (Sentry removed,
OWASP-hardened). The frozen lineage record under `../upgrades/` documents what this baseline already includes.

- **What the baseline already includes:** [../upgrades/UPGRADE_SUMMARY.md](../upgrades/UPGRADE_SUMMARY.md)
- **Why directors was the chosen baseline:** [../upgrades/BASELINE_DECISION.md](../upgrades/BASELINE_DECISION.md)

## D2 — 2026-07-05 — dropped the `-landing` app

The directors baseline carried a 4th sub-app, `alt-static-basecode-landing` (Next 15 marketing site).
It is **not needed for this project** and has been dropped — `alt-static-basecode` is now a **three
sub-app** baseline (`-backend`, `-admin`, `-frontend`) plus `docs/`. Current-state docs were updated to
match; the frozen lineage under `../upgrades/` (and D1's clone-source SHAs) still mention `-landing` as
an accurate record of the baseline it came from.

## D3 — 2026-07-06 — unified admin forgot-password onto the invite reset-by-token flow

Admin "forgot password" now reuses the **same token machinery as the admin invite** instead of the
legacy `v2/password/forgot` guest endpoint. `AuthController::forgotPassword()` issues a one-per-email
`AdminInvite` token, emails the shared `emails.admin_invite.{en,ar}` blade (different subject only) via
a public `POST /admin/forgot-password`; the admin `forgot-password-form` posts `{ email, back_link }`
there, landing the admin on the same `reset-password/[token]` page as invites. This brings alt to
**cyan parity** on the reset flow, with **one deliberate deviation**: the endpoint is **enumeration-safe**
— it always returns success and only sends mail if the admin exists (cyan validates `exists:admins,email`,
which leaks account existence). The new route is **admin-only and additive**, so the mobile contract is
unaffected. Backend `a8184ca` / admin `70e646c` (both merged via PR #1). Detail: `HANDOFF.md`.

## D4 — 2026-07-06 — backend adopts the `dev` → PR → `main` workflow

The backend repo historically committed straight to `main` (it had no `dev` branch, diverging from
admin/frontend). It now uses **`dev`** as its working branch and merges to `main` via PR, matching the
other two app repos and CLAUDE.md's "work on `dev`, never push to `main`" rule. This resolves the
"backend branch convention" open item from the prior handoff. (`docs/` remains `main`-only — it is a
docs-only sibling repo with no app build/branch flow.)

## D5 — 2026-07-06 — lucide-react is the single icon library (dropped @iconify/react + @heroicons/react)

Both Next apps carried **two** icon libraries from the baseline (`@iconify/react` and
`@heroicons/react`). They are now unified on **`lucide-react`** and both old deps have been removed
from `package.json` + `yarn.lock`. The migration ran in two phases against a machine-verified
heroicon/iconify → lucide name map (P1: iconify removal incl. the `bi:tiktok` inline-SVG replacement;
P2: heroicons → lucide). All icon swaps are 1-for-1 (`className` sizing preserved), so this is a
zero-behaviour-change refactor — no icon is used by the mobile contract (admin/frontend only). Gates
green on both apps (`type-check` + `production`). Admin `de87b4b` (P2) / frontend `c74b82c` (P2),
stacked on the P1 lucide commits. Detail: `HANDOFF.md`.

## D6 — 2026-07-06 — real booleans over string pseudo-booleans; `status` → `is_active`

The baseline stored many flags as strings: `yes`/`null` and `yes`/`no` toggles (`is_saudi`,
`with_share`, `prefilldata`, …) and 2-value `status` (`active`/`blocked`) columns. These are being
converted to **real booleans**. Two tracks: **(A)** the `yes`/`no`/`null` + `with_`/`is_` flags —
this mirrors the **cyan** reference, which already did and documented it
(`115-cyan-basecode/.../docs_old/BOOLEAN_REFACTOR_*.md`); **(B)** a **new** conversion cyan never
attempted — strictly 2-value `status` columns become **`is_active boolean default(true)`**.
Multi-value process statuses (`app_notifications` Pending, `login_attempts` failed,
`email_attachments`, `sms_templates` send-state, guest workflow `guest_status_id`, mobile guest
`accepted`/`invited`) are **excluded**. Migrations are edited **in place** + `migrate:fresh` (no prod
data). The `/api/mobile/*` contract is unaffected (storage/internal only; mobile already sends
booleans and its speaker/sponsor resources don't expose `status`). Plan +
detail: [../upgrades/cleanup-hardening/BOOLEAN_REFACTOR_PLAN.md](../upgrades/cleanup-hardening/BOOLEAN_REFACTOR_PLAN.md) /
[../tasks/001-boolean-db-cleanup/TASK.md](../tasks/001-boolean-db-cleanup/TASK.md).

## D7 — 2026-07-07 — real date/time column types + cyan masked date input (no more Unix-string dates)

Dates/times were stored as **Unix-epoch strings** (day-picker fields: `getUnixTime()` +
`zonedTimeToUtc(…)/1000`) mixed with `Y-m-d H:i:s` datetime strings (`Carbon::now()` fields) in the
**same** `string` columns. Converted to proper types: date-only → **`date`** (cast `date:Y-m-d`,
API/forms `YYYY-MM-DD`), datetime → **`timestamp`** (cast `datetime`, ISO 8601 UTC), time-only kept
as **`string(5)` `HH:mm`** (matches cyan `meeting_room_slots`). Columns edited **in place** +
`migrate:fresh` (no prod data; same playbook as D6). FE/admin drop the react-day-picker modal for
cyan's **Cleave masked inputs** (`masked-date/-time/-datetime-input.tsx` + shared `utils/date.ts`
`formatDate`/`formatDateTime`); TZ conversion happens **at display** (`Asia/Riyadh`) so users outside
KSA are handled correctly. **`custom-day-input.tsx` is kept** (unused, for future custom cases) and —
revising the original plan — **`timezoneFix.ts` is kept too** because it is that component's only
remaining dependency (matches cyan). Mobile contract unaffected: guest dates live only in **Admin**
resources; the offline QR `check_in_time` write round-trips through the new `datetime` cast and
`routes/api.php` is unchanged. Gates green: backend `pint --dirty --test` + `migrate:fresh` +
`php artisan test`; both Next apps `type-check` + `production`. Detail:
[../tasks/002-datetime-db-cleanup/TASK.md](../tasks/002-datetime-db-cleanup/TASK.md).

## D8 — 2026-07-07 — agenda `date` is venue-local wall-clock (mobile contract change)

Follow-on to D7. Session/Workshop `date` (a `dateTime` column, cast `datetime`) was serialized with
`->toISOString()` (UTC `…Z`) while being **entered** via a native `datetime-local` (naive wall-clock,
no TZ). Because the API emitted UTC, the admin edit form's `data.date.slice(0,16)` pre-filled the
input with the **UTC** clock (`11:30` for a `14:30` Riyadh event), so opening a session/workshop and
saving **silently shifted the time −3h** (the Riyadh offset). **Decision (mirrors cyan, which never
UTC-converts these): treat agenda `date` as venue-local (Asia/Riyadh) wall-clock end-to-end.** All 9
resources (`AdminSessionsResources`, `AdminWorkshopResource`, Mobile `SessionList/Detail`,
`FavoriteSession`, `WorkshopList/Detail`, `RegisteredWorkshop`, nested in `SpeakerResource`) now emit
`->format('Y-m-d\TH:i:s')` (naive, **no `Z`**). No FE code change needed: the admin `.slice(0,16)`
pre-fill and `format(new Date(date))` display both become correct for every viewer (a `Z`-less string
parses as local, and format renders in local — the shift cancels). The `datetime` cast + naive
`datetime-local` submit already round-trip through the app's `Asia/Riyadh` timezone. **Mobile contract
change (flagged in `docs/mobile/BACKEND_INCOMING_CHANGES_FOR_MOBILE.html` §24):** mobile must parse
`date` as wall-clock and NOT convert to the device timezone. Audit timestamps
(`created_at`/`updated_at`, `check_in_time`, etc.) stay UTC ISO 8601 — only the scheduled agenda
`date` is venue-local. Gates: backend `pint --dirty --test` passed; isolated `SessionsTest` /
`WorkshopFeedbackTest` green (the one combined-filter search failure is a pre-existing test-isolation
flake, passes in isolation and cannot be affected by a serialization-format change). Detail:
[../tasks/002-datetime-db-cleanup/TASK.md](../tasks/002-datetime-db-cleanup/TASK.md).

## D9 — 2026-07-07 — caught errors are typed `unknown`, read via the `getApiError` helper

Closes the "cheap cleanups" phase. Every `catch (error: any)` in both Next apps (94 blocks across 82
files) was changed to `catch (error: unknown)`. Bodies that read the axios error now go through a new
shared helper **`utils/api-error.ts`** — `getApiError(error: unknown): ApiErrorResponse | undefined`
(returns `error.response` when `isAxiosError`, else `undefined`) — so `error?.response?.data?.*` reads
become `getApiError(error)?.data?.*` with real types instead of `any`. Log-only bodies
(`console.error(error)` / `throw error`) are simply re-annotated to `unknown`. This satisfies
CLAUDE.md's "no widening to `any`" rule and gives one canonical place to evolve the API-error shape.
**Going forward, new code must use `catch (error: unknown)` + `getApiError`, never `catch (e: any)`.**
`@typescript-eslint/no-explicit-any` stays **off** globally (unchanged) — this was a targeted burn-down,
not a rule flip. Commits: admin `5ceacc3`, frontend `8544c39` (both pushed; on `dev` **and** `main` since 2026-08-06).
Gates green: `type-check` + lint 0 warnings, both apps.

## D10 — 2026-07-08 — backend gate is `pint --test` (repo is Pint-clean); backend tooling chain adopted

The backend was **not** Pint-clean at baseline, so the gate had been the workaround `pint --dirty`
(format only changed files; a repo-wide run churned ~170+ unrelated files). Task 003 (backend tooling
chain) closes that: an explicit **`pint.json`** (laravel preset + `no_unused_imports` + `ordered_imports`)
plus **one repo-wide Pint baseline** (commit `96413df`, 172 files, formatting-only) made the repo
Pint-clean. **Decision: the backend format gate is now the full `pint --test` (`composer lint`), not
`pint --dirty`.** Alongside it the chain added: **Larastan** static analysis at **level 0** (`bb61db9`)
with a generated baseline (124 real structural errors) and a **committed ratchet** (shrink to zero →
bump the level toward 6; runs in `composer analyse`/`qa`, never in the hook); **composer QA scripts**
(`lint`/`lint:fix`/`analyse`/`test`/`qa`); a **PHP-native pre-commit hook** (`.githooks/pre-commit`,
`9741e90`) that runs Pint on staged `*.php` and graceful-skips if Pint is absent (parity with the
admin/FE husky+lint-staged hook); and `.vscode/` **un-ignored + committed** with `[php]`→Pint
(`de75eed`) — fixing an inconsistency where admin/FE tracked `.vscode/` but the backend gitignored it.
Rector + CI were considered and left out (Rector = optional one-off in the cleanup plan; no repo has CI).
**Going forward the backend quality gate is `composer qa` = `pint --test` + `phpstan analyse` +
`php artisan test`.** Detail: [../tasks/003-backend-tooling-chain/TASK.md](../tasks/003-backend-tooling-chain/TASK.md).

## D12 — 2026-07-12 — admin bearer token is HttpOnly via a Next BFF proxy (Saudi P2 backport)

The admin stored its Sanctum bearer in a **JS-readable** `token` cookie (read by `cookie.get('token')`
across ~136 client sites), so any XSS in the admin could exfiltrate it → full account takeover. The
Phase-1 fix (`af2298b`, `secure`+`sameSite:'lax'`) hardened transport/CSRF but **could not** close the
XSS-read vector — `httpOnly` can only be set server-side. **Decision: adopt Saudi Forum 11's Point 2 —
the token lives ONLY in an HttpOnly cookie set by a Next.js BFF proxy; the browser never handles it.**
Un-defers Task 004 from Track B of `CLEANUP_AND_HARDENING_MASTER_PLAN.md` (user, 2026-07-12) — done now
because the basecode is **pre-launch with no clone carrying prod data**, so the large 135-file codemod is
cheap to bake into every future clone. What landed: `utils/{auth-cookies.ts,server/proxy.ts}` +
`pages/api/{proxy/[...path],auth/{login,login-confirmation,logout}}.ts`; isomorphic `utils/axios.ts`
(browser→`/api/proxy`, SSR→direct Laravel); `auth/provider.tsx`+`withAuth.tsx`+login/verify forms moved
onto a JS-readable **flag cookie** (`alt_admin_auth`) + an `authenticated` marker; a codemod removing all
dead `cookie.get('token')` reads (136) and `Authorization: Bearer` headers (261); and a **full CSP** in
`next.config.js` adapted to alt (env-derived origins, reCAPTCHA only — **no iconify** per D5, no Maps/Fonts,
`blob:` for print-js, `'unsafe-eval'` dev-only). Cookie is `HttpOnly; SameSite=Strict; Secure(prod);
Max-Age=6h`. **Not mobile-facing** — admin-web only, `routes/api.php` untouched, mobile keeps its own
`MobileAuthController` flow. Gates green (`type-check` + `production`); BFF behavior verified at runtime
(login strips token + sets HttpOnly cookie, proxy injects Bearer from the cookie, logout clears both, OTP
+ streaming + CSP header all confirmed). **Real-env browser QA (live backend login, reCAPTCHA, heavy
export through the proxy) is still outstanding before `dev`→`main`** — Saudi hotfix-reverted their P2 once
over the streaming/CSP edges. Detail: [../tasks/005-admin-httponly-token/TASK.md](../tasks/005-admin-httponly-token/TASK.md)
(folder `005`; implements the plan's "Task 004 / Track B").

## D13 — 2026-07-12 — fix TitleSeeder null → `migrate:fresh --seed` restored

`php artisan migrate:fresh --seed` was **broken on `dev`**: 6 `TitleSeeder` rows passed
`show_in_user_form => null`, but that column is a NOT NULL `boolean default(false)` (D6 boolean refactor)
and is cast to boolean on the model — so the insert threw `SQLSTATE[23000] Column 'show_in_user_form'
cannot be null` and seeding died at Title (Country/Admin/GuestStatus/Category ran first). This was the
pre-existing bug the dropped migration-squash recon surfaced (see `tasks/004-migration-squash/TASK.md`
closing note); tasks 001/002 gates had run `migrate:fresh` **without** `--seed`, so it went unnoticed.
**Decision: fix the seeder, not the schema** — `null` meant "not shown", which for a NOT-NULL boolean is
`false`; the column + cast are correct as designed. Changed the 6 `null` → `false`. Verified: full
`migrate:fresh --seed` clean, all 8 seed-path seeders green, 12 titles seed (6 shown / 6 hidden / 0 null),
Pint + Title tests pass. Backend `dev` `a6fe3d1`. Not mobile-facing (seed data only). No other seeder in
the run path had the same null-into-bool pattern.

## D14 — 2026-07-12 — sensitive registrant files are private + served via signed URLs (Saudi P1 backport)

Registrant PII files — `guests.personal_image` (photos) and `guests.document_copy` (passport/ID copies)
— were written to Laravel's **`public` disk** and handed out as **raw, unauthenticated, CDN-cacheable
URLs** (`PUBLIC_STORAGE_URL/personal_image/…`), so anyone with or guessing a URL could fetch someone's
passport. **Decision: adopt Saudi Forum 11's Point 1 — sensitive files live on a `private` disk
(`storage/app/private`, never web-served) and are delivered only through short-lived signed URLs.**
Un-defers the plan's Task 005 / Track B (user, 2026-07-12); done now because the basecode is pre-launch
with no clone carrying prod data (squash/migration of live files would otherwise complicate it). What
landed: `private` disk in `filesystems.php`; new `GuestDocumentController` (`signedUrl()` +
signature-validated `stream()` with a `basename()` traversal guard, type allow-list, `Cache-Control:
no-store`); a `signed`-middleware route `GET /api/files/guest-doc/{type}/{file}` (`guest.doc.stream`);
uploads (incl. the **mobile self-service avatar** in `MobileAuthController`) repointed to `private`;
admin resource URLs → signed (30-min TTL) and **mobile `avatar` → signed (24-h TTL)**; 16 server-side
read-backs (badges/PDF/social-card/email-photo across 7 files) repointed from
`base_path('public/storage/…')` to `Storage::disk('private')->path(…)`; an idempotent
`guests:migrate-docs-to-private --dry-run` command (no-op on fresh clones, kept for clones with data +
a CDN-purge reminder); 2 stale phpstan-baseline `env()` ignores removed. **Dual TTL by decision:** admin
30 min vs mobile 24 h so app caches survive a session. **MOBILE CONTRACT CHANGE** — `avatar` is now a
signed, expiring URL (same field/type, bounded lifetime); mobile must re-fetch after expiry, not build
the URL itself → flagged in `docs/mobile/MOBILE_NOTICE_PRIVATE_AVATAR_SIGNED_URL.md`; **hold `dev`→`main`
until mobile acks.** Public assets (sponsor/speaker/badge/media/publication/attachment images) stay
public by design. Gates: `composer qa` green (`pint --test` + phpstan **No errors** + tests **452/3
pre-existing** — the 2 avatar failures confirmed to fail on the clean parent too, so no regression);
`migrate:fresh --seed` clean; runtime verified (private file streams 200 via valid signature, 403 on
tamper/no-sig, 404 on traversal, 404 at the old public path). Detail:
[../tasks/006-private-document-storage/TASK.md](../tasks/006-private-document-storage/TASK.md).

## D15 — 2026-07-13 — fix two Tailwind v4 regressions (error focus-ring + rtl:space-x-reverse)

Backport of Saudi Forum 11's `docs/upgrades/FIX_TAILWIND_V4_REGRESSIONS.md` — two v4-upgrade regressions
that alt's own v3→v4 pass missed (alt did the *normal* focus-ring restore, ledger D-era `TAILWIND_V4_CLEANUP_PLAN`,
but not these). Both className-only, no logic change. **Fix 1 — error focus-ring → `/50`:** v4 dropped
`ring-opacity-*`/`--tw-ring-opacity`, so a bare `focus:ring-red-500` / `focus-within:ring-error` on
error-state inputs rendered at full opacity (harsh solid ring) vs the soft `/50` normal ring. Carried the
opacity on the color class (`…/50`) — admin 11 files (commit `5d99b43`), frontend 5 files (`052f16f`).
Decorative rings that already had `/50` (e.g. the PIF step-2 Trash2 button) intentionally left alone.
**Fix 2 — drop `rtl:space-x-reverse`:** v4 rewrote `space-x-*` to logical properties
(`margin-inline-start/end`) that already flip under `dir="rtl"`, so `rtl:space-x-reverse` now *double*-flips
— RTL horizontal spacing landed on the wrong side. Deleted it everywhere (nothing replaces it) — admin 124
occ / 69 files (`aadadf8`), frontend 25 occ / 9 files (`721d458`). **Method note:** the removal is a
className-only token delete; an early scripted attempt corrupted indentation via a global whitespace
collapse and was fully reverted before redoing it surgically (token + one adjacent space only). Gates:
`type-check` + `production` green on both apps. Not backend/mobile-facing. **Visual EN/AR QA still pending**
(soft red ring on invalid inputs; RTL spacing on checkboxes/radios/back+share buttons/toolbars) — the one
thing automation can't confirm.

## D16 — 2026-07-13 — consolidate storage-URL env vars to a single source of truth

Collapsed the redundant/inconsistent storage-URL env vars across all three apps onto **framework-native
hosts**. Plan + full detail: [../upgrades/STORAGE_URL_CONSOLIDATION_PLAN.md](../upgrades/STORAGE_URL_CONSOLIDATION_PLAN.md).
Pre-launch (no prod data / CDN cache / shipped mobile), so the aggressive path — drop vars, change values,
no byte-identical-URL guarantee needed — was fine; the emitted URLs came out identical anyway.

**End state:** backend keeps **zero** `PUBLIC_STORAGE_URL*` vars (public URLs via
`Storage::disk('public')->url()` → `config/filesystems` → `APP_URL.'/storage'`); admin keeps **one**
(`NEXT_PUBLIC_STORAGE_URL` = storage **root**) + `utils/storage.ts` deriving `/uploads`, `/attachments`,
`/{module}` paths; frontend keeps **zero** (its var was comment-only dead). Sensitive registrant files stay
on the private disk + signed `*_url` (D14) — never rebuilt from an env var.

**Landed (pushed; on `dev` and `main` since 2026-08-06):** FE `89c1ce3`; admin `b5bb5b2`→`9137fd9`→`fd628cd` (+ tracked `.env.example`);
backend `58ca08c` (new public `social_card_image_url`) + `5cebb86` (46 `env('PUBLIC_STORAGE_URL2')` sites /
28 files, incl. 7 mobile resources, → `Storage::disk('public')->url()`; self-healing sites →
`rtrim(...url(''),'/')`; phpstan baseline pruned 45→18 env ignores, masking nothing). Byte-identical output
verified via tinker (bare + leading-slash) → **mobile contract unchanged, no ack needed.**

**Fixed en route:** the `social_card` see-more URL had a wrong path (`/social_card` vs `/uploads/social_card`)
— now correct from the API. `visa_copy`/`issued_visa` confirmed **dead** (no column; upload allow-lists
`document_copy` only) so no URL added. **Deliberately deferred (separate correctness item, not env-var work):**
guest `custom-file-input{,-3}.tsx` still rebuild a public `/storage/uploads/{field}/` URL for files that now
live on the **private** disk (the `/upload` endpoints return `data` only) — likely a broken preview; fix by
returning a signed `url` from those endpoints. Tracked in the plan's follow-up.

**Gates:** FE + admin `type-check` + `production` green; backend `composer qa` green (`pint --test` + phpstan
**No errors** + tests **452 pass / 3 fail** — the same pre-existing ExampleTest-403 + 2 avatar failures,
confirmed identical on the stashed parent → no regression) + `migrate:fresh --seed` clean. **User owns the
gitignored `.env` edits** (drop the retired vars; retarget `NEXT_PUBLIC_STORAGE_URL` to the root).

> **Addendum 2026-07-17 — three corrections to the above. All pushed now (the "unpushed" note is stale).**
>
> 1. **The refactor missed the Blade email templates.** `5cebb86` swept `app/` only, so **22
>    `env('PUBLIC_STORAGE_URL2')` refs across 8 email views survived** (`otp/{en,ar}`,
>    `notify_guest/{en,ar}`, `password_reset/{en,ar}`, `partials/{header,footer}-poster`). When the var was
>    then dropped from `.env` (`4dfdca9`), `env()` returned null and **every email poster URL rendered as
>    `nullemails-config/…` — broken banners in all outgoing mail**. Repointed to
>    `\Storage::disk('public')->url()` in **`a9a1ed4`**; byte-identical, views compile (`view:cache`).
>    Lesson: sweep `resources/` as well as `app/` when retiring an env var.
> 2. **"Zero refs" only became true later.** `c1606a5` removed the last dead commented
>    `PUBLIC_STORAGE_URL` handlers from `GuestsExportView`. Verified zero across
>    `app/ config/ routes/ resources/ database/`.
> 3. **"`visa_copy`/`issued_visa` confirmed dead" is superseded by D18** — they were never dead, just
>    unfinished. They now have columns and full persistence.

## D17 — 2026-07-17 — one tracked env template per app: `.env.example_prod`

Unified how env templates are named, tracked and shaped across all three apps, so a clone gets a complete,
accurate template for free instead of hand-carrying files.

**Convention:** each app tracks **`.env.example_prod`** — a production-shaped template holding only
**live** vars, placeholder values, **no secrets**. `.env` / `.env.local` / `.env.production` stay gitignored
and are carried/created per environment (DevOps creates `.env.production` on deploy; its absence is why
`yarn production` can't run locally — expected, not a fault).

- **Backend:** `.env.example2` → **`.env.example_prod`** (`c7dd2ee`), retired `PUBLIC_STORAGE_URL*` +
  `APP_DEBUG=false` (`4dfdca9`). `.env.example` stays the Laravel-stock file. **Audited all 60 vars against
  Laravel 12.62: nothing unsupported.** Note the app deliberately runs on the **old** var names
  (`CACHE_DRIVER`, `BROADCAST_DRIVER`) because `config/` still reads those — do **not** "modernise" to L12's
  `CACHE_STORE`/`BROADCAST_CONNECTION` without changing `config/` too, or they'll be silently ignored.
- **Admin:** `.env.example` → **`.env.example_prod`** (`a5e83e6`), `NEXT_PUBLIC_ENV` moved to the top
  (`e79e537`), structure unified with `.env.local` (`6f5ecdc`, `0dcf74a`). Down to the 6 live vars.
- **Frontend:** had **no** tracked template at all — added one (`27bad95`, `a22bd5b`) documenting the 5 live
  vars.
- **Cookie-age vars → code constants** (frontend parity with admin `a361586`): `NEXT_PUBLIC_LANG_COOKIES_AGE`
  (`7248f39`), `NEXT_PUBLIC_{,REMEMBER_}TOKEN_COOKIES_AGE` (`b5df5d3`, doc comment `220e65e`). Identical in
  every environment, so they're `LANG_COOKIE_AGE_DAYS=30` / `TOKEN_COOKIE_AGE_DAYS=1` /
  `REMEMBER_TOKEN_COOKIE_AGE_DAYS=7` in code now, not env.
- **`CLONE_CHECKLIST` corrected** (docs `2b118d5`): it claimed the prod template was gitignored and must be
  carried by hand — it's tracked (placeholders only), so only `.env` itself is carried.

**Known gap (accepted):** frontend `_app`/`_document` read `NEXT_PUBLIC_GTM`, but the old env files set
`NEXT_PUBLIC_GOOGLE_TAG_MANAGER` — a name mismatch, so **GTM has been dormant, never firing**. The template
now documents the correct name; wiring it up (or removing the GTM blocks) is a separate task.

## D18 — 2026-07-17 — guest document + day fields are first-class in the basecode

Three guest fields had UI but no working data layer. Completed all of them rather than leave half-built
features that fail silently or 500.

- **`visa_copy` / `issued_visa` (`00fe02a` → `4883f9d`):** the public join form has `visa_copy`; the **admin
  guest create/edit forms have both** as real uploads. But `/upload-document` allow-listed only
  `document_copy` (→ **422 "failed to verify path name"**), and there was **no column, no `$fillable`, no
  save** — an uploaded visa hit the private disk and was then **silently dropped**. Widened both allow-lists
  + `GuestDocumentController::TYPES`, added the columns (mirroring `document_copy`), wired `store()` /
  `storeGuest()` / `updateGuest()`, and emit signed `*_url` (private disk, D14).
- **`days` (`ef218f6`):** was a **phantom** — `$fillable` + an index() `JSON_CONTAINS(days,…)` filter + an
  export column + a dashboard breakdown, but **no clone except `98-pif-2026` ever added the column**.
  `GET /admin/guests?days=X` returned **500** (`Unknown column 'days'`). `DashboardStats`/`GuestsExportView`
  were already guarded (`Schema::hasColumn` / ternary) — which is what revealed the field was *intentionally
  opt-in per clone*. **Decision: ship the column in the basecode** (`json` nullable, mirroring pif-2026's
  migration) so every clone gets it. **Still has no writer** — all 3 write sites stay commented and no UI
  submits it, so it reads NULL until a clone wires them up.
- **`json_decode(null)` guards (`ef218f6`):** `days` **and** `interests` were unguarded → a PHP 8.2
  deprecation on *every* guest response. The migration alone does **not** fix this — the deprecation fires on
  the null **value**, not the missing column, and both fields are nullable. Guarded with the ternary already
  used in `GuestsExportView:92`; output unchanged.
- **Factories repaired (`9042919`):** `Guest::factory()` threw `Unknown column 'status'` — it still set the
  free-text `status` column retired in the guest-status refactor (now the `guest_status_id` FK). Nothing
  caught it because the feedback tests use `Guest::create()` directly. Also wired
  `category_id => Category::factory()` (**new `CategoryFactory`**) since `category_id` is NOT NULL and always
  was — so the factory could never stand alone despite three sibling factories declaring
  `'guest_id' => Guest::factory()`.

**Gates (all):** `pint` + phpstan **No errors** + `migrate:fresh --seed` clean + tests **452 pass / 3 fail**
(same pre-existing ExampleTest-403 + 2 avatar). **Not mobile-facing** — `routes/api.php` untouched.

## D19 — 2026-07-18 — guest-drafts: capture abandoned registrations (own RBAC feature)

The admin carried a `guest-drafts` UI across clones but its **backend was never built** (no clone had it —
verified against pif-directors-gathering, gfeai-v2, etc.), so all three screens 404'd. Ported the complete
backend from **deve-go `60fe949`** (`draft-guests` branch) and finished the UI. Task + file-by-file port:
[../tasks/008-guest-drafts-port/TASK.md](../tasks/008-guest-drafts-port/TASK.md).

**What it is:** a registrant who fills step 1 and requests an OTP but never completes is upserted into a new
`guest_drafts` table (keyed by email); the draft is **deleted on successful registration**. So the table is,
by definition, everyone who started and didn't finish — a follow-up/lead-recovery list plus drop-off
diagnostics (OTP sent/verified/attempts, email vs phone). New: migration, `GuestDraft`, `GuestDraftResource`,
`GuestDraftsController` (index/show/export), `GuestDraftsExport`. Capture hooks are **additive** in
`AuthController` (`saveGuestDraft()` in email/phone `Verification`; mark-verified in the `Confirmation`s) +
`GuestsController::store()` (delete on complete). `guest_drafts` is a **self-contained new table** — it does
not touch the `guests` schema, so blast radius on existing behaviour is nil.

**Durable decisions:**
1. **Build, not remove.** Real purpose + a complete, safe-to-port reference. (The earlier "guest-drafts is a
   dead stub" note in the storage plan §4a is superseded.)
2. **Dedicated `guest_drafts` RBAC permission** (`view`/`export`/`see_more`) in
   `app/Support/AdminPermissions.php`, **enforced on the 3 routes via `admin.can`** — so this PII (people who
   never finished) is grantable independently of `guests_listing`. Note this is *stricter* than
   `guests_listing`, whose routes have no `admin.can` gating (frontend-only) — a deliberate deviation.
3. **`personal_image` served as a signed URL** (private disk, D14), not deve-go's public rebuild.
4. **Capture more than the reference did.** deve-go dropped several step-1 fields. Here the frontend OTP
   payload (`VerifyEmailForm.formData`) also sends **gender/title_id/personal_image**, and the backend
   additionally captures **`category`** (frontend sends the slug → resolved to `category_id`) and
   **`invitation_token`**. Employee-ID + Days rows were removed from the see-more modal (never populated).

**Known limitation:** the pif **four-step** form has no invitation-token prop, so its drafts don't capture
`invitation_token` (one-step forms do). Separate wiring if needed.

**Landed (pushed):** backend `7a96707`, admin `270a60d`, frontend `a8a94ec`. Admin listing already used the
modern `ListingFilters` stack, so deve-go's separate `search-guest-drafts.tsx` was **not** ported.

**Gates:** pint + phpstan **No errors** + `migrate:fresh --seed` clean + tests **452/3** pre-existing; admin +
frontend `type-check` + build green. In-browser QA confirmed capture (incl. signed photo, category via slug,
invitation token) and delete-on-completion. **Not mobile-facing.**

## D20 — 2026-07-19 — dropped the dead `days` guest column (supersedes D18's ship-it call)

D18 shipped `guests.days` on the theory a clone would wire the writer. A re-audit this session confirmed
it stayed a **phantom**: no writer anywhere (the registration store and every `Guest::create`/`update`
omit it; the 3 write sites remained commented), with read-only consumers only — the admin
`JSON_CONTAINS(days,…)` filter, the dashboard "by days" stats, the export column, and the admin resource
— all against an always-NULL column. Cross-checked the sibling clone **`122-gfeai-v2`**: same dead column
there too, populated only by a demo seeder and **superseded by a new `forum_days` field**, so `days`
carries no live meaning in the lineage. **Decision: remove it fully** — column + `$fillable` + filter +
stats + export cell/heading + resource field + the commented writes — lockstep across all three repos,
including the orphaned `days` / `days_want_to_attend` / `day1-3` translations (EN+AR). **Kept:** the live
`guest_drafts.days`, the `event_days` module, and the frontend countdown `days` label (a timer string,
unrelated). Backend `4899cfd`, admin `c4f86c5`, frontend `e2da469`. Gates: pint + phpstan **No errors** +
`migrate:fresh --seed` clean + tests **457/3** pre-existing; admin + frontend `type-check` green.

## D21 — 2026-07-19 — GTM: keep in frontend (correctly wired), never in admin

Closes the D17 "GTM dormant" gap. **Admin: no GTM at all** — verified zero component code and no GTM env
var in either `.env.example_prod` or `.env.local`; nothing to remove. **Frontend: keep it** — `pages/_app`
+ `_document` read `NEXT_PUBLIC_GTM` (via `@next/third-parties/google`), and that variable now ships in
**both** the tracked `.env.example_prod` and the local `.env.local`, **empty by default (disabled)** so a
clone just drops in its `GTM-XXXX`. The old `NEXT_PUBLIC_GOOGLE_TAG_MANAGER` name mismatch that made it
dormant is already gone. No code/env changes were required — this entry records the decision and the
verified state.

## D22 — 2026-07-19 — Task 007: API response unification onto the standard envelope

Every API controller now returns the standard envelope from the 006 `ApiResponse` trait
(`{ success, status, message, data, meta? }` / `{ success:false, status:'failed', message, data, errors? }`),
via `BaseApiController::apiSuccess`/`apiError`. Rolled out in tiers: **admin (Tier A/B)** landed earlier this
session; **mobile (Tier C)** completes it — all `mobile/*` controllers migrated (`MobileAuth`, `MobileEventDay`,
`MobileSpeaker`, `MobileSponsor`, `MobileAttendee`, `MobileSession`(+`Feedback`), `MobileWorkshop`(+`Feedback`),
`MobilePublication`, `MobileMediaCenter`, `MobileQr`, `MobileRoom`, `MobileNotification`, `MobileChat`).

**Durable decisions:**
1. **Mobile break is accepted + documented.** Payloads that were flat/scalar/root-level now live under `data`.
   The per-endpoint delta is [../mobile/RESPONSE_SHAPE_DELTAS.md](../mobile/RESPONSE_SHAPE_DELTAS.md) (flipped to
   **IMPLEMENTED**), the "adapt later" artifact required by the Task 007 formula.
2. **`AppConfigController` left unwrapped** (`/app-config`, `/app-config/version`) — config documents the Flutter
   client deserializes wholesale, wrapping adds churn with no unification value. Flagged in delta §18.
3. **`status` keeps legacy values** (`"success"`/`"failed"` or the endpoint's existing custom value) so any
   superset reader survives; `success`+`message` are additive on every response.
4. **`routes/api.php` unchanged** — body refactor only, so the route contract (also the mobile contract) is intact.
5. Two listing-only controllers (`GuestEmailLogsController`, `InvitationEmailLogsController`) keep extending
   `Controller` to avoid a `private applyFilters()` collision with `BaseApiController`; they carry the full
   envelope via `->additional([...])` instead.

**Gates:** pint + phpstan **No errors**; tests **457/3** (the 3 are pre-existing D14 signed-avatar/env failures,
not from this work). Backend feature tests updated in lockstep with each controller. **Remaining:** mobile-team
ack of the deltas, then backend `dev` → `main`.

## D23 — 2026-07-19 — gate scanning is a first-party admin feature (standalone "agent admin" retired; new `scanning` RBAC feature)

On-site gate scanning was an **out-of-repo standalone web client** (the "agent admin"), reached via a
separate `loginAgent` and dual-prefix (`/admin/*` **and** no-`/admin`) scan endpoints that Task 010 had
frozen. `108-tasama` and `112-pif-partners-forum-demo` had already pulled the scanner **into** their admin
CMS keyed off an `admins.type='gate'` column — but ALT dropped `type` in favour of RBAC, so it was the
deliberate outlier with no in-admin scanner. **Decision: port scanning into the ALT admin as a first-party,
RBAC-gated feature and retire the standalone client + its shims.** Task + phase-by-phase detail:
[../tasks/011-scan-into-admin/TASK.md](../tasks/011-scan-into-admin/TASK.md).

**Durable decisions:**
1. **New `scanning` RBAC feature** (`['view']`) in `app/Support/AdminPermissions.php`, **distinct from
   `gates`/`areas`/`scans`.** `gates`/`areas`/`scans` = *manage/report* gates in the CMS; `scanning` =
   *operate* a gate (the live scan UI). A "gate agent" is just a role granting `scanning` (usually nothing
   else) — **no `type` column, no separate login**. They log in normally and the dashboard shows the scanner
   because `checkFeaturePermission('scanning', user)` is true.
2. **Data scope is enforced server-side** (closes the 108/112 gap where any token scanned any gate).
   `GatesController::deniesGateScope()` gates `showAgentSide`/`setup`/`start`/`pause`/`scan`: the target gate
   must match `admin.gate_id` (once bound by `setup`) and/or fall inside `admin.area_id`; **super-admin
   short-circuits**.
3. **Endpoints modernized + un-frozen** into a `/admin/gate-scan` group behind `admin.can:scanning`
   (`gates`, `gates/{id}`, `gates/{id}/setup|start|pause|scan`, `search-guests`, `scans/{id}/image`,
   `scans/{id}/guest`). The retained offline-sync endpoints (`attend`, `guest-data-offline`,
   `guest-data-sync`, `guests-printed-since`) moved behind the same feature. **Dropped:**
   `AuthController@loginAgent` + route, the four no-`/admin` scan aliases, and the dangling
   `validate-check-in/{regNumber}` route (handler gone since Task 010, no callers). This **lifts Task 010's
   RENAME_MAP §F freeze.**
4. **Camera lib swapped, dep count flat.** 108/112's `react-web-qr-reader` (unmaintained, React 16/17) →
   **`html5-qrcode`** (maintained, framework-agnostic, dynamic-imported so it never runs server-side); also
   dropped `react-lottie` (their scan-pulse dep) for a CSS pulse — net new deps ≈ 0.
5. **Recovery flow completed.** 108/112 only uploaded a badge photo for an unrecognised QR; ALT's
   `wrong-qr-recovery-modal` also **searches by name and links the guest to the orphan scan**
   (`search-guests` → `scans/{id}/guest`).
6. **Not mobile-facing.** The scanner is the web iPad client, not the Flutter app; the retired/renamed URIs
   were confirmed absent from the mobile contract, so `routes/api.php`'s mobile surface is intact.

**Deferred (still open):** the recommended `scans.gate_id` FK — the port ships on the existing `gate_name`
string link (`Gate::scopeWithUniqueCount`); and **live browser-QA + the RBAC/scope manual matrix** (needs a
running stack + camera), validated so far only by feature tests + type-check/build.

**Landed (on `dev`):** backend `211e17d` (scanning feature + endpoints + `GateScanTest`) → `cd66c21`
(`admins/select` filters by `scanning`); admin `1c87ff0` (gate-scan UI port + nav + `/scans` icon +
`type=gate` cleanup + EN/AR). **Gates:** backend `composer qa` green (pint + phpstan + tests incl.
`GateScanTest`); admin `yarn type-check` + `next build` green (`yarn production` needs the gitignored
`.env.production`).

## D24 — 2026-07-20 — `routes/api.php` brought to the cyan RESTful standard (grouped + `admin.can:` gated + `whereUuid` + `toggle-status`)

The legacy admin surface of `routes/api.php` was 966 lines of mostly-flat, ungated, non-RESTful routes
(vs cyan's 559-line grouped/gated/RESTful shape). Task 010 rewrote it as a **hard cutover** — no
alias/back-compat window; backend + admin + frontend callers moved together. Task + rename map:
[../tasks/010-api-routes-cleanup/TASK.md](../tasks/010-api-routes-cleanup/TASK.md) +
[../tasks/010-api-routes-cleanup/RENAME_MAP.md](../tasks/010-api-routes-cleanup/RENAME_MAP.md).

**Durable decisions:**
1. **Every admin resource folded into a `Route::prefix('admin/<res>')->group()` block**, static routes
   before `/{id}` wildcards, `->whereUuid('id')` on every UUID param.
2. **RESTful rename is a hard cutover** — `/x-select` → `/x/select`, `/guests-new` → `POST /guests`,
   `/guests-update/{id}` → `PUT /guests/{id}`, `/invitations-list/{id}` → `/invitations/{id}/list`, etc.
   Category updates went `POST` → `PATCH`. Admin API client + frontend public `countries/select` +
   `titles/select/{cat}` repointed in lockstep.
3. **`block/{id}` + `activate/{id}` replaced by a single `PATCH /{id}/toggle-status`** outright (no legacy
   pair kept); `toggleStatus()` added to the 14 controllers that only had `block`/`activate`.
4. **`admin.can:<feature>` gating on every admin group**, cross-checked against `AdminPermissions::CATALOG`
   (Super-Admin short-circuits, deny-by-default); per-action perms on writes; dashboard gated per-widget.
   **Left ungated on purpose:** all `-select`/`select-all` + `categories/slug` (feed cross-module dropdowns
   / guest forms — gating them would 403 an admin working in a *different* feature).
5. **Dead code removed:** dead comments, duplicate registrations, 7 orphaned `GuestsController` methods,
   the dead `guests-status-*` block, and 5 zero-caller ops/queue endpoints (+ `OperationActionsController`,
   `QueueController`).
6. **`mobile/*` is untouched** — the mobile contract (also `routes/api.php`) is intact; no `mobile/*` URI
   changed, so no `RESPONSE_SHAPE_DELTAS` row was needed. (The scanner/agent surface was frozen by Task 010
   then un-frozen + modernized by Task 011 / D23.)
7. **Final close-out (2026-07-20):** the last two non-RESTful, ungated, zero-caller leftovers —
   `POST /admin/guests-upload-zip` + `POST /admin/match-guests-images` (bulk guest-image upload/match) —
   were folded into the guests group as `POST /admin/guests/upload-zip` + `POST /admin/guests/match-images`
   behind `admin.can:guests_listing,edit`. The four offline-sync endpoints (`attend`, `guest-data-offline`,
   `guest-data-sync`, `guests-printed-since`) were **left at their URIs** — a deliberate Task 011 decision
   (D23), kept behind `admin.can:scanning`.

**Landed (on `dev`):** backend `4cf7036` (Tier 0+1 dead-code/endpoint cleanup) → `c5a3a31` (Tier 2 RESTful
cutover + prefix groups + `whereUuid` + `toggle-status`) → `9328d65` (Tier 4 `admin.can` gating) → `68723ee`
(dead ops/queue removal); admin `e36b384` (API client repoint + `toggle-status`); frontend `53d42e0`
(public `countries/select` + `titles/select`). **Route count 418 → 384.** **Gates:** backend `composer qa`
green (pint + phpstan **No errors** + tests **465/3** — the 3 are the pre-existing `ExampleTest` `/`→403 +
two avatar signed-URL fails, not from this work); admin + frontend `yarn type-check` green. **Remaining:**
manual smoke test per renamed/gated feature (needs a running stack + role matrix).

## D25 — 2026-07-20 — LinkedIn **automatic** "Share on LinkedIn" completed for the per-category social share

ALT already shipped the **manual** half of the category social-share feature (`with_share`,
`share_poster{,_ar}`, `share_type` = `manual|automatic`, `share_text_{en,ar}`, blade social card, admin form
section, bulk-update modal), but the **`automatic`** path the admin form advertised was never wired. Task 012
completed it, ported **best-of-both** from cyan (P37.4) + hci and adapted to ALT conventions. Task:
[../tasks/012-linkedin-auto-post/TASK.md](../tasks/012-linkedin-auto-post/TASK.md).

**Durable decisions:**
1. **Scope = Tier A only** (LinkedIn auto-post). ALT keeps its **blade** social card; cyan's Social Card
   **Layout Designer** / JSON document-rendering engine (Tier B) was explicitly **out of scope**.
2. **Per-category LinkedIn app credentials** — two nullable columns `linkedin_client_id` /
   `linkedin_client_secret` on `categories` (additive migration `2026_07_20_000001`), only consulted when
   `share_type = automatic`; unset → the OAuth endpoints 422. Creds round-trip through `update()`
   `$request->only()` + `CategoriesResources` (the gap cyan had). Not added to `bulkUpdateSocialMedia` — an
   app id/secret is inherently per-category, set individually in the form.
3. **LinkedIn API surface stays on v2** (`/v2/ugcPosts`, `/v2/assets?action=registerUpload`, `/v2/userinfo`,
   header `X-Restli-Protocol-Version: 2.0.0`) — the current documented surface for the **consumer**
   "Share on LinkedIn" product (`w_member_social`). The `/rest/*` Posts API needs a Marketing product and
   would 403 here; **do not "modernise" to `/rest/*`.**
4. **The three OAuth routes are public** (guest-facing, no admin/guest auth), in the public/localization
   group: `GET /linkedin/auth-url`, `GET /linkedin/call-back`, `POST /linkedin/post`. `getVisibility()` now
   also returns `share_type` so the public success page knows manual vs automatic.
5. **Frontend URL read via `config('app.frontend_url')`, not `env()`** — new config key mirroring
   `PUBLIC_FRONTEND_URL`, added so `LinkedInController` satisfies larastan `noEnvCallsOutsideOfConfig`
   (the pervasive legacy `env('PUBLIC_FRONTEND_URL')` calls elsewhere stay baselined; new code uses config).
6. **Frontend kept ALT-native** — the ported share component uses lucide `Share2`, `getApiError`, and
   `react-hot-toast` (not cyan's iconify / `ThreeDotsWave`). New `pages/[lang]/linkedin-redirect.tsx` OAuth
   landing page; `share_type` + `category_slug` thread `success.tsx` → `success-sections.tsx` →
   `sharebtn-sections.tsx`.
7. **`mobile/*` untouched** — new routes are public web only; no mobile contract delta.

**Landed (pushed, on `dev` — backend `c9884ca`, admin `c2885c2`, frontend `e8d7991`):** backend — migration + `LinkedInController` + `Category` fillable +
`CategoriesController` (`getVisibility`+`update`) + `CategoriesResources` + `config/app.php` + `routes/api.php`;
admin — `categories-form.tsx` + `interfaces/category.tsx` + EN/AR `web.json`; frontend — `linkedin-redirect.tsx`
+ `success/sharebtn-sections.tsx` + `success/success-sections.tsx` + `join/[category]/success.tsx`. **Gates:**
backend `composer qa` green (pint + phpstan **No errors** + tests **465/3** pre-existing); admin + frontend
`yarn type-check` + eslint green. **Remaining:** manual QA with a real LinkedIn Share app + running stack.

## D26 — 2026-07-20 — SMS provider config moved from `.env` to a DB-driven admin CRUD (cyan "SMS SMTP" port)

SMS transport credentials were pinned to `.env` via `config('services.unifonic')` and read directly in
`SendGuestSMSListener`. Task 013 ported cyan's **SMS provider config** stack so SMS credentials are now
DB-managed by admins (multi-row, `is_active` + a single `is_default`) exactly like the existing
`smtp_configs` mail stack — the foundation for the follow-up SMS tasks. Task:
[../tasks/013-sms-provider-config/TASK.md](../tasks/013-sms-provider-config/TASK.md).

**Durable decisions:**
1. **Mirror `smtp_configs`, not cyan verbatim** — ALT already runs the DB-driven SMTP pattern, so the SMS
   stack reuses its exact shape: multi-row, `is_active` + single `is_default`, secret masked on the wire
   (`app_sid_masked`, last 4), leave-blank-to-keep on edit, `BaseApiController` + `apiSuccess/apiError`,
   `admin.can:` gating, admin BFF-proxy calls, lucide + `getApiError` + `react-hot-toast` (no
   `js-cookie`/Bearer, no heroicons). Additive migration `2026_07_20_000002_create_sms_provider_configs_table`.
2. **Provider abstraction via `App\Services\Sms\SmsSender` + `provider_key`** — v1 validates `unifonic`
   only (`Rule::in(['unifonic'])`); a new provider = one enum append + a `sendVia*` branch + (optionally)
   surfacing the generic `api_key`/`api_secret` slots in the form. Unknown `provider_key` throws
   (`InvalidArgumentException`) so a stale row fails loud instead of silently no-op'ing. `SmsSender` returns
   the raw HTTP `Response` (Unifonic returns 200 even on failure — the `success` JSON key is the real signal).
3. **Secrets encrypted at rest** — `app_sid`/`api_key`/`api_secret` use the `encrypted` cast; the resource
   returns only `app_sid_masked`, and `api_key`/`api_secret` never leave the backend (`$hidden`).
4. **`send-test` + the listener honour the non-production guard** — on non-prod both return/record
   `sent:false` + a reason instead of hitting the provider, so dev/stage never sends real SMS.
5. **RBAC `sms_config` widened** `['view','update']` → `['view','create','update','delete','block']` to
   cover the new CRUD + toggle-active (block), matching `smtp_configs`. Routes live under a gated
   `admin/sms-provider-configs` group (admin-only).
6. **Clean cutover, no dual path** — the `config/services.php` `unifonic` block was **removed** (a fresh env
   setting `UNIFONIC_*` would silently do nothing), and the dead `GuestsController` debug methods
   `sendSMS2`/`sendSMS3`/`smsReplacePlaceholders`/`buildQrCodeLink`/`buildInvitationLink` (+ their commented
   `web.php` routes + the now-stale `env()` phpstan-baseline entry, `count: 2`) were pruned — they duplicated
   the real listener path and only referenced the old env creds.
7. **SMS templates left as-is (out of scope)** — ALT already has an unlinked `sms/templates` + `sms/config`
   admin page set; only the provider-config layer was missing. The follow-up "new SMS tasks" the user flagged
   will surface those. **`mobile/*` untouched** — new routes are admin-only.

**Landed (pushed, on `dev` — backend `96a15ce`, admin `661f134`):** backend — migration + `SmsProviderConfig` + `SmsProviderConfigResource`
+ `SmsProviderConfigController` + `Services/Sms/SmsSender` + `SendGuestSMSListener` (rewired) +
`AdminPermissions` + `config/services.php` (unifonic removed) + `GuestsController` + `routes/{api,web}.php` +
`phpstan-baseline.neon`; admin — `interfaces/sms-provider-config.ts` + `admin-modules/sms/provider-configs/*`
(form + listing + delete/send-test modals) + `pages/[lang]/sms/provider-configs/{index,create,edit/[id]}` +
`data/sidebar-links.tsx` + `data/module-icons.tsx` + EN/AR `web.json`. **Gates:** backend `composer qa` green
(pint **passed** + phpstan **No errors** + tests **465/3** pre-existing); admin `yarn type-check` + eslint
clean + `next build` compiles (new `/sms/provider-configs*` routes present). **Remaining:** manual send-test /
delivery QA in production with a real Unifonic app.

## D27 — 2026-07-20 — Per-flow SMTP override (choose which SMTP account sends each email flow)

Emails always used the single active-default `smtp_configs` row via `DynamicSmtpService::applyDefaultIfAvailable()`.
Task 015 lets admins pick a different SMTP account per flow, falling back to the default when blank/inactive.
Task: [../tasks/015-per-flow-smtp-override/TASK.md](../tasks/015-per-flow-smtp-override/TASK.md).

**Durable decisions:**
1. **One shared resolver** — `DynamicSmtpService::applyConfigById(?id)` applies an active override, else the
   active default (or `.env`). Inactive/deleted overrides never hard-fail a send.
2. **Category SMTP is two pickers** — `categories.smtp_config_id` covers notification emails
   (register-complete / accept / reject). Guest email-OTP uses a separate
   `categories.otp_smtp_config_id` (join form posts the `category` slug). Both blank → active default.
   Admin CMS login OTP is out of scope.
3. **Snapshot at create** — the resolved override is copied onto `guest_emails` / `automations` /
   `invitation_emails` so later category/setup edits don't change what a queued/sent row used. Guest
   email-OTP has no send-row → resolves live from the category slug.
4. **Automations: DB override beats `MAIL_HOST_BULK`** — the static `smtp-bulk` mailer is only used when no
   override is present.
5. **Invitations resolve invitation → collection → default** — nullable `smtp_config_id` on both
   `invitations` and `invitation_collections`.
6. **Admin pickers** list active configs only via ungated `GET admin/smtp-configs/select` (blank = use
   default). Additive migration `2026_07_20_000003_add_smtp_config_override_columns`.
7. **`mobile/*` untouched** — only an admin select route was added; public/OTP request/response shapes
   unchanged.

**Landed (pushed, on `dev` — backend `2adc387`, admin `76ae079`):** backend — migration + `applyConfigById` + snapshot wiring in
`GuestsController` / `InvitationEmail::createFromInvitation` / automation expansion + notification
`toMail()` paths + `AuthController` guest OTP + validation/resources + `smtp-configs/select`; admin —
`smtp-config-select` + pickers on category / automation / invitation / invitation-collection forms + EN/AR.
**Gates:** pint + phpstan clean; admin `yarn type-check` green. **Remaining:** manual QA with multiple SMTP
accounts.

## D28 — 2026-07-20 — Guest OTP SMS de-hardcoded onto the dynamic provider + per-flow SMS provider override (SMS mirror of D27)

Guest **phone-OTP** was sent by a hardcoded FGC gateway (`https://cnc.fgc.sa/authenticate` + `sendSmsNotifications`)
with a **plaintext username/password committed in `AuthController`** (`sdbankApi` / `SDB` sender). Both private
methods (`authenticateSMSAPI`, `sendSMS`) and the `Http` import are **deleted** — no static SMS gateway code
remains. OTP now flows through the same DB-driven stack register-complete SMS uses (D26: `SmsProviderConfig` +
`SmsSender`). Same shape as D27 for email.
Task: [../tasks/014-otp-sms-dynamic-config/TASK.md](../tasks/014-otp-sms-dynamic-config/TASK.md).

**Durable decisions:**
1. **OTP relies only on the dynamic provider** — `SmsSender` currently speaks Unifonic only, so OTP is now
   delivered by whatever active provider resolves (category override → active-default). The FGC gateway is
   gone; if a client mandates a specific gateway, add it as a `provider_key` branch on `SmsSender` (not
   hardcoded in a controller).
2. **Category SMS is two pickers** (mirrors D27) — `categories.sms_config_id` = register-complete
   notification SMS; `categories.otp_sms_config_id` = phone-OTP SMS. Both nullable FK → `sms_provider_configs`,
   null-on-delete. Blank/inactive/deleted → active-default. OTP resolves live from the join-form `category`
   slug; register snapshots onto `guest_sms.sms_config_id` at create.
3. **OTP sends in every environment** — unlike `SendGuestSMSListener` (which blocks non-production bulk/
   notification SMS), phone-OTP calls `SmsSender` directly and sends on dev/stage too, because it's a code the
   registrant is actively waiting for (preserves prior behaviour). ⚠️ In non-prod this now hits the *real*
   active-default provider instead of the old FGC test gateway.
4. **Admin picker** lists active providers via new ungated `GET admin/sms-provider-configs/select`; reusable
   `sms-provider-config-select` component. Additive migration `2026_07_20_000005_add_sms_config_override_columns`.
5. **`mobile/*` untouched** — `phone-verification` request/response shapes unchanged; only an admin select
   route was added. Mobile login-OTP (separate path) not in scope.

**Landed (pushed, on `dev` — backend `ae2ad17`, admin `568e56a`):** backend — migration `000005` + `AuthController::phoneVerification`
rewritten (FGC methods deleted) + `SendGuestSMSListener` honours the snapshot + `GuestsController` register/
resend snapshot + `Category`/`GuestSMS` fillable + `CategoriesController` validation/persist +
`CategoriesResources` + `SmsProviderConfigController::selectList` + route. admin — `sms-provider-config-select`
+ two category pickers + `interfaces/category` + EN/AR. **Gates:** pint + phpstan clean; admin `yarn
type-check` + eslint green. **Remaining:** manual QA with a real active provider (dev now sends via the
active-default, not FGC).

## D29 — 2026-07-20 — SMS flow parity: accept/reject + automations + invitations now text (mirrors the email flows)

Before this, email fired on register-complete / accept / reject / automations / invitations, but SMS only
fired on register-complete + phone-OTP. Task 016 closes the three gaps so every guest email flow has an SMS
twin, each with its own optional provider override (extends D26/D28).
Task: [../tasks/016-sms-flow-parity/TASK.md](../tasks/016-sms-flow-parity/TASK.md).

**Durable decisions:**
1. **Guest-backed flows reuse `guest_sms` + `SendGuestSMSEvent`** — accept / acceptToCategory / reject
   (Stage 1) and automations (Stage 3) already target a `Guest`, exactly like register-complete, so they
   create a `guest_sms` row (snapshotting `sms_config_id`) and dispatch the existing listener. **No new
   per-flow SMS table** for these. The listener's non-production block still applies (dev/stage won't send
   bulk/notification SMS).
2. **Invitations get their own `invitation_sms` table + listener** — invitations are token-based, not
   guest-backed, so they mirror `invitation_emails`: nullable `sms_template_id` + `sms_config_id` on both
   `invitations` and `invitation_collections`; `InvitationSms::createFromInvitation` resolves
   invitation → collection; `SendInvitationSmsEvent`/`SendInvitationSmsListener` render invitation-specific
   placeholders (`{{ first_name }}`, `{{ last_name }}`, `{{ invitation_link }}`) and send via `SmsSender`.
   Fired from invite / bulk / reminder alongside the email event; `extract-bulk` inherits the source
   collection's SMS template unless overridden.
3. **Automations add `with_sms_template`** (mirrors `with_email_template`) + `sms_template_id` +
   `sms_config_id` on `automation_setups`. SMS is **independent of email** — a setup may text, email, or
   both. `AutomationController::send` fires the guest_sms event per guest when the toggle is on; `split`
   carries the SMS fields to each chunk.
4. **Provider override, same rule as D28** — blank/inactive/deleted `sms_config_id` → active-default. The
   choice is snapshotted onto the send-row at create time.
5. **Out of scope:** OTP text stays inline (D28); `SmsSender` remains Unifonic-only; **`mobile/*`
   untouched** (only admin forms/routes changed).

**Landed (pushed, on `dev` — backend `7ba6152`, admin `79f1995`):** backend — migrations `000006` (invitation SMS) + `000007` (automation
SMS); `InvitationSms` model + `SendInvitationSmsEvent`/`SendInvitationSmsListener` (+ `EventServiceProvider`);
`InvitationsController` / `InvitationsCollectionController` persist + dispatch; `AutomationSetupsController`
store/split persist; `AutomationController::send` dispatches guest_sms; `AutomationSetup` /
`Invitation` / `InvitationCollection` fillable + casts; `Invitation`/`InvitationCollection`/`AutomationSetups`
resources expose the fields. admin — SMS template + provider pickers on invitation / collection / extract-bulk /
automation forms; `with_sms_template` toggle; `sms-template-select` `errors` prop made optional; interfaces +
EN/AR (`sms_override`, `with_sms_template`). **Gates:** backend `pint --test` **passed** + `phpstan` **No
errors**; admin `yarn type-check` + eslint green. **Remaining:** manual QA with a real active SMS provider.

## D30 — 2026-07-21 — Invitations send on a single delivery channel (email | sms), not both

**Reverses the parallel-send half of D29.** D29 wired an SMS twin *alongside* the invitation email, so a
collection could fire both at once. In practice that made status, error-tracking, and "resend" ambiguous
(resend *what* — the email, the SMS, or both?). An invitation collection now picks **exactly one** channel.
Task: [../tasks/017-single-channel-invitations/TASK.md](../tasks/017-single-channel-invitations/TASK.md).

**Durable decisions:**
1. **`channel` enum on `invitation_collections` + `invitations`** (`email` | `sms`; `whatsapp` reserved but
   rejected server-side), default `email`, folded into migration `2026_07_20_000006`. The store/update paths
   **scope the template + provider to the chosen channel** and null the other side, so only that channel can
   fire. `extractBulk` and the collection edit propagate `channel` down to child invitations (sends read the
   per-row snapshot).
2. **Admin picks the channel up-front** via a dropdown, and unconfigured channels are **disabled** (gated on
   a configured default SMTP / SMS provider) with a link to set them up — you can't choose a channel you
   can't send on. Overrides follow the category switch-before-dropdown pattern (toggle on → show picker).
3. **Status is channel-agnostic** — a successful SMS send bumps `invitation.is_sent` / `send_count` exactly
   like email, and the `invite` guard checks `phone` for SMS collections (not `email`). Resend re-sends the
   one channel the collection is on.
4. **Observability:** `InvitationResource` exposes `channel` + `sms_template_name`; the listing shows a
   channel badge and the see-more modal has an SMS-history tab twinning the email history.
5. **WhatsApp is deliberately deferred** — reserved in the picker as a disabled "coming soon" option; the
   next channel task, not this one.

**Landed (pushed, on `dev`):** backend `0fcd3c5` (`InvitationsController` / `InvitationsCollectionController`
scope + propagate, channel-aware guard, SMS status bump; `Invitation`/`InvitationCollection` fillable +
`smsTemplate` relation; `Invitation`/`InvitationCollection` resources; migration `000006` channel column);
admin `f1589df` (channel picker + per-channel override sections on the create + collection-edit forms;
channel badge in listing; SMS-history tab; wider bulk-send modal with channel/phone/template columns;
reorganized update-info modal; `DialogShell` 4xl/5xl; `ui-select` disabled options; titles preloaded once;
interfaces + EN/AR). **Gates:** backend `pint --test` + `phpstan` **No errors**; admin `yarn type-check` +
eslint green. **Remaining:** manual QA with a real active provider.

## D31 — 2026-07-21 — Category communications restructured (master email/SMS gates + `with_email_otp` rename) + SMS log observability

Two related consolidations. Task:
[../tasks/018-sms-logs/TASK.md](../tasks/018-sms-logs/TASK.md) (logs) and the categories comms work folded
into 017's session.

**Durable decisions:**
1. **Master channel switches `with_email` / `with_sms`** on categories, enforced centrally in
   `Category::getNotificationTemplate` — when a master flag is off the backend returns no template for that
   channel, so it **never sends** on it regardless of per-event settings. Child per-event values are kept
   (turning the master off hides but doesn't destroy them).
2. **`with_otp` → `with_email_otp` rename** for symmetry with the pre-existing `with_sms_otp`. Value/meaning
   unchanged (category requires an email-OTP step). Cross-repo lockstep: backend resource + guest OTP guard,
   admin form, **and** the public frontend join pages (they read the flag off the category payload and pass
   it as `withOtp` — miss one and email OTP silently disables). **Mobile payload rename** — invitation-verify
   responses now return `with_email_otp`; documented in
   [../mobile/MOBILE_NOTICE_CATEGORY_WITH_EMAIL_OTP_RENAME.md](../mobile/MOBILE_NOTICE_CATEGORY_WITH_EMAIL_OTP_RENAME.md).
   Both flag changes folded into the categories migration (no new migration — `migrate:fresh` workflow).
3. **Admin-access scope on the categories form** — a new "Admin access" tab lists every admin (any
   role/status) so a fresh category can be dropped into their data scope at create/edit time, backed by a new
   `GET admin/categories/assignable-admins` endpoint.
4. **SMS logs** — read-only guest + invitation SMS log listings mirroring the email logs, behind a new
   `sms_logs` RBAC feature (`view` / `export`). `guest_sms` and `invitation_sms` each get a
   controller/resource/export + a super-gated admin page under `/logs`, beside the email logs in the sidebar.
   Note: SMS `is_sent` is a real boolean (email logs use `yes`/`no` strings), and SMS has no
   delivered/open/click, so those columns are intentionally dropped.

**Landed (pushed, on `dev`):** backend `3c73f0f` (categories comms + `assignableAdmins` + `with_email_otp`
rename across resource/model/migration/seeder/export + guest OTP guard) + `34e09e7` (SMS log
controllers/resources/exports + `sms_logs` permission + routes); admin `702d9b1` (admin-access tab +
validation-error surfacing + form width/gating) + `8b0960f` (SMS log pages + sidebar) + `69ddd13`/`4c83678`/
`2cf9409`/`4c83678` (earlier P17 comms/status UI); frontend `0d0d82b` (join pages read `with_email_otp`).
**Gates:** backend `pint --test` + `phpstan` **No errors**; admin + frontend `yarn type-check` + eslint
green. **Remaining:** manual QA with live providers.

## D32 — 2026-07-22 — Logistics + e-visa re-added (Task 019): ops-confirmed data stays its own table; the February e-visa console is dead code

The P5.trim (backend `08d542e`, admin `e3a0677`, both 2026-06-24) removed hotels/rooms,
traveling-status, guest logistics and e-visa as "unused modules". Task 019 brings them back,
modernized to everything this repo adopted afterwards (Tasks 001/002/009/010, P17.4).
Task: [../tasks/019-logistics-evisa-port/TASK.md](../tasks/019-logistics-evisa-port/TASK.md).

**Durable decisions:**

1. **`guest_logistics` remains a separate 1:1 table — do NOT fold it into `guests`.** The two hold
   different data, not duplicates. `guests` carries what the registrant DECLARED on the join form
   (`expected_date_of_arrival`, `flight_arrival_time`, `require_flights`, `check_in_date`);
   `guest_logistics` carries what operations BOOKED (`arrival_date`, `arrival_time`,
   `check_in_date`). Both tables legitimately have `check_in_date`. Collapsing them destroys the
   "guest asked for the 5th, we booked the 6th" comparison. The admin UI labels the ops side
   `admin_*` for this reason — those ~24 EN/AR keys survived the trim and were reused.

2. **The February e-visa "operations console" is NOT ported, and should not be.** It lives only on
   the unmerged `origin/evisa` (backend `95d426d`) + `origin/Imtnan` (admin `a10fd14`).
   `merge-base(evisa, e-visa) = a379b35` and `evisa` is not an ancestor of `e-visa` — they are
   parallel attempts, and hci merged the March one. hci `main` has no trace of the console and its
   last e-visa commits are 2026-03-03. It also **conflicts**: it drives `e_visa_status` with
   `pending`/`in_process`/`received` while the shipped lifecycle uses `in_progress`/`issued` — two
   state machines on one column, `in_process` vs `in_progress` being a silent trap. Its
   `deriveGuestState()` additionally reads columns that do not exist here and treats
   `empty($guest->valid_visa)` as "no visa", which with a real boolean makes `null` (never asked)
   read as eligible.

3. **E-visa eligibility is `valid_visa === false`, strictly.** `valid_visa` is a nullable boolean
   here (Task 001 Track A), not hci's `'yes'`/`'no'` string: false = declared no visa/iqama =
   eligible; true = already holds one; **null = never asked** (only foreign nationals are). A
   `!== true` shortcut would make every Saudi/GCC registrant eligible. The e-visa listing endpoint is
   hard-scoped to `valid_visa = false` — do not widen it; those rows carry passport-grade PII.

4. **E-visa file I/O is entirely on the `private` disk** (Task 006). hci reads
   `base_path('public/storage/uploads/...')` and writes `storeAs('public/...')`. Here reads go via
   `addFromDisk('private', …)`, writes via `Storage::disk('private')->putFileAs()` into
   `GuestDocumentController::TYPES`, and the package zip saves to the private disk. The visa xlsx is
   piped into the zip with `Excel::raw()` rather than hci's temp file, so no PII spreadsheet is left
   on disk.

5. **Bulk guest queries stay on `GuestsController`.** `applyAdminGuestAccessFilter` (the P18.3
   category scoping) is private there, and hci's `EVisaController` carries a third copy of it.
   Not duplicated: `EVisaListing`, `ExportVisa` and the four logistics exports live on
   `GuestsController`; `EVisaController` and `GuestLogisticsController` hold only per-guest,
   id-keyed operations.

6. **`inferFeatureId` is first-match-wins, and nested pages MUST precede their parent.** Rooms live
   at `/hotels/[hotel_id]/...`, logistics at `/guests/[slug]/logistics/...`. Both needed their rule
   inserted ABOVE the broader `/hotels` and `/guests` rules, or the page gates on the wrong feature
   and renders for an admin whose API calls then 403. Callers pass `router.pathname`, so the literal
   `[hotel_id]` segment is matchable and exact. This is the bug class already shipped in `sms_logs`.

**Landed (pushed: NO — local on `dev`):** backend `303c629` (hotels/rooms/traveling-status +
guest_logistics schema, models, controllers, resources, routes, RBAC), `0ed06f3` (4 logistics
exports), `0ea04fb` (`valid_visa` persistence fix), `89f1673` (e-visa generation, PDF, issued-visa,
export, listing); admin `8641f65` (hotels/rooms/traveling-status CRUD), `01764ab` (per-guest
logistics screen + hotel assignment), `83ba223` (e-visa console); frontend `b0e7e49` (`valid_visa`
sent as a boolean).

**Gates:** backend `pint --test` + `phpstan` **No errors** + `migrate:fresh --seed` clean; admin
`yarn type-check` + `next build` **123/123 pages**; frontend `yarn type-check` + build clean; EN/AR at
parity (web 1611/1611). **Mobile contract unaffected** — every new route is `/admin/*`, nothing
removed or renamed, no `Mobile*` resource touched, so no mobile notice was issued.

**Remaining:** `sendIssuedVisa` is NOT ported — hci `main` emails the issued visa
(`workflow_value = 'ON_ISSUED_VISA'`, sets `e_visa_status = 'sended'`). `issued_visa_send_count` /
`issued_visa_sent_at` exist here with casts and the console renders a send-count column, but nothing
writes them; the columns are kept deliberately for when it is built. Missing prerequisites:
`categories.issued_visa_email`, `guests.issued_visa_sent`, `guests.issued_visa_sent_by`, and the
`ON_ISSUED_VISA` workflow value. Also open: the logistics screen's `Promise.all` fails hard for an
admin with `guest_logistics` but not `guests_listing,see_more`; and manual QA of the whole flow.

## D33 — 2026-07-22 — `inferFeatureId` is first-match-wins: six mismatches found, and the sweep that finds the seventh

Closing out Task 019 surfaced the same defect class six times, twice in code that had already
shipped. `utils/inferFeatureId.ts` maps a route to its RBAC feature by `Array.find` — **first match
wins** — and it is consumed by `TypeGate` (page access) and `TopSection` (action buttons), while the
sidebar carries its own hand-declared `featureId`. When the two disagree, nothing errors: the link
appears and the page refuses, or the page renders and the API 403s.

**Durable rules:**

1. **A nested route MUST be registered above its parent.** `/hotels/[hotel_id]/...`,
   `/guests/[slug]/logistics/...`, `/logs/(guest|invitation)-sms`, `/emails/smtp-configs` all sit
   under a broader rule that would otherwise swallow them.
2. **Callers pass `router.pathname`**, i.e. the Next route PATTERN with the parameter name intact
   (`/hotels/[hotel_id]/rooms/create`) — never a resolved id. Matching the literal `[hotel_id]` is
   therefore exact, and lets a rule cover a detail page without also matching sibling static routes
   like `/hotels/create`.
3. **Every new page needs a rule.** A missing rule resolves to `null`, and `TypeGate`/`TopSection`
   then fall back to Super-Admin only — which silently defeats a purpose-built feature (see
   `/gate-scan`, where the whole point of the `scanning` feature from D23 is a gate-agent role that
   is not a super admin).
4. **The sidebar's declared `featureId` and `inferFeatureIdFromPath(href)` must agree.** This is
   mechanically checkable and should be a test — a ~20-line script parsing both files caught the two
   pre-existing cases (`/emails/smtp-configs`, `/gate-scan`) that six rounds of human/agent review had
   missed. All 28 live sidebar links now agree.

**Instances:** rooms, guest logistics and e-visa (caught during Task 019, never shipped wrong);
`sms_logs` (shipped in P18.2 — `/logs/guest-sms` + `/logs/invitation-sms` gated on `email_logs`);
`/emails/smtp-configs` and `/gate-scan` (both long-standing).

**Also landed in this pass (P20):**
- admin `621abdb` sms_logs rules; `1bd50e5` smtp-configs + gate-scan rules
- admin `7193f6f` — the logistics screen called `/admin/guests/...` while the axios base already ends
  in `/api/admin`, so every request 404'd and the screen rendered its error state on every load. It
  shipped that way in P19.4. **Rule: admin Axios paths are relative to a base that already includes
  `/api/admin` — never prefix `/admin/`.**
- admin `436d225` — the logistics loader's `Promise.all` spans two features
  (`guest_logistics` + `guests_listing,see_more`); the guest leg is now tolerant, hides the
  guest-declared fields when unreadable, and drops them from the PUT so they cannot be nulled unseen.
- backend `ecce5a6` — `CategoriesExport::map()` omitted `visibility`, printing 20 values under 21
  headings; every column from position 6 was shifted one left.
- backend `c9bbdf9` — `phpunit.xml` had the sqlite/`:memory:` env commented out, so `php artisan test`
  ran `RefreshDatabase` against the real MySQL dev database. Two `EventDaysTest` assertions compared a
  raw `date` column (`2026-11-16` on MySQL vs `2026-11-16 00:00:00` on SQLite) and were rewritten to
  compare through the model cast. With the avatar fixtures and the stock `ExampleTest` also fixed,
  `composer qa` is green end to end (467/0) and the documented "465 pass / 3 pre-existing" baseline is
  retired.

## D34 — 2026-07-22 — E-Visa lifecycle closes: `sent` is the terminal state, and `check:rbac` guards the map

**Durable decisions:**

1. **The e-visa lifecycle is `null → in_progress → issued → sent`.** `sent` is written by
   `EVisaController::sendIssued` when the issued visa is emailed to the guest. hci `main` uses the
   typo'd `'sended'` for this state — alt does **not**; anything ported from hci that compares against
   `'sended'` is wrong here.
2. **`sendIssuedVisa` reuses the existing guest-email machinery**, it does not send mail directly: a
   `GuestEmail` row with `workflow_value = 'ON_ISSUED_VISA'` plus `SendGuestEmailEvent`, exactly like
   the other category-driven notifications. That inherits the D27 per-flow SMTP override for free.
3. **`categories.issued_visa_email` is the template selector**, nullable FK → `email_templates`,
   picked on the category form's actions tab, ungated (there is no `with_issued_visa` toggle). The
   send endpoint 422s with a message naming that exact path when it is unset.
4. **`guests.issued_visa_sent` from hci was deliberately NOT added.** `issued_visa_sent_at` is a
   nullable timestamp and already encodes "has it been sent"; a parallel `'yes'`/`'no'` string would be
   a second source of truth. `issued_visa_sent_by` (FK → admins) *was* added, for the audit trail.
5. **Re-sending is intentional.** Every send increments `issued_visa_send_count`; the confirm step
   tells the operator how many times the guest has already received it. Do not "fix" this into a
   one-shot.

**A process note worth keeping.** The two lanes that built this each did their job correctly and the
result was still unusable: the backend required `categories.issued_visa_email` and 422'd with "set it
in Categories → …", while nothing in the admin could set it, because that lane was scoped to the
e-visa console. Neither lane was wrong; the SEAM was. When splitting work, the thing to verify is not
each half but the path a user actually walks through both.

**Guard added:** `yarn check:rbac` (admin, zero-dependency) cross-checks every sidebar `featureId`
against `inferFeatureIdFromPath(href)` and against the backend catalog, exiting non-zero on
disagreement. Written because the D33 trap had produced six bugs and every one was mechanically
detectable. Consider running it beside `yarn type-check` in the gate.

**Landed (pushed, on `dev`):** backend `6219994`; admin `1416537` (check:rbac) and its parent (send
action + category picker). **Gates:** `composer qa` green (pint + phpstan + 467/0);
`yarn type-check` + `next build` 123/123; EN/AR parity 1618/1618; `migrate:fresh --seed` clean with
both new columns.

**Remaining:** none of Task 019, P20 or P21 has been exercised in a browser. The e-visa **send**
action in particular has never been run — it emails a real guest via a real template.

## D35 — 2026-07-24 — Email/SMS audit hardening pass (P23): 33 fixes, and the two deliberately left

A full audit of the email/SMS subsystem (transport → send flows → single-channel invitations → category
gating → logs → admin/frontend) surfaced 45 candidates; **33 survived adversarial verification** and were
fixed across 14 reviewed commits (backend P23.1–P23.12, admin P23.13, frontend P23.14).

**Durable decisions:**

1. **SMS sends in every environment now** (`28eca40`). The `config('app.env') !== 'production'` block was
   removed from both SMS listeners and the provider test endpoint. Rationale: `SmsSender` only talks to the
   real Unifonic API — there is no `log`/fake transport like email's `MAIL_MAILER=log`, so the guard made
   local/stage SMS untestable. **Consequence:** any SMS trigger (incl. a bulk automation/invitation run)
   sends real texts with real cost from dev/stage — test with your own number. Phone-OTP already sent in all
   envs, so this only makes the other flows consistent.
2. **Email was left sending in every environment too — H2 NOT applied.** The audit flagged email's missing
   non-prod guard as high severity, but a guard was explicitly rejected: it breaks local email testing.
   Email delivery is controlled the normal Laravel way (`.env` `MAIL_MAILER` → `log`/mailpit); the only real
   exposure is a staging box pointed at prod data with an active `smtp_configs` row, which is an ops concern.
3. **Accept/reject notifications were silently dead and now work (H1, `d0cd0b0`).** `GuestsController::accept`
   and `reject` constrained-eager-loaded the category without `with_email`/`with_sms`, so the D31 master gate
   read them as `null` and returned no template on **every** accept/reject — killing that mail/SMS even when
   the admin had configured it. Register/accept-to-category load the full model and were unaffected.
4. **Phone OTP hardened (`0c8ee04`):** CSPRNG (`random_int`), a **30-minute TTL**
   (`config('auth.phone_otp_ttl_minutes')`, env `PHONE_OTP_TTL_MINUTES`) enforced at both `phoneConfirmation`
   and the registration gate, and persist-after-send (the token row is written only once the SMS goes out).
5. **Automation `send()` is idempotent (`14085a6`):** a setup already `started`/`splitted` returns **409**
   instead of re-blasting; only not-yet-sent rows are processed. Also: the status-change / category-move /
   poster side-effects now run on the **email** path (previously only the no-email branch), and `split()`
   carries `with_e_badge` (chunked automations no longer drop the e-badge).
6. **Category OTP flags require their master channel (`e8da311`, L-nocrossvalidation).** `store()`/`update()`
   reject `with_email_otp` without `with_email` (and the SMS pair), matching the documented master-switch
   semantics (D31). Validation-only — never touches data at rest.
7. **`env('PUBLIC_FRONTEND_URL')` → `config('app.frontend_url')`** in the guest SMS listener and the email
   variable resolver (`d1ae700`, `eaa76be`) — `env()` returns null under `config:cache`, yielding host-less
   links in a queued worker. Two stale `phpstan-baseline.neon` env entries were pruned in the same breath.

**The two deliberately deferred (with a precondition, not just "later"):**

- **L-otpgate — gate the OTP *send* on the master `with_email`/`with_sms` flag.** The docs say master-off
  should block OTP too, but `with_email`/`with_sms` were added with `default(false)` and **no backfill
  migration**, and `migrate:fresh` is now banned (real data), so the live state is unknown. Gating the send
  before auditing risks 500ing registration for any category with `with_email=false` + `with_email_otp=true`.
  **Precondition:** run `SELECT id, name_en FROM categories WHERE (with_email_otp=1 AND with_email=0) OR
  (with_sms_otp=1 AND with_sms=0);` — 0 rows → safe to gate; any rows → fix that data first. D35.6's new
  validation stops fresh inconsistencies regardless.
- **L-reminderemail — SMS-only invitations have no reminder path** (`sendReminderByEmailsList` keys on the
  `email` column, which is null for SMS-only invites). This is a feature (phone-list endpoint + admin UI +
  mobile-contract check), not a fix — fold into the WhatsApp work.

**Landed (committed on `dev`, NOT pushed — awaiting review):** backend P23.1–P23.12
(`d0cd0b0`,`28eca40`,`5b934c1`,`0c8ee04`,`d1ae700`,`b1f07e8`,`d7bf834`,`42e6269`,`14085a6`,`eaa76be`,`7eb1e54`,`e8da311`);
admin `f17ed0a`; frontend `c69f35c`. **Gates:** every commit green — backend `pint --test` + `phpstan`
No-errors; admin/frontend `yarn type-check`. **`routes/api.php` untouched → no mobile-contract delta.**

**Remaining:** the L-otpgate data audit above; manual QA of the changed flows against a live provider —
especially the now-real test-SMS endpoint and the 30-min OTP TTL vs. the 4-step form's real completion time.

## D36 — 2026-07-24 — WhatsApp channel added end-to-end (invitations + guest/automation notifications + inbound RSVP + WhatsApp OTP), built as a twin of the email/SMS stack

A full WhatsApp delivery channel (Meta WhatsApp Cloud API) was added across the backend, admin and public
frontend, mirroring the existing email/SMS subsystems feature-for-feature. `x-hci-campaign` was assessed as a
possible base and used as a **reference only** for the Meta-specific mechanics (Graph payload, webhook signature,
template variable/button mapping) — every ALT-facing piece (provider config, templates, single-channel invitations,
observability tables, event/listener sends, category master-gate, admin listing/form layout, join-form OTP) was
built as a faithful twin of our own code, not ported. Plan:
[../upgrades/WHATSAPP_INTEGRATION_PLAN.md](../upgrades/WHATSAPP_INTEGRATION_PLAN.md).

**Durable decisions:**

1. **Everything twins email/SMS; x-hci is reference-only.** `WhatsAppProviderConfig` mirrors `SmsProviderConfig`
   (multi-row, `is_active` + single `is_default`, encrypted creds, secret masked on the wire, leave-blank-to-keep);
   `WhatsAppTemplate` is a Meta-pointer twin of `SmsTemplate`; sends go through queued events/listeners
   (`SendInvitationWhatsAppEvent` / `SendGuestWhatsAppEvent`) exactly like the SMS pair; observability is separate
   tables (`invitation_whatsapp` / `guest_whatsapps`, `createFromInvitation` snapshot) twinning `invitation_sms` /
   `guest_sms`; admin screens copy the `sms/provider-configs` + `sms/templates` layout. No `DynamicFormRenderer`,
   no new conventions.
2. **Transport is `App\Services\WhatsApp\WhatsAppSender`, keyed on `provider_key`** — v1 speaks `meta_cloud` only
   (`Rule::in(['meta_cloud'])`); a new provider = one enum append + a `sendVia*` branch. It builds the Meta Graph
   `type:template` payload (name + language + components: body params + URL/quick-reply button) and returns the raw
   HTTP `Response`. `WhatsAppTemplateRenderer` resolves a template row → send-args (invitation / guest / OTP), with
   per-language (`_en`/`_ar`) Meta template name, body-variable and button-URL-index mapping.
3. **Schema changes are additive forward-only migrations.** `migrate:fresh` is banned now that the DB holds real
   data, so four new `2026_07_24_*` migrations ADD the whatsapp tables/columns (`whatsapp_provider_configs`,
   `whatsapp_templates`, whatsapp FKs + `invitation_whatsapp` on invitations, `guest_whatsapps` + automation/category
   whatsapp columns) — no existing migration was edited.
4. **Invitations extend the single-channel model (D30) to a third channel.** `channel` is now
   `email | sms | whatsapp`; store / extract / collection-edit scope the template + provider to the chosen channel and
   null the others, so exactly one channel fires. Guest-backed flows (register / accept / reject / automations) gate on
   the category `with_whatsapp` master switch via `Category::getNotificationTemplate('...','whatsapp')`, mirroring the
   D31 email/SMS master gates.
5. **Inbound RSVP webhook.** `GET /webhooks/whatsapp` verifies `hub.verify_token` against the active-default
   provider's `verify_token` (`hash_equals`); `POST /webhooks/whatsapp` validates the Meta `X-Hub-Signature-256`
   HMAC with the provider `app_secret`, then per-entry: delivery **statuses** update the matching log row, and
   quick-reply **button** replies set `reply_status` (`confirmed` / `declined`) + `replied_at`. A `confirmed` reply
   fires a QR follow-up (a `purpose='qr'` active template) through the same send pipeline.
6. **WhatsApp OTP shares the SMS phone code — a delivery twin, not a second code.**
   `AuthController::whatsAppVerification` writes to the **same** `phone_verifications` table and is verified by the
   **same** `phoneConfirmation`; only the send endpoint differs (a `purpose='otp'` template + `otp_whatsapp_config_id`).
   The registration gate accepts either `with_sms_otp` or `with_whatsapp_otp`; `store()`/`update()` cross-validate
   `with_whatsapp_otp ⟹ with_whatsapp`. The public join form offers WhatsApp as an OTP channel, reusing the phone
   step / confirm / `sms_otp_token` verbatim.
7. **RBAC: two new features** — `whatsapp_config` + `whatsapp_templates` (CRUD, mirroring `sms_config` / sms
   templates) and `whatsapp_logs` (`view` / `export`), all under `admin.can:`-gated route groups.

**Landed (committed on `dev`, NOT pushed — awaiting review):**

- backend (12): `d33affa` (W0.1 provider CRUD), `f185608` (W0.2 template CRUD), `28ec5d8` (W0.3 `WhatsAppSender` +
  send-test), `d4daaf8` (W1a invitation pipeline), `51b0622` (W1b invitation controllers), `2ee06bf` (W1c invitation
  logs + `whatsapp_logs` RBAC), `555a91e` (W2a guest/automation pipeline + `with_whatsapp` gate), `114bc35` (W2b
  fan-out + register/accept/reject), `7a370b0` (W2c guest logs), `f7e8ea8` (W3 webhook), `14eff38` (W4 OTP), `c224dd3`
  (W4 expose `with_whatsapp_otp`).
- admin (5): `8d4f2f3` (W0.4 provider UI), `030df8b` (W0.5 template UI), `0f09641` (W1d invitation channel picker),
  `e536590` (W2 automation toggle + category gate + log pages), `f58872f` (RSVP badge alignment).
- frontend (1): `657c173` (W4 join-form WhatsApp OTP option).
- docs: `1d9ee7f` on `main` (`WHATSAPP_INTEGRATION_PLAN.md`, committed earlier).

**Gates:** every commit green — backend `pint --test` + `phpstan` **No errors**; admin + frontend `yarn type-check`
(husky eslint/prettier). EN + AR landed in the same commits.

**Mobile contract:** additive only, but new **public** surface area — three new public routes
(`GET` + `POST /webhooks/whatsapp`, `POST /guests/whatsapp-verification`) and an additive `with_whatsapp_otp` field
on the invitation/category verify responses. Nothing was removed or renamed, so no existing mobile endpoint changed;
the webhook + verification routes are server-to-server / web and not consumed by the Flutter client. Confirm against
`docs/mobile/BACKEND_INCOMING_CHANGES_FOR_MOBILE.pdf` before wiring mobile to any of it.

**Remaining:** manual QA against a real Meta WABA — template approval, live template send, webhook signature + RSVP
round-trip (confirm → QR follow-up), and OTP delivery; nothing is exercised against Meta yet. Also: when a category
enables **both** `with_sms_otp` and `with_whatsapp_otp`, the join form has no channel picker and prefers WhatsApp —
add a picker (or a precedence decision) if both-on becomes a real configuration.

## D37 — 2026-07-25 — Automations move to a single delivery channel (email | sms | whatsapp), mirroring the invitations channel model; the create form reskinned to the Section-card pattern

An automation setup previously used the **multi-toggle** model — three independent booleans
(`with_email_template` / `with_sms_template` / `with_whatsapp_template`), any combination of which fired for the
same guest in one run. It now picks **exactly one** delivery channel, adopting the invitations single-channel
`channel` concept (D30 / D36) for a clean, trackable "this automation went via X" record. This is a deliberate
**behaviour reduction** — an automation can no longer notify across multiple channels at once — chosen over the
multi-channel-cards alternative after weighing it against tracking/consistency.

**Durable decisions:**

1. **`channel` is the single source of truth + the tracking field; the `with_*_template` booleans stay as
   send-path storage and are derived from it server-side.** `store()` sets exactly one boolean true from
   `channel` and nulls the two non-selected channels' template/config ids (e-badge is gated to email). This keeps
   the `AutomationController` fan-out and `split()` — which both read the booleans — **untouched**, so the actual
   send path did not change. `split()` also carries `channel` onto chunked setups.
2. **Schema is additive forward-only.** A new `2026_07_25_000001_add_channel_to_automation_setups` migration ADDs a
   nullable `channel` string; no existing migration was edited (`migrate:fresh` remains banned). Pre-existing rows
   keep their old booleans and get `channel = null` — harmless, since automation setups are one-shot and have
   already fired.
3. **Validation is channel-driven.** `channel` is `required|in:email,sms,whatsapp`; the template ids move from
   `required_if:with_*_template,yes` to `required_if:channel,<x>`. `AutomationSetupsResources` now exposes
   `channel` (and the previously-missing `with_whatsapp_template` / `whatsapp_template_id` / `whatsapp_config_id`,
   for parity with the email/SMS fields).
4. **The admin create form is reskinned to the invitations/categories Section-card pattern.** Three grey
   toggle-boxes → a single gated **Channel** dropdown (`check-default` disables a channel with no configured
   default + an amber "Configure X →" link), each channel revealing its template + a switch-before-dropdown
   provider override; a centered `lg:col-10` layout and a sticky Cancel / Submit footer. The ~10 previously
   hardcoded English strings (paste placeholders/help, cross-check buttons, totals, "Attachments", "Custom PDF
   attachment", the with-share hint) are routed through `translate()`, with EN + AR keys added in the same change.

**Files:** backend — `2026_07_25_000001_add_channel_to_automation_setups.php`, `AutomationSetup.php`,
`AutomationSetupsController.php` (`store` + `split`), `AutomationSetupsResources.php`; admin —
`automation/automation-form.tsx`, `translations/{en,ar}/web.json` (11 new keys).

**Mobile contract:** no change. Automation-setups is admin-only (`TypeGate super`), not in `routes/api.php`'s
mobile surface — no endpoint added, removed, or renamed.

**Gates:** backend `pint --test` **passed** + `phpstan` **No errors** + `php artisan test` **468 passed** (the one
failure, `SessionsTest > search finds matching sessions`, is a pre-existing order-dependent flake — it passes
30/30 in isolation and is unrelated to automations). Admin `yarn type-check` **green**; `eslint` clean on the form.
`yarn production` was not run (no local `.env.production`; env files are gitignored and not created). EN + AR keys
in place.

**Status:** landed in the working tree, **NOT committed** — awaiting review.

## D38 — 2026-07-25 — Automation create becomes a filter-and-select guest picker (replaces paste-emails): `guest_ids` list_type + `/guests/select-ids`, built on the shared listing primitives

Admins disliked the old automation workflow (go elsewhere → filter/export guests → copy their emails
or registration numbers → paste them into the automation form). The `/automation/create` page is now
an in-app **filter → pick → run** flow: the shared guest filter panel, a checkbox table, and a "Run
automation" popup that fires on the picked guests.

**Durable decisions:**

1. **Audience contract = selected guest IDs.** The automation store gains a third `list_type`,
   `guest_ids`, resolved via `Guest::whereIn('id', …)` — additive alongside the existing `emails` /
   `registration_numbers` paths. The picker ships the resolved IDs, so what is selected is exactly
   what runs (count locked at click time). Validation is `guest_ids => required_if:list_type,guest_ids|array`
   + `guest_ids.* => uuid` (no per-row `exists` — `whereIn` drops unknowns, and per-row `exists`
   would fire one query per ID on a large select-all).
2. **"Select all matching" resolves server-side.** `GET /admin/guests/select-ids` reuses `index()`'s
   `applyFilters` + `applyAdminGuestAccessFilter` and returns `{ ids, count }`, so selecting the whole
   filtered set is identical to what the listing shows and works across pagination. Admin-only
   (`admin.can:guests_listing`), not in the mobile contract. The `store` `guest_ids` path trusts the
   submitted IDs (no re-scoping) — fine while automation-create is super-only; re-scope if it ever
   opens to restricted admins.
3. **Create page replaced — no dual path.** `/automation/create` renders the picker; the paste form
   (`automation-form.tsx`) is **deleted**. Its Name/Channel/Actions block was extracted to a shared
   `automation-settings-fields.tsx` (react-hook-form `FormProvider`) consumed by the run-automation
   modal, keeping the single-channel model + gated overrides from D37.
4. **Built on the shared listing primitives**, not a hand-rolled table: `ListingTable` +
   `ListingFooter` + an e-visa-style selection bar (checkbox `select` column, "Total selected: N of
   M", select-all/clear/primary buttons). Documented as the reusable pattern in
   [../ai/LISTING_SELECT_PATTERN.md](../ai/LISTING_SELECT_PATTERN.md) — the canonical exemplar is the
   e-visa listing.

**Files:** backend — `AutomationSetupsController@store` (guest_ids branch), `GuestsController@selectIds`,
`routes/api.php`; admin — `automation/automation-guest-picker.tsx`, `automation-run-modal.tsx`,
`automation-settings-fields.tsx`, `pages/[lang]/automation/create.tsx`, deleted `automation-form.tsx`;
docs — `ai/LISTING_SELECT_PATTERN.md` (+ `ai/README.md` pointer).

**Gates:** backend `pint --test` + `phpstan` **No errors** + `php artisan test` **469 passed**; admin
`yarn type-check` + `eslint` clean. EN + AR keys in place.

**Also in this wave:** the delivery **channel became optional** — an automation may be actions-only
(status / poster / category with no message). The modal gates the channel behind a "Notify guests"
switch and `store` accepts `channel = null` (extends D37's single-channel model). UI polish: the
picker + listing + view-more render on `ListingTable` / `ListingFooter`; the listing name links to
details and its columns regroup into Notification / Actions; and the shared `SearchGuestsBySuper`
**Reset** now clears its inputs before navigating (**P24.24** — also fixes the `/guests` page).

**Landed (committed on `dev`, NOT pushed):** backend `49ece52` (P24.23); admin `17ad5cf` (P24.23 —
picker / modal / shared-fields, paste form removed, listing regroup) + `ffb01d3` (P24.24 — Reset fix).
Docs on `main` carry this entry + `ai/LISTING_SELECT_PATTERN.md`.

## D39 — 2026-07-25 — Automation scheduling: "Send immediately" (auto-dispatch on create) or "Schedule for later"; a shared `AutomationDispatchService` + a cron-driven command; **prod must run the Laravel scheduler**

The run-automation modal gains a **Scheduling** step: *Send immediately* (default) or *Schedule for
later* + a **clean masked date + time pair** (the shared `MaskedDateInput` / `MaskedTimeInput`, same
components as the logistics forms, combined into `scheduled_at` on submit — not the native
`datetime-local`). Adapted from `120-pif-private-events-platform`'s scheduling scaffold, but rewired to
**our** multi-channel fan-out (120's dispatcher is email-only/older-pattern — not copied).

**Durable decisions:**

1. **Send/dispatch is one shared loop.** The per-guest fan-out (which of `SendAutomationEmailEvent` /
   `SendGuestSMSEvent` / `SendGuestWhatsAppEvent` / `SocialCardGenerationEvent` to fire, plus the
   status/category/poster side-effects and per-row `is_sent` idempotency) lives in
   `App\Services\AutomationDispatchService::dispatch()`. It is called by all three entry points — the
   manual "Send" button (`AutomationController@send`), the immediate-on-create path
   (`AutomationSetupsController@store`), and the scheduled command — so every path behaves identically.
   The service **fires the existing events**; it does not replace the event→queued-listener
   architecture (sending still flows event → listener → supervisor `queue:work`). It sits *above* the
   events purely to avoid duplicating the loop.
2. **"Send immediately" auto-dispatches on Create.** `store()` with `schedule_type = immediate` (or
   absent) creates the setup + rows, then dispatches inline and sets `status='started'`,
   `send_status='sent'`, `last_dispatched_at` — no second "Send" click. This is why the picker's
   primary CTA both creates AND sends.
3. **Scheduling state machine is a SEPARATE column** (`send_status`: draft | scheduled | processing |
   sent | cancelled | failed) kept apart from the legacy operation `status`. New columns are additive
   forward-only (migration `2026_07_25_000002_add_scheduling_to_automation_setups.php`): `schedule_type`,
   `scheduled_at`, `send_status`, `last_dispatched_at` + a `(send_status, scheduled_at)` index. **DB is
   source of truth** (not a delayed queue job) — that is what makes Cancel + a status column clean and
   recoverable.
4. **A cron-driven command fires due rows.** `automations:dispatch-scheduled` (scheduled `->everyMinute()
   ->withoutOverlapping()` in `App\Console\Kernel`) claims each due row atomically
   (`scheduled → processing`, guarded update, skip if not claimed), dispatches via the service, then sets
   `status='started'`, `send_status='sent'`, `last_dispatched_at`; on throw → `send_status='failed'` +
   `Log::error`.
5. **Cancel** = `POST /admin/automations-setups/{id}/cancel` (gated `admin.can:automation,update`), allowed
   **only** while `send_status='scheduled'` → `cancelled`. The listing shows a Cancel action for exactly
   those rows and a `send_status` badge column (+ the scheduled time).
6. **Timezone:** `scheduled_at` is a naive **Asia/Riyadh** datetime (app timezone) compared to
   `Carbon::now()` in the same zone — no conversion. The masked date input's `minDate` blocks past
   days; the backend re-validates `after:now`.

**⚠️ Deploy requirement (NEW dependency):** scheduled automations only fire if the **Laravel scheduler**
runs in prod — either a cron `* * * * * php artisan schedule:run`, or a supervisor program running
`php artisan schedule:work`. This is **separate** from the existing `queue:work` supervisor (which is
reused unchanged for the actual sending). Without the scheduler, *immediate* automations still work
(dispatched inline on create), but *scheduled* ones sit in `send_status='scheduled'` forever. Also run
`php artisan migrate` (the `2026_07_25_000002_*` migration) on deploy.

**Files:** backend — `AutomationDispatchService` (new), `Console/Commands/DispatchScheduledAutomations`
(new), `Console/Kernel` (schedule), `AutomationController@send` (now calls the service),
`AutomationSetupsController@store` (schedule fields + auto-dispatch) + `cancel()`, `AutomationSetup`
(fillable/casts), `AutomationSetupsResources` (4 fields), `routes/api.php` (cancel route), migration
`2026_07_25_000002_*`; admin — `automation-settings-fields.tsx` (Scheduling section + type/defaults),
`automation-run-modal.tsx` (payload), `automation-listing.tsx` (send_status badge + Cancel),
`interfaces/automation-setup.tsx`, EN + AR `web.json`.

**Gates:** backend `pint --test` **passed** + `phpstan` **No errors** + `php artisan test` **469 passed**;
admin `yarn type-check` **green** + `eslint` clean on the changed files. `yarn production` not run (no
local `.env.production`; env files are gitignored). EN + AR keys in place.

**Landed (pushed, on `dev`):** backend `38729eb` (P24.25 — service / command / Kernel / controllers / model /
resource / route / migration); admin `673f00d` (P24.25 — schedule step + send_status badge + Cancel + the
shared web.json keys, EN+AR). Docs on `main` carry this entry + HANDOFF.

**Still pending before it works end-to-end:** the migration is **applied on the dev DB** (all 4 columns
present); still needed — the **scheduler** running for scheduled sends (see the Deploy requirement above),
`php artisan migrate` on **prod** at deploy, and **manual testing** (automated gates pass, but the flow
hasn't been exercised against a real DB yet).

> **Addendum 2026-07-27 — the Deploy requirement above now has a concrete runbook.**
> [`process/QUEUE_SETUP_PROD.md`](../process/QUEUE_SETUP_PROD.md) (+ a PDF twin for DevOps) carries the
> supervisor program for `queue:work`, both scheduler options (cron `schedule:run` or a `schedule:work`
> program, `numprocs=1`), `queue:restart` after every deploy, and the verification commands. Brought over
> from the 123 clone and de-branded. The decision itself is unchanged — this only means nobody has to
> re-derive the config. **Installing it on the prod box is still outstanding.**

> **Addendum 2026-07-28 — a third scheduling option: `manual`.** Beside *immediate* and *scheduled*, an
> automation can now be saved as a **draft to run later by hand** (`schedule_type = 'manual'`,
> `send_status = 'draft'`, no dispatch on create); an admin runs it from the listing's Run action, which
> goes through the **existing** manual send endpoint `POST /automations/send/{id}` — no new endpoint, no
> new state machine. **No migration was needed:** `schedule_type` is `string(20)` in
> `2026_07_25_000002`, not an enum, so it accepts the third value as-is (don't "fix" this with a
> redundant migration). The listing's Run action is gated on `send_status === 'draft'`.
> **Parked item #2 above is still open and now has a second home:** the *details* page's Run button is
> still not gated on `send_status`, so it can dispatch a scheduled automation early. Backend `6e7fd94`,
> admin `3fc30c9` (`P031.1`) — **both pushed and merged to `main` on 2026-08-06**. Detail:
> [`tasks/031-automation-manual-run/TASK.md`](../tasks/031-automation-manual-run/TASK.md).

## D40 — 2026-07-25 — Automation details page rebuilt on the shared listing primitives (D38 pattern), Clicked column hidden, shared `sent_at` label tidied

The per-guest automation details page (`/automation/details/[id]`) was still the old NextCrazy-generated
pattern (raw `<table>` + `TableCaption`/`TablePagination` + a bespoke `automation-details-search` + manual
fetch). Rebuilt on the **same shared primitives** as the D38 listing — `useListingState` +
`ListingTable` / `ListingFilters` / `ListingFooter` + `TopSection` — so it matches the admin theme.

**Durable points:**

1. **Setup-level fields live once, not per row.** `name` + `email_template` are constant across every row
   of a setup, so they moved out of the table into a small **summary card** (name · template · total
   guests). The table is now purely per-guest: Guest (name + email) · Category · Sent (badge) · Sent at ·
   Delivered · Opened.
2. **Behaviour preserved:** the **MORE** menu (Split by 500 / Send To All), **Export**, the is_sent status
   filter (all / yes / null), and the back button all carry over unchanged. Pagination now routes through
   `useListingState`, which also **fixes the old broken `automations/details/` paginate path**.
3. **`Clicked` column hidden** for now (kept in the interface + backend + resource; a one-line comment in
   `automation-details.tsx` shows how to restore it). Delivered / Opened / Clicked are webhook-populated
   tracking flags that are usually empty.
4. **Shared `web:sent_at` EN label tidied** from the literal `"sent_at"` → `"Sent at"` (AR was already
   proper). That key is shared, so the logs / e-visa headers get the nicer label too — an intentional,
   net-positive side effect.
5. **Orphaned `automation-details-search.tsx` deleted** (only this page used it).

**Open edge (parked):** the details page still exposes the manual "Send To All" action, which would
dispatch a *scheduled* automation **early** if clicked before its time — it is not gated on
`send_status = scheduled`. Left as-is (a manual override is defensible); gate it later if unwanted.

**Also parked (raised, user chose to hold):** the listing now shows **two** near-redundant status columns —
legacy **Operation status** (`status`: not_started / started / splitted) and the new **Send status**
(`send_status`). They move in lockstep for immediate automations and only diverge for scheduled ones;
collapsing to a single column is a pending UI decision.

**Files:** admin — `automation/automation-details.tsx` (rewrite), `pages/[lang]/automation/details/[id].tsx`
(simplified to `mode` only), deleted `shared/forms/filters/automation-details-search.tsx`, `translations/{en,ar}/web.json`
(`opened`, `no_email`, `sent_at` tidy).

**Gates:** admin `yarn type-check` + `eslint` clean. (`yarn production` not run — no local `.env.production`.)

**Landed (pushed, on `dev`):** admin `903013c` (P24.26).

## D41 — 2026-07-25 — WhatsApp RBAC rules added to `inferFeatureId` (completes D33 coverage; the D34 `check:rbac` guard is now in the documented gate)

The WhatsApp channel (D36) shipped its routes + sidebar features (`whatsapp_logs`, `whatsapp_templates`,
`whatsapp_config`) but **never added the matching `inferFeatureId` rules** — so it hit the exact D33
first-match-wins trap that D33/P18.2 fixed for SMS. `check:rbac` failed with **4 errors**:
`/logs/(guest|invitation)-whatsapp` fell through to the generic `/logs` rule → `email_logs`, and
`/whatsapp/templates` + `/whatsapp/provider-configs` matched nothing → `null` (Super-Admin only).

**User impact (same as the P18.2 SMS bug):** an admin granted `whatsapp_logs` saw the link and got "not
authorized"; one granted only `email_logs` rendered the WhatsApp log page and then ate a 403; and the two
config features were effectively **ungrantable**.

**Fix (3 rules mirroring SMS, `utils/inferFeatureId.ts`):** `/logs/(guest|invitation)-whatsapp` →
`whatsapp_logs` **placed before** the generic `/logs` rule; `/whatsapp/templates` → `whatsapp_templates`
**before** the generic `/whatsapp` → `whatsapp_config`. `check:rbac` now green (0 errors, 46 → 49 rules).

**Process note (why it slipped):** `yarn check:rbac` (the D34 guard) is a **standalone script** — NOT run
by `composer qa`, `yarn type-check`, or `yarn production` — so a gate run that skips it misses this bug
class. It is now written into the documented gate: `ai/CURRENT_WORKFLOW.md` + `process/SETUP_AND_UPDATE.md`
(and the old "check:rbac not in the gate" note in `tasks/PHASE22_PARKED_TODO.md` is marked resolved). Run
it on any change touching routes / the sidebar / `inferFeatureId.ts`.

**Baseline fix:** applied here in `alt-static-basecode` (the baseline) so clones (e.g. `123-pif-pep-v2`)
catch up via `--ff-only` rather than each patching locally.

**Landed (pushed, on `dev`):** admin `fe1ac09` (P24.27).

## D42 — 2026-07-25 — Seating Plan Manager ported into the baseline as a 4th sub-app (`alt-static-basecode-seating`), API-wire, RBAC-graded

**What:** brought the standalone **Seating Plan Manager** — a Vite/React SPA the client demoed for a month in
v1 (`120-pif-private-events-platform`) — into the 121 baseline as a **new 4th sub-app**, so every clone
(immediate consumer `123-pif-pep-v2`) inherits it. It integrates **API-wire** (no UI merge): signs in as an
admin, polls guests, writes seat + attendance back onto the guests table, stores the board in `seating_layouts`.

**Decisions (promoted from `tasks/021-seating/TASK.md` D-1…D-8):**
- **D-1/D-2:** build in the **baseline** (not the clone); a standalone sub-app — retarget its API client, don't
  rebuild in Next (proven, decoupled, canvas-heavy; SSR buys nothing).
- **D-3 attendance:** 6 new shared columns (`checked_in_at/by`, `checked_out_at/by`, `food_checked_in_at/by`);
  a `Guest::booted()` bridge keeps the legacy `check_in` boolean in sync. "One-truth" cleanup deferred.
- **D-4 handoff:** the SPA uses its **own login** (no token handoff — the admin bearer is HttpOnly, D12); the
  admin side is a gated deep-link. One-click SSO (a minted scoped token) is a possible follow-up.
- **D-5 RBAC:** a dedicated **`seating`** feature with graded actions **`view`/`check_in`/`manage`**, mapped to
  the SPA's existing 11-permission tiers via its one `mapAdminToUser` function. Enforced **both** server
  (`admin.can:seating,<action>`) and client — v1 enforced tiers client-side only. Never `admins.type`.
- **D-6:** full port incl. `seating_layout_versions` + `seating_audit_logs`.
- **D-7:** keep the Vite/JS/Tailwind-v3 SPA as-is (own build chain). **D-8:** repo `alt-static-basecode-seating`.

**Migrations:** additive, forward-only at `2026_07_25_000005..000008` (000001–000004 were taken by automation +
task 020). Seat/attendance **writes ride** `PUT /admin/guests/{id}` (`guests_listing,edit`) — so a write-capable
seating operator also needs that grant; a dedicated seating-scoped write endpoint is a possible follow-up.

**Mobile contract:** new routes are **admin-only** (`/admin/seating-layout*`, `/admin/seating-audit-log`,
`/admin/me`) + 6 additive `GuestsResources` fields → additive only, nothing removed/renamed.

**Gates:** backend `pint --test` clean + `phpstan` No-errors + `php artisan test` **480 pass** (6 new); admin
`type-check` clean + `check:rbac` OK; seating `npm run build` clean.

**Status — MERGED + PUSHED (2026-07-26).** Backend + admin `feat/seating` merged to `dev` (`076cb8d` / `0f34b2d`);
docs on `main`. **SPA re-based (addendum):** the initial fork was stale (`d145e13`); it was rebuilt on the current
120 upstream **`85fecfb`** — the version the client actually tested (dark-mode theming, room presets, table-shape
geometry, a SyncStatus indicator; 3 backend-less modals dropped) — keeping the **full 120 git history** so future
upstream updates can be pulled, with the retarget re-applied (`guests/{id}` + `permissions.seating`). Published to
`eissa-alt/alt-static-basecode-seating` (`dev`+`main` `09d23a5`), `npm run build` clean; 123 `pif-pep-v2-seating`
re-cloned to match. **Backend + admin needed NO change** — the `85fecfb` API contract equals what was built (the
D-6 full port already covered the `seating-audit-log` endpoints the newer SPA calls). **Pending owner:** set `.env`
(`CORS_ALLOWED_ORIGINS`, `NEXT_PUBLIC_SEATING_MANAGER_URL`, seating `VITE_*`); `php artisan migrate`; live QA.

> **Addendum 2026-07-28 — the admin launch button described above no longer renders.** The
> `seating`-gated "Seating Manager" deep-link was removed from the guests-listing toolbar (admin
> `5b19580`, `P025.1`). **Everything behind it is intact** — the `seating` RBAC feature, the
> `alt-static-basecode-seating` sub-app, the backend `/admin/seating-*` endpoints, the
> `OpenSeatingManagerButton` component, its EN/AR labels, and `NEXT_PUBLIC_SEATING_MANAGER_URL` — so
> re-adding the button is a one-line change and the SPA is still reachable by its own URL under its own
> login. Read this decision as "seating is integrated, but not advertised from the admin."
> Detail: [`tasks/025-hide-seating-link/TASK.md`](../tasks/025-hide-seating-link/TASK.md).

## D43 — 2026-07-27 — Guest history records WHAT an edit changed — as a redacted delta, never a snapshot

**What:** `history_logs` rows gained `previous_payload` / `payload` (additive migration
`2026_07_27_000001`), filled in `GuestsController::updateGuest`, and the admin renders them as a
`Field | From | To` table in the guest see-more modal. Prompted by a comparison against
`115-cyan-basecode`, which had the richer version first (`2026_05_06_000004_extend_history_logs_for_payload`).

**Durable decisions:**

1. **A DELTA, never a snapshot — this is the load-bearing one.** Only fields that actually changed are
   stored. Cyan snapshots the whole record on both sides, but it can afford to: cyan's guests table has
   **no PII columns at all** (everything is one `form_data` JSON blob) and its uploads sit on the
   **public** disk. 121 is the opposite on both counts — ~110 fillable columns including `religion`,
   `birth_date`, `document_number`, `full_name_on_document`, plus five **private-disk** file fields that
   exist because of D14. A verbatim port would have made `history_logs` the largest uncontrolled PII
   store in the database, rewritten on every edit, with **no erasure path** (`guest_id` has no
   `onDelete`). It is also a volume problem independent of privacy: the **Seating Plan Manager writes
   every seat drag through `PUT /admin/guests/{id}`**, the same handler that logs — a full snapshot
   would persist a passport number twice for a seat move. **Do not "complete" this feature by adding
   snapshots.**
2. **File fields record only THAT they changed** (`[file]` sentinel), never the value. `personal_image`,
   `document_copy`, `visa_copy`, `issued_visa` are private-disk paths served exclusively through signed
   URLs (D14); writing the path into a plain json column routes around that control.
3. **`[]` and `null` mean different things, and the distinction survives to the client.** `[]` = the save
   ran and changed nothing → the UI says "No changes in edit". `null` = unknown — a row written before
   this feature, or an action that never carried a field edit → the UI keeps the plain one-line format.
   There is **no backfill** (the old values were never recorded), so the list stays permanently mixed.
4. **Cyan's `source` / `actor_type` / `ip` / `user_agent` were deliberately NOT ported.** `source`
   collides with the existing `guests.source` column; `actor_type` invites confusion with the retired
   `admins.type`; and cyan never renders `ip` nor even exposes `user_agent` — speculative storage.
   Capturing registrant IP on the public `/complete-data` / `/reconfirm` routes is a privacy-notice
   decision, not a code one.
5. **The dynamic-forms coupling is real but shallow, and it is in the producer only.** Cyan's admin diff
   renderer walks a `Record<string, unknown>` and prints raw keys — zero `FormDefinition` coupling, so it
   ports. Only `'payload' => $formDataNormalizer->normalize($form, …)` is coupled, and totally. So
   CLAUDE.md hard-rule 4 is **not** engaged by this port: the schema and the UI came across, the payload
   producer was re-authored against 121's static columns.
6. **Scope: one write site.** Only `updateGuest` carries a payload. The other 27 `HistoryLog::create`
   sites are state transitions (`Badge Printed`, `Hotel Assigned`, …) where the action name says
   everything.

**Not yet true:** this is **not** a complete audit trail. `importGuestsExcel`, the `upload*` methods,
`attend()` / `guestsSyncOffline()`, `reGenerateSMP()` and `MobileAuthController::updateProfile` (which
mutates guest phone and avatar) write **no history row at all**. Describe the feature accordingly, or
scope a follow-up.

**Also fixed en route:** the history view rendered an Email/First/Last table that looked like part of the
log but was read from react-hook-form — the guest's values *right now*, unrelated to any row. Deleted.
And the view now lives in the see-more modal: the `/extra/{id}` page it used to live on is reachable by
direct URL only, and resolves to feature-level `guests_listing` while the API requires
`guests_listing,see_more` — so the modal is both discoverable and the correct RBAC host.

**Landed (on `dev`, not pushed):** backend `0df228e` (P022.1); admin `a959703` (P022.2) + `5ae2afb`
(P022.3). Gates: pint + phpstan clean, **484 tests** (4 new — `history_logs` had zero coverage before),
admin `type-check` + `eslint` + `check:rbac` green, EN/AR 1760/1760. Dev DB migrated; **prod
`php artisan migrate` still pending**. Task log: `tasks/022-guest-history-payload/TASK.md`.

## D44 — 2026-07-27 — Single-record guest reads are scoped to the admin's categories/statuses

**What:** `applyAdminGuestAccessFilter` had always scoped guest **list** queries, but the single-record
reads had no equivalent — `GET /admin/guests/{id}` and `GET /admin/history-logs/{id}` returned any guest
to any admin holding the UUID, ignoring the categories/statuses that admin is bound to. Both are gated
`guests_listing,see_more`, so the caller is an admin — but one reading a guest they cannot see in any
listing. Task 022 (D43) sharpened the stakes by putting a field-level edit diff behind the history
endpoint, but the leak is the guest record itself via `show()`; the diff is incremental exposure on a
hole that predated it.

**Durable decisions:**

1. New `App\Support\GuestAccessScope::denies()` is a **single-record twin** of the list filter and must
   mirror it exactly: super bypasses; an admin with **no** restrictions set sees nothing (the list
   filter answers that case with `0 = 1`, not "everything"); category and status bounds are ANDed; `null`
   is permitted in the status list for the admin UI's "No-Value" option. Shape follows
   `GatesController::deniesGateScope`. **If either side's rules change, change both.**
2. **Behaviour change:** a scoped admin who could previously open any guest by UUID now gets **403**. The
   listing never showed them those guests, so no legitimate flow should depend on it.
3. **Not covered, deliberately:** `GuestDraftsController::show` has the same shape and `guest_drafts`
   does carry `category_id`, but it is a separate RBAC feature and whether drafts should be
   category-scoped at all is undecided.

**Landed:** backend `ae1c210` (`P023.1`) — committed + **pushed** to `origin/dev`. Gates: pint + phpstan
clean, **489 tests** (5 new). Not mobile-facing. Task log: `tasks/023-guest-access-scope/TASK.md`.

## D45 — 2026-07-27 — Admins get a phone number, named `phone` to match guests

**What:** admins gained a phone number, captured on create/edit and shown in the listing. Additive,
forward-only migration `2026_07_27_000002` adds a nullable `phone` (`string(100)`) to `admins`; exposed
via `AdminsResources` (login + `/me`) and the flat `AuthController::profile()`.

**Durable decisions:**

1. **Column is `phone`, not `phone_number`** — guests use `phone` everywhere (DB, API, forms), so admins
   match it for consistency (owner's explicit call). A clone that renames this field must rename it on
   both entities together.
2. **DB column nullable; "required" enforced at the request/form layer.** Existing admins predate the
   field and `migrate:fresh` is banned (real data), so a NOT-NULL column is impossible. `store` validates
   `required`; `update` validates `sometimes|required`. Owner chose required on create **and** edit — so
   editing a pre-existing admin now forces entering a phone before it can be saved.
3. **Widget = `PhoneInputV2`** (the guest phone widget: country flags + libphonenumber E.164), not the
   plain `CustomInput` the other admin fields use — consistency + real validation.
4. **Deferred, not dropped:** a phone **search/filter** on the admins listing (needs a backend `index`
   filter) and an **admins export** (none exists today).

**Also (same admin commit, owner request):** the create form now defaults **status ON** and the **Data
scope** section **expanded** (edit mode unaffected — `reset()` loads the record). And a shared-component
fix: `PhoneInputV2`'s error text used `text-error`, which is **not a defined color** in the admin's
Tailwind v4 theme, so validation messages rendered gray — switched to `text-red-500`; this also corrects
the guest admin forms, which share the component.

**Landed:** backend `f1c3dc3` (`P024.1`), admin `f743fae` (`P024.2`) — committed + **pushed**. Gates:
pint + phpstan clean, 489 tests; admin `type-check` + eslint green; EN/AR reused (`web:phone`,
`validation:invalid_phone`, no new keys). Dev DB migrated; prod `migrate` + manual QA pending. Not
mobile-facing (`/admin/admins`, `routes/api.php` untouched). Task log: `tasks/024-admin-phone/TASK.md`.

## D46 — 2026-07-28 — Cloning a category deep-copies it: `replicate()` for columns, plus relations and its own poster files

**What:** `CategoriesController::clone` was producing a partial copy — the admin had to re-enter settings
by hand on every clone. Rewritten to copy everything a category owns, inside one `DB::transaction`.

**Durable decisions:**

1. **`replicate()`, never `getAttributes()` + `fill()`.** The old path ran the copied attributes through
   `fill()`, so **any column missing from `$fillable` was silently dropped** — and would keep being
   dropped for every column added in future. `replicate()` copies every column regardless of `$fillable`
   and resets id/timestamps. This is the part that must survive: a clone must not depend on `$fillable`
   being complete.
2. **Relations are copied explicitly, in the transaction:** badge assignments (`badge_category` pivot,
   via the Eloquent relation), admin data scope (`admins.category_ids` — a JSON column, not a pivot, so
   it reuses the existing `adminsWithCategory()` / `syncCategoryAdmins()` helpers), and meeting-room
   links (`meeting_room_categories` — no Eloquent relation on `Category`, so rows are inserted directly).
   **Adding a new category relation means adding it here too** — there is no generic mechanism.
3. **Share-poster files are duplicated, never shared.** `duplicatePoster()` copies each poster to a fresh
   random basename on the public disk, so editing or deleting one category's poster can never affect the
   other; `share_poster_url` is recomputed to match `CategoriesResources`. A missing source file returns
   the original name unchanged rather than failing the clone.
4. **Consequence to know before cloning:** copying the admin data scope means the clone is visible to
   every admin who could see the original — it inherits its source's visibility rather than starting
   private. That is the intended meaning of "a copy of this category", but it is a real access-surface
   change on a wide-scoped source.

**Unchanged:** the slug/name copy behaviour (`{slug}-copy`, `-copy-1`, …, and `(Copy)` / `(نسخة)`
appended to the names) and the `titles.cat_list` update.

**Landed:** backend `0be944e` (`P027.1`), +94/−41 — committed + **pushed**. Gates: pint clean at commit
time via the pre-commit hook; **`composer qa` has not been re-run since**. **No automated test covers
`clone`** — add one alongside the browser QA. Not mobile-facing (`/admin/categories`, `routes/api.php`
untouched). Task log: [`tasks/027-category-clone-fix/TASK.md`](../tasks/027-category-clone-fix/TASK.md).

## D47 — 2026-07-28 — Guest-listing row actions are gated per category, defaulting OFF

**What:** each category now decides which of the five guest-listing row actions its guests expose —
resend email / SMS / WhatsApp, print badge, mark collected — and the issued-visa email template sits
behind its own `with_issued_visa` toggle instead of being inferred from "a template is selected". Two
additive, forward-only migrations: `2026_07_28_000001` (the five row-action booleans) and
`2026_07_28_000002` (`with_issued_visa`).

**Durable decisions:**

1. **The category switch is an ADDITIONAL gate on top of RBAC, not a replacement.** An action renders
   only when the category switch **and** `checkActionPermission(...)` both pass. Neither one alone.
2. **⚠️ The five row-action toggles default to `false` and are NOT backfilled.** Existing categories opt
   in explicitly. **Operational consequence: the moment this migration runs, every guest row action
   disappears from every category until an admin turns it back on — including *print badge*.** Turn the
   switches on per category immediately after the prod migrate, before any on-site use. The dev DB is
   already migrated; **prod is not.**
3. **`with_issued_visa` IS backfilled** (`update … where issued_visa_email is not null`). The asymmetry
   with #2 is deliberate: the old issued-visa behaviour was itself derived from a selected template, so a
   backfill reproduces it exactly; the row actions had no equivalent signal to derive from.
4. **A switched-on comms action stays visible but disabled when its provider isn't configured**, with a
   `web:provider_not_configured` tooltip — so an admin can tell "not allowed for this category" from
   "not set up yet". Readiness comes from the three existing `*/check-default` endpoints, mirroring the
   categories and automation forms.
5. **The listing reads the switches off the guest row's embedded category** (`row.category?.with_*`).
   `GuestsResources` serialises the whole `category` model, so the new columns ride along with no
   resource change — but that also means **the gating depends on `category` staying eagerly loaded** on
   the guests index. Dropping that eager load would hide every row action.

**Landed:** backend `072dff2` + admin `718bcf7` (both `P028.1`) — committed + **pushed**. EN + AR in the
same commit (11 keys each). Gates: pre-commit hooks green at commit time; **full gate not re-run since**.
Dev DB migrated; **prod `migrate` + per-category switch-on + manual QA pending.** Not mobile-facing
(`/admin/*`, `routes/api.php` untouched). Task log:
[`tasks/028-category-guest-action-gates/TASK.md`](../tasks/028-category-guest-action-gates/TASK.md).

## D48 — 2026-09-21 — Deals are asked of guests by a per-guest link, not registered through a category

**What:** the Deals Signing Ceremony stopped being a category of its own (client decision). The
private category's guests are sent a single-use link from a **Deal** action in the guest listing,
the same shape as complete-data here and speaker onboarding on Tourise. The deal is stored against
that guest (`deals.guest_id`). Task log: [`tasks/041-deals-per-guest-link/TASK.md`](../tasks/041-deals-per-guest-link/TASK.md).

**Durable decisions:**

1. **One entry point.** `POST /api/deals` and the `psf-deals-2027` form shape are gone from both
   front ends. A deal arrives only through `POST /api/complete-deal` with a live `deal_tokens`
   token. Do not re-add a category route "for convenience": it would bypass the guest link.
2. **The token is the identity.** The form never asks who the guest is. The token is burned
   inside the same transaction as the write, by a conditional update, so a failed validation
   keeps the link usable and a double submit cannot both succeed. Re-sending the link mints a
   fresh token, which is how the team asks for a second round.
3. **Gated per category and per role.** The action shows only where `with_request_deal` is on,
   sends the category's `deal_email`, and needs the `guests_listing` `notify` action. The endpoint
   re-checks the switch server-side.
4. **An agreed point of contact is the guest.** With `contact_agreed: yes` the API stores
   `guests.email`, whatever is posted. The page shows it locked (read-only, still submitted),
   and skips the company-domain rule for it, since the private category allows personal
   addresses.
5. **The old category is switched off, never deleted.** Migration
   `2026_09_21_000004_retire_deals_categories` deactivates any category on the retired shape.
   It finds it **by shape, not slug** (it existed under more than one). Deletion is blocked by
   `RESTRICT` foreign keys from `invitations` and `invitation_collections`, and the admin has no
   category delete.

**Landed:** backend `1ec18c6` + `08ff1c4`, admin `30c143e`, frontend `60458a5` + `269d351`, on
`main`. Not mobile-facing (`docs/mobile/` has no deal routes).

## D49 — 2026-09-21 — Deal signing slots: one agreement per slot, held from submission, confirmed or released by the review

**What:** each agreement books a signing slot the team opened, instead of typing a time. Modelled
on GFAI's bilateral booking. Task log: [`tasks/042-deal-signing-slots/TASK.md`](../tasks/042-deal-signing-slots/TASK.md).

**Durable decisions:**

1. **A slot per agreement, one agreement per slot.** A signing stage holds one ceremony at a
   time. Venues exist so a second stage is a row, not code. There is no capacity or party-size
   matching, unlike GFAI rooms.
2. **Held from submission.** `Deal::HOLDING_STATUSES = [pending, accepted]`. A pending deal takes
   its slots off every other guest's page. Rejected and returned **release** them. The cost is
   that an unreviewed deal blocks its slots, so review the pending ones promptly.
3. **Booking is serialised in the database, not the page.** The slot row is locked, and the
   "still free?" check is **itself a locking read**. A plain read under MySQL's REPEATABLE READ
   is served from the transaction's snapshot and misses the other guest's insert; this was
   found on GFAI. A lost slot rolls back the whole submission, token included, with
   `409 slot_taken` and the agreement's index. Verified on MySQL, since SQLite ignores locks.
4. **Slots copy the catalogue's times.** Editing a time-slot window never moves an applied slot.
   Moving a booked slot is allowed, but **does not email the guest**, as on GFAI.
5. **Re-accepting re-takes slots and can fail.** Accepting a rejected or returned deal re-checks
   each slot under the same locks, and refuses (409) if another deal took one since.
6. **Deleting cannot strip a booking.** `deal_items.slot_id` is null-on-delete, never cascade,
   and the admin refuses to delete a held slot or a venue with held slots.
7. **The decision is the confirmation.** Accept and reject are saved only from the confirm
   window, which shows where and when every agreement is booked. They email from the templates
   in Deals > Settings; a step with no template sends nothing.
8. **No new RBAC.** Scheduling rides on the `deals` feature: reading on `view`, every change on
   `schedule`, deciding on `review`. The Delete row action was removed at client request; the
   endpoint remains behind `delete`.

**Landed:** backend `b3eb2a5` + `4ba6e76`, admin `34a00f8` + `08212f4`, frontend `9701903` +
`3ea1813`, on `main`. The migration prefix `2026_09_22_000001` is a day ahead of the work; it
cannot be renamed, since it has already run locally. ⚠️ Production needs slots opened before any
deal link is sent.

## D50 — 2026-09-20 — Registration photo review runs in the browser and advises; the server rule is off by default

**What:** the client asked for an AI photo review because the existing one was too strict. The
existing one was `WhiteBackgroundImage`, a border-sampling heuristic and not AI. The review now
runs in the browser (Tourise's face-api face check) and **advises only**. Task log:
[`tasks/040-registration-feedback-2026-09/TASK.md`](../tasks/040-registration-feedback-2026-09/TASK.md).

**Durable decisions:**

1. **Never block on the photo check.** The photo always uploads. A failing check (no face, more
   than one face, face too small, sunglasses) leaves a note and a toast. Face detection's false
   negatives fall unevenly (niqab, headscarf shading the brow, dark rooms, turned faces), so
   blocking would strand exactly those guests. The review of badge photos before printing is
   now the real backstop.
2. **A detector that cannot load says nothing.** It never produces a warning the guest cannot
   act on.
3. **The server rule is pluggable and OFF.** `App\Rules\AcceptablePhoto` behind
   `PHOTO_REVIEW_DRIVER` = `off` (default) | `azure` | `local`. The Azure AI Vision driver
   (people detection, then the background measured outside the person) is built but dormant:
   no subscription exists. It falls back to `local` without credentials, and to accept on any
   API failure.

**Landed:** frontend `674b446`, backend `8868788`, on `main`.

