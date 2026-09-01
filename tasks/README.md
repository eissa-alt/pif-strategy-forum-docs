# Tasks

The work-log axis for this **task-based** repo (alt-static-basecode is a reusable clone baseline,
not sprint-cadenced — so we track discrete tasks, not sprints). One folder per task; each holds a
single `TASK.md`. The team follows progress here.

## Layout

```
tasks/
├── README.md            ← this index
├── _TEMPLATE/TASK.md    ← copy to start a new task
└── NNN-short-slug/      ← one folder per task
    └── TASK.md
```

## How to open a task

```bash
cp -r _TEMPLATE NNN-short-slug      # NNN = next zero-padded number
# fill in TASK.md: goal, scope, status, log, DoD
```

- **`NNN`** is a zero-padded running number (`001`, `002`, …). Don't reuse numbers.
- **`short-slug`** is kebab-case, a few words (`001-docs-reorg`, `002-email-editor`).
- Keep `TASK.md` current as you work — the **Status** line + **Log** are the source of truth
  for "where is this task." When done, set Status to `done` and fill the DoD checklist.
- Decisions that outlive the task → promote them to [../decisions/LEDGER.md](../decisions/LEDGER.md)
  (don't bury a durable decision inside a closed task).
- Code commits still follow the repo convention: `P<phase>.<task> — <short imperative>` on `dev`
  in the relevant sub-app repo. This folder is the **narrative log**, the git history is the diff.

## Index

| # | Task | Status |
|---|---|---|
| 001 | [boolean-db-cleanup](001-boolean-db-cleanup/TASK.md) | in-progress |
| 002 | [datetime-db-cleanup](002-datetime-db-cleanup/TASK.md) | done |
| 003 | [backend-tooling-chain](003-backend-tooling-chain/TASK.md) | done |
| 004 | [migration-squash](004-migration-squash/TASK.md) | dropped |
| 005 | [admin-httponly-token](005-admin-httponly-token/TASK.md) | done (code) — QA pending |
| 006 | [private-document-storage](006-private-document-storage/TASK.md) | done (code) — mobile ack + QA pending |
| 007 | [rsvp-decline-not-interested](007-rsvp-decline-not-interested/TASK.md) | done (code) — QA pending |
| 008 | [guest-drafts-port](008-guest-drafts-port/TASK.md) | done (code) — QA'd; ledger D19 |
| 009 | [usefetch-adoption](009-usefetch-adoption/TASK.md) | done — standing convention (closed 2026-07-19) |
| 010 | [api-routes-cleanup](010-api-routes-cleanup/TASK.md) | done — cleanup + reorg + RESTful rename (cross-repo cutover) + whereUuid + RBAC gating (Tiers 0–4); closed + bulk-image leftovers folded in 2026-07-20, ledger D24 |
| 011 | [scan-into-admin](011-scan-into-admin/TASK.md) | done (code) — gate scanning ported into admin, standalone scanner retired, new `scanning` RBAC feature; live browser-QA pending |
| 012 | [linkedin-auto-post](012-linkedin-auto-post/TASK.md) | done (code) — per-category LinkedIn automatic "Share on LinkedIn" completed (backend + admin + frontend); ledger D25; LinkedIn-app + browser QA pending |
| 013 | [sms-provider-config](013-sms-provider-config/TASK.md) | done (code) — DB-driven SMS provider config ("SMS SMTP") ported from cyan (backend + admin), listener rewired off env, `services.unifonic` removed; ledger D26; prod send-test QA pending |
| 014 | [otp-sms-dynamic-config](014-otp-sms-dynamic-config/TASK.md) | done (code) — deleted the hardcoded FGC OTP gateway (+ committed creds) from `AuthController`; phone-OTP now uses the dynamic `SmsSender`/`SmsProviderConfig` stack; added SMS mirror of the D27 pickers (category `sms_config_id` + `otp_sms_config_id`) + `sms-provider-configs/select`; gates green; manual QA pending |
| 015 | [per-flow-smtp-override](015-per-flow-smtp-override/TASK.md) | done (code) — admins can override the default SMTP per flow: category has two pickers (notifications register/accept/reject + guest email-OTP), + automations + invitations; `applyConfigById` resolver, snapshot at create, override beats `MAIL_HOST_BULK`; gates green; manual QA pending |
| 016 | [sms-flow-parity](016-sms-flow-parity/TASK.md) | done (code) — SMS now fires on accept/reject (reuses `guest_sms`), automations (`with_sms_template` toggle), and invitations (new `invitation_sms` table + listener), each with its own SMS-provider override picker; migrations `000006`+`000007`; ledger D29; gates green; manual QA pending |
| 017 | [single-channel-invitations](017-single-channel-invitations/TASK.md) | done (code) — an invitation collection now sends on exactly one channel (email\|sms; whatsapp reserved); `channel` enum folded into `000006`, scoped template/provider, channel-aware guard + status, channel picker + observability; **reverses the parallel-send half of D29**; ledger D30; gates green; manual QA pending |
| 018 | [sms-logs](018-sms-logs/TASK.md) | done (code) — read-only guest + invitation SMS log pages mirroring the email logs, new `sms_logs` RBAC feature (view/export); ledger D31; gates green; manual QA pending |

| 019 | [logistics-evisa-port](019-logistics-evisa-port/TASK.md) | done (code) — re-added hotels/rooms, traveling-status, per-guest logistics + 4 exports, and e-visa generation/PDF/console, all modernized to Tasks 001/002/009/010; found and fixed 6 pre-existing defects incl. `valid_visa` being silently discarded on every registration; February's e-visa ops console deliberately NOT ported (dead on hci main); ledger D32; gates green; `sendIssuedVisa` + manual QA pending |
| 020 | [reconfirmation](020-reconfirmation/TASK.md) | done (code) — guest attendance reconfirmation ("second RSVP"), built on 121's own machinery (not a 120 port): `reconfirmed_*` columns + `reconfirmation_tokens`, public `/reconfirm` page, `{{ reconfirmation_url }}` across email/SMS/WhatsApp via shared `ReconfirmationLink`, admin column/filter/see-more/exports; delivered by the D38/D39 automation; gates green (backend 474 tests, admin/frontend type-check); **committed + pushed** (backend `27b764f`+`045223a`, admin `e12d548`, frontend `854b44d`); dev-DB migrate + manual QA + mobile notice pending |
| 021 | [seating](021-seating/TASK.md) | done — Seating Plan Manager (Vite/React SPA from 120) integrated as a **4th sub-app** `alt-static-basecode-seating` (API-wire, no UI merge): backend schema/endpoints + dedicated `seating` RBAC feature (view/check_in/manage) + `Guest` attendance bridge, admin deep-link launch button; SPA **re-based to the current upstream `85fecfb`** (full 120 history kept) + retargeted (`guests/{id}` + `permissions.seating`); ledger D42; gates green (backend 480 tests, admin type-check/check:rbac, seating build). **Merged + pushed** (backend `076cb8d`, admin `0f34b2d`, seating `09d23a5`, docs `main`); 123 re-cloned. Pending owner: `.env` + `php artisan migrate` + live QA |

| 022 | [guest-history-payload](022-guest-history-payload/TASK.md) | done (code) — guest history now records **what** an edit changed: `previous_payload`/`payload` on `history_logs` (migration `2026_07_27_000001`), captured in `updateGuest`, rendered as a `Field \| From \| To` diff with a "No changes in edit" note, surfaced as a **see-more tab** (the `/extra/{id}` page was URL-only). A **delta, never a snapshot** — a full snapshot would copy passport-grade PII into the log on every seat drag, since the Seating SPA writes through the same endpoint; file fields record only that they changed (D14). Ledger D43; gates green (backend 484 tests, admin type-check/eslint/check:rbac, EN+AR 1760/1760); committed + **pushed** (`0df228e`, `a959703`, `5ae2afb`); dev DB migrated, prod migrate pending |

| 023 | [guest-access-scope](023-guest-access-scope/TASK.md) | done — single-record guest reads (`GET /admin/guests/{id}` + `GET /admin/history-logs/{id}`) now scoped to the admin's categories/statuses via new `App\Support\GuestAccessScope::denies()` (a twin of the list filter); a scoped admin opening an out-of-scope guest by UUID now gets 403. Closes a hole Task 022 widened. Ledger D44; gates green (pint + phpstan + 489 tests, 5 new); **pushed** (`ae1c210`); RBAC-matrix QA pending |
| 024 | [admin-phone](024-admin-phone/TASK.md) | done (code) — `phone` field for admins (same name + `PhoneInputV2` widget as guests, not `phone_number`): additive migration `2026_07_27_000002` (nullable; required at request/form layer), `AdminsResources` + `profile()`, create/edit form field, listing column. Also: create-form status defaults ON + Data-scope starts expanded, and a shared `PhoneInputV2` fix (error text `text-error`→`text-red-500`, undefined color in Tailwind v4). Phone search + admins export **deferred**. Ledger D45; gates green (backend 489 tests, admin type-check/eslint); committed + **pushed** (backend `f1c3dc3`, admin `f743fae`); dev DB migrated, prod migrate + manual QA pending |

| 025 | [hide-seating-link](025-hide-seating-link/TASK.md) | done — removed the `seating`-gated "Seating Manager" deep-link from the guests-listing toolbar. Pure UI removal: the RBAC feature, the seating sub-app, the `OpenSeatingManagerButton` component, its EN/AR labels and `NEXT_PUBLIC_SEATING_MANAGER_URL` all stay, so re-adding is one line. Partially reverses the "admin launch button" half of D42 (addendum recorded there). Admin `5b19580` |
| 026 | [guest-created-date-filter](026-guest-created-date-filter/TASK.md) | done (code) — filter guests by registration-date range. Bounds (`created_from`/`created_to`) go into the **shared `applyFilters()`**, so the listing *and* every guests export honour them; `whereDate` keeps both bounds inclusive + date-only, either alone is valid. Admin adds a shared `date-range-input.tsx` (react-day-picker range mode) emitting `YYYY-MM-DD` straight into the query params. Backend `6abee02`, admin `6978795`; browser QA pending |
| 027 | [category-clone-fix](027-category-clone-fix/TASK.md) | done (code) — "clone category" now produces a real copy: `replicate()` instead of `getAttributes()`+`fill()` (which **silently dropped any non-`$fillable` column**, present or future), all inside one transaction, plus badge assignments, admin data scope (`admins.category_ids`), meeting-room links, and share-poster files **duplicated** to fresh names so the clone never shares an image. Ledger D46. Backend `0be944e`; no test covers `clone`; browser QA pending |
| 028 | [category-guest-action-gates](028-category-guest-action-gates/TASK.md) | done (code) — per-category switches for the five guest-listing row actions (resend email/SMS/WhatsApp, print badge, mark collected) + `with_issued_visa` gating the issued-visa template. Migrations `2026_07_28_000001`+`000002`. ⚠️ **The five row-action toggles default OFF and are not backfilled — on prod migrate every row action disappears until each category opts in** (`with_issued_visa` *is* backfilled). Comms actions stay visible-but-disabled when no provider is configured. Ledger D47. Backend `072dff2`, admin `718bcf7`; dev DB migrated, **prod migrate + QA pending** |
| 029 | [title-switch-coercion](029-title-switch-coercion/TASK.md) | done (code) — creating a title failed when its optional toggles were untouched: `show_in_badge`/`show_in_user_form` are NOT NULL, and an unchecked switch arrived as `null`. Coerced with `$request->boolean()` in `store` **and** `update` (same fix `is_active` already had); `order` defaults to `0`, `allowed_genders` to `[]`. Backend `7b23178`; no test covers title CRUD; QA pending |
| 030 | [automation-status-validation](030-automation-status-validation/TASK.md) | done (code) — automation creation was blocked unless a guest status was picked: `guest_status_id` had no `nullable` (so `uuid\|exists` ran against `null`) **and** its `required_if` matched the string `'yes'` while the form sends a real boolean, so it never fired. Now `nullable\|required_if:change_guest_status,true\|uuid\|exists:…`. Backend `5e40445`; QA pending |
| 031 | [automation-manual-run](031-automation-manual-run/TASK.md) | done (code) — third scheduling option beside D39's immediate/scheduled: **manual** (`schedule_type='manual'`, `send_status='draft'`, not dispatched on create) run later via the existing `POST /automations/send/{id}`. No migration needed (`schedule_type` is `string(20)`, not an enum). Adds a draft-gated Run action + `ConfirmModal`, direct Run/Split buttons on details, and run-neutral wording (admin `af2a24a`, also stamped P027.1). D39 addendum. Pushed + merged to `main` 2026-08-06 (backend `6e7fd94`, admin `3fc30c9`); D39 parked item #2 still open — the *details* Run is still not gated on `send_status` |
| 032 | [email-reconfirmation-var](032-email-reconfirmation-var/TASK.md) | done (code) — `{{ reconfirmation_url }}` added to the email editor's `GUEST_VARIABLES` palette. The backend resolver already substituted it (Task 020); it just wasn't insertable. One line. Pushed + merged to `main` 2026-08-06 (admin `77cea97`) |
| 033 | [fork-port-back](033-fork-port-back/TASK.md) | **in-progress** — port back what the three clones (`122-gfeai-v2`, `123-pif-pep-v2`, `124-ewc-2026-v2`) fixed. Three-pass audit done and committed (docs `7f049d3` + `212d58a`); **nothing ported yet.** Working list = [FORK_PORT_BACK_FINDINGS.md](../upgrades/FORK_PORT_BACK_FINDINGS.md): ~60 verified items / 87 checkboxes, 17 of them one-liners. ⚠️ Binding process: **one item at a time, no commit until the owner reviews the diff.** Top risks: multi-use invitation links die after the first registration; the tokenized complete-data link creates a duplicate guest in 2 of 3 shapes; `GET /admin/guest-data-offline` 500s on every call. Also records baseline defects no fork fixes (2 public forced-500 test hooks, unthrottled public `/pdf/{id}`, cross-client storage buckets on live PDF paths) |

| 034 | [psf-2027-registration-shapes](034-psf-2027-registration-shapes/TASK.md) | **in-progress** — the client sheet `PSF 2027 Registration Sheet Requirement_13082026` turned into two guest form shapes: `psf-private-2027` (1 step, **no migration** — every field already existed) and `psf-public-2027` (2 steps, **6 new columns**: `position_level`, `ecosystems` json capped at 2, `attendance_purpose` + `_other`, `pif_communications_consent`, `company_profile`). Full vertical: public forms, admin create/edit, see-more panels, `GuestsExport` (**appended** — the class formats by hardcoded column letter), seeder catalogue, EN+AR. New shapes live in a sibling `froms/psf/` subtree; `default/` untouched. Also widened uploads to accept **PDF** for the company profile — every path was jpg/png-only, via a new `Base64Document` rule + private-disk-only `storeBase64Document()`, leaving `storeBase64()` byte-identical for the public-disk callers. Gates green (pint + phpstan + **635 tests**, 17 new; both Next apps type-check + build; check:rbac). ⚠️ **Nothing verified in a browser**; migration only exercised on sqlite (no local MySQL); `FormShapeFieldsSeeder` must be re-run; photo white-background detection **not built** |
| 035 | [psf-2027-deals](035-psf-2027-deals/TASK.md) | **todo** — spec only, nothing built. The sheet's `DEALS REGISTRATION PAGE 2027` tab: two signing parties (+3 attendees each), monetary value in SAR, a review lifecycle (accepted/rejected/returned), reference number/barcode, post-review emails and its own dashboard. Split out of 034 because it produces no `guests` row and so cannot be a form shape. Carries the sheet's requirements verbatim plus 8 open decisions to lock at kickoff |

| 036 | [psf-2027-branding](036-psf-2027-branding/TASK.md) | **in-progress** — the public site put on the real PIF brand: palette from the Brand Manual (`primary` pink→**PIF Green #004D43**, `secondary` teal→**PIF Gold #C4984F**, `accent` orange→**#02CD9A**, named `brand.*` values) and the PSF26 lockup extracted from the client PDF as transparent vector + PNG, replacing a logo left over from another event. The Fund webfont was **already** correctly installed, so nothing to do there. Primary inverts bright→very dark, so every `text-primary` was checked for a dark ground: all 67 are on white cards and contrast **improves** 3.86:1 → **9.80:1 AAA**. Frontend only — admin stays on its internal theme by owner decision. Gate green. ⚠️ not opened in a browser; `logo-white.png`/`eg-logo.png` now unreferenced but left in place |

_Add a row per task as you open it; newest at the bottom._

## Parked buckets

Not tasks — batches of follow-ups an owner explicitly chose to defer. Read before opening new work in
the same area, so a known-parked item isn't rediscovered as a "bug".

| File | Covers | Status |
|---|---|---|
| [PHASE3_PARKED_TODO.md](PHASE3_PARKED_TODO.md) | admin/frontend code-quality audit leftovers | closed 2026-07-19 |
| [PHASE22_PARKED_TODO.md](PHASE22_PARKED_TODO.md) | follow-ups surfaced by the P22 client-name sweep | **open** — parked 2026-07-22 |
