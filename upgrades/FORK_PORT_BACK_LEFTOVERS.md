# Fork port-back — leftovers and open decisions

**Opened:** 2026-08-08 · **Task:** [033](../tasks/033-fork-port-back/TASK.md) ·
**Working list:** [`FORK_PORT_BACK_FINDINGS.md`](FORK_PORT_BACK_FINDINGS.md)

Things found **while doing** the port-back that were deliberately **not** acted on. None of these
are "forgotten" — each was hit, understood, and parked because acting would have meant guessing at a
decision that belongs to the owner, or widening scope past the item in hand.

Ordered by how much they can hurt.

---

## 1. Needs checking against the real deploy

### 1.1 Twelve more `env()` calls that go null under `config:cache`

**Updated 2026-08-12 — 12 files down to 5.** `P033.11` fixed the reCAPTCHA secret, `P033.24` swapped
the 11 calls that already had a config key, `P033.23` deleted 7 more along with dead code, and
`P033.26` fixed `DynamicSmtpService`. What remains, per `phpstan-baseline.neon` under
`larastan.noEnvCallsOutsideOfConfig`:

| File | Calls | Variable |
|---|---|---|
| `SpeakersController` · `SponsorsController` · `SpeakerLabelsController` · `SponsorLabelsController` | 2 each | `FRONTEND_BASE_URL`, `REVALIDATE_SECRET_TOKEN` |
| `AuthController` | 1 | `SECURITY_ALERT_EMAILS` |
| `SendAutomationEmailNotification` | 1 | `MAIL_HOST_BULK` |

**Why it matters:** Laravel stops loading `.env` once the config is cached, so every one of these
reads `null` in a normal production deploy. The four ISR-revalidation controllers are the notable
group — a null `FRONTEND_BASE_URL` means speaker/sponsor page revalidation silently stops firing in
production, with no error.

**Why still parked:** unlike the ones already fixed, **none of these has a config key yet**, so each
needs one added and a judgement about the right default. Mechanical, but not a swap. The linter
tracks them — the baseline entries *are* the backlog, and PHPStan fails the build if one is fixed
without removing its entry.

### 1.2 `NEXT_PUBLIC_BASE_URL` has no fallback — renders `undefined…` under `yarn build`

Found while fixing the favicon (`P033.45`). The var is defined only in `.env*`, and our quality-gate
command **`yarn build` is a plain `next build` with no `.env`** — so every raw concatenation of it
renders the literal string `undefined`:

| File | Line | Renders as |
|---|---|---|
| `utils/publicUrl.ts` | 4 | `undefined/<path>` — the helper itself, so every caller inherits it |
| `pages/_app.tsx` | 92 | `canonical` = `undefined<route>` |
| `pages/_app.tsx` | 109 | OG `url` = `undefined<route>` |
| `pages/_app.tsx` | 112 | OG `image` = `undefined/seo/seo-v3.jpg` |

`_app.tsx:75` is the only one guarded (`|| ''`). `P033.45` sidestepped it for the favicon by using a
same-origin relative path, which is the correct fix for an asset on our own origin — but `canonical`
and the OG tags **must** be absolute, so they need a real base-URL fallback, not a relative path.

**Why it matters:** wrong `canonical` and OG tags are an SEO and link-preview problem, and they fail
silently — the page renders fine and nothing errors. **Needs checking against the real deploy
first:** if production always builds with `.env.production` present (`yarn production`), the live
site is fine and this only affects local/CI builds. That is the question to settle before choosing
between a build-time assert and a runtime default.

### 1.3 The admin has no favicon at all

`admin/pages/_document.tsx:22` points at `/favicon.ico`, but `admin/public/` contains only
`_favicon.ico` — underscore-prefixed, the generator's placeholder. So the admin tab has shown no
icon since the baseline was created.

**No fork fixes this** — ewc's admin has the identical gap; gfeai and pep each dropped in their own
branded `favicon.ico` next to the placeholder, which is a per-project asset, not a port. The
baseline fix is to provide a working default (copy `_favicon.ico` → `favicon.ico`) so every clone
starts with an icon and replaces it with its own branding. One file, needs a call on whether the
baseline should ship a generic icon or stay deliberately unbranded.

---

## 2. Decisions, not ports

### 2.1 Eight dead filter keys on the guests listing

`P033.17` wired `reconfirmed_will_attend`. Eight others are pushed into the URL by
`search-guests-by-super.tsx` but **no control renders them**, so nobody can set them except by
hand-editing the URL:

| key | backend filter exists? |
|---|---|
| `require_accommodation`, `require_flights` | **yes** — only the UI is missing |
| `accommodation_requests`, `admin_check_in_date`, `admin_check_out_date`, `flight_comments`, `self_booking`, `transportation_comments` | **no** |

Plus the three `dinner_invite` params still forwarded by the admin after `P033.4` removed the
backend filter (the column never existed).

**The decision:** delete the dead plumbing, or build the missing controls and the six backend
filters. Both are defensible; the six with no backend filter are mostly free-text/comment/date
fields that may not be worth filtering on at all.

### 2.2 `ExportEBadgesFiltered` — park it properly or delete it

`P033.7` removed the `dd()` from it. The method is still **unrouted with zero callers in any repo**,
builds a per-guest PDF into a variable it never uses, and returns `''`. It reads as working code.

**The decision:** give it a PARKED docblock (the convention `ExportInvitations` already uses two
hundred lines above it) or delete it outright. Nothing depends on it either way.

### 2.3 `check_in_time` ↔ `checked_in_at` duplication

Two generations of attendance columns holding one fact, reconciled by a `Guest::booted()` hook. The
2023 set (`check_in` / `check_in_time` / `check_in_count`) and the July-2026 seating set
(`checked_in_at` + `checked_in_by`, plus check-out and food pairs).

The legacy columns cannot simply be dropped — admin filters, reports **and the out-of-repo iPad
scanner** read them. Needs a migration, an admin/reports sweep and scanner sign-off. Recorded in
findings §9.

### 2.4 reCAPTCHA still 500s on a connection failure

`P033.9` guards the *response shape*. A `ConnectionException` is thrown before any response exists,
so `??` cannot catch it and a Google outage still surfaces as a 500 on admin login.

**Owner decision, taken 2026-08-08: leave as is.** Access is already fail-closed in that case, so
this is an error-surface gap, not a security one. Recorded here so it is not re-raised as new.

### 2.5 complete-data discards most of the four-step form

`updateGuestMissingData` uses `$request->only([...])` with a **12-field allow-list**. A guest
completing the four-step shape through a tokenized link fills in visa, accommodation and flight
fields that are then silently dropped. Before `P033.14` they were lost to a duplicate record;
now they are lost to the allow-list.

**The decision:** widen the allow-list per form shape, or accept that complete-data is a
profile-only flow and stop rendering the logistics steps on it. gfeai chose the second route
(`6b15463` splits `/complete-data` from `/complete-logistics`) — that is the large
`completion_tokens` item in findings §5.

**Updated 2026-08-24 — it is symmetrical, which strengthens the second option.** The *read* side is
narrowed the same way: `GuestsCompleteMissingDataResources` returns only **13 fields** (id, gender,
title, first/last name, company, job_title, email, phone, personal_image, will_attend,
dietary_requirements). So on a four-step completion form, nationality, residency, document, visa and
birth fields render **empty** despite being stored — the form is not "prefilled" past step 1 — and
anything typed into them is then dropped by the 12-field allow-list on submit.

So the flow is already, in practice, a profile-only flow: it neither loads nor saves the logistics
half. Making that explicit (gfeai's split) matches what the code already does; widening both ends
would be the larger change.

---

### 2.6 Other ALT clients' buckets on live render paths — PARKED 2026-08-12

**21 hardcoded URLs across 15 files**, all pointing at object storage belonging to other ALT
clients, on paths that render client-facing documents:

| Asset | Bucket | Used for | Refs |
|---|---|---|---|
| `spacer-500-px.png` | `sec-emails` | Outlook width shim in email layouts | 8 |
| `man_dark.png` / `woman_dark.png` | `hci-2026` | social-card placeholder avatars | 4 |
| `badges2/sticker.jpg` | `ims` | badge printing | 3 |
| `static-badge.png`, `shape_3.jpg` | `devego` | static badges, wrong-QR sheet | 2 |
| `ticket-edited-v3.jpg` | `tourise` | **guest ticket background** | 1 |
| `gosi_logos.png` | `glmc` | e-visa header | 1 |
| `e_visa_footer.png` | `temp-001` | e-visa footer | 1 |
| `invitation.jpg` | `faf` | invitation PDF (parked template) | 1 |

Three separate problems: the artwork **belongs to another customer** (a fresh clone prints tickets
carrying Tourise's design), the hosts are **outside our control** (if IMS deletes that object our
badges break), and every render makes an **outbound request to a third party**. Note `temp-001` is
not even a client bucket — it is a scratch bucket, serving the footer of a document sent to
embassies.

**A complete implementation was written and then discarded on owner instruction** (2026-08-12), not
because it was wrong — gates were green — but to defer the visual change. It is recoverable:

- ~~patch: `scratchpad/P033.26-render-assets.patch`~~
- ~~new config file: `scratchpad/P033.26-config-assets.php`~~

🔴 **GONE — confirmed 2026-08-24.** The scratchpad was session-scoped and has been cleaned up; the
directory no longer exists and neither file is anywhere under `/private/tmp/claude-501`. **The work
is lost and must be redone from this note.** The note below is now the only surviving record of what
it did, which is why it is written in that much detail — treat it as the spec, not as a summary of a
recoverable patch.

What it did: a `config/assets.php` with 10 env-backed keys, all defaulting to **null**, with
templates omitting the element when unset (except the 8 email shims, which are `height="0"` and
invisible either way). Named `assets`, not gfeai's `pdf`, because 8 of the 21 are email assets.
10 keys documented in `.env.example` + `.env.example_prod`. All 14 touched templates were
compile-checked and the social card was rendered both set and unset — PDF/email rendering has **no
automated coverage**, so that was the only real verification available.

**The decision this needs:** with no env set, badges print without the sticker, tickets without a
background, social cards without a placeholder avatar. Blank-but-honest versus keeping another
client's artwork until each project supplies its own. The alternative is copying the 8 images into
this repo, which relocates the confidentiality problem rather than solving it.

### 2.7 Invitation batches: no admin-side pre-check for registered emails

`P033.32` added the **server** gate — `POST /admin/invitations` now 422s a batch containing an email
that already has a registration, before anything is minted. That is the half that matters for
correctness, and it holds for every caller.

**Not done: the admin-side pre-check.** Today an admin pastes 200 rows, submits, and learns from a
422 that row 137 is already registered — with nothing created, so the whole batch has to be
resubmitted. gfeai does this check in the form as well (`ec316cc`), in **both** fill modes:

- **Excel** — validate on parse, before the rows reach the payload
- **Manual** — validate per row as it is entered

The endpoint `POST /admin/invitations/check-emails-list` already exists and is what such a check
would call.

**Why parked:** pure UX, two distinct fill paths, and it needs a decision about how to present a
partial failure — block the whole batch, or drop the offending rows and continue? The server
currently blocks the whole batch, so a form that silently dropped rows would disagree with it.

### 2.8 `visa_status_types` / `residencyStatusTypes` is dead plumbing

The same shape as the `optionalFields` problem removed in `P033.37`, found while doing `P033.38`.

`pages/[lang]/join/[category]/index.tsx` reads `data.visa_status_types` and passes it on as
`residencyStatusTypes`. **The API never returns that key** — a grep across the whole backend finds
nothing that produces it — so the value is always `null`, on the join page as well as the
completion page.

**No visible defect:** `fours-steps/step-2` falls back to the full `ResidencyStatusTypesSelect`
when the prop is empty, so the residency dropdown shows every option and always has.

**The decision:** delete the dead prop chain (as was done for `optionalFields`), or implement it —
return the category's allowed residency statuses from the API so a category can restrict the list.
The second is a small feature; the first is cleanup. Doing neither leaves a prop that looks
configurable and is not.

### 2.9 Export column letters: the headings-lookup mechanism was declined

The findings doc asked to **"port the mechanism, not corrected letters"** — a headings-based lookup
(gfeai `5c39fd8` / `7ce4225`) replacing hardcoded column letters across `app/Exports/`.

**Decided against, 2026-08-23.** The corrected letters shipped instead (`P033.33`, `P033.34`) with
`ExportColumnAlignmentTest` pinning them. Reasoning: **28 classes use hardcoded letters; 4 were
wrong.** A lookup rewrites all 28 to fix 4, and each rewrite is a chance to introduce the same
class of fault in a class that was previously correct — with no test coverage on the other 24 to
catch it. The alignment test closes the regression risk far more cheaply.

**Revisit when** an export gains or loses a column and the letters drift again, or when the other
24 classes get test coverage. At that point the lookup becomes cheap and the argument flips.

**If picking this up:** the four that were wrong were `GuestsExport`, `GuestsExportGuestView`,
`GuestsExportView` and `InvitationsCollectionsWithInvitationsExport`. `COL_TITLE_FROM='Z'` in the
last one ran the title converter over *genders* and blanked them — the most damaging of the four,
because it silently destroyed data in the output rather than mislabelling it.

### 2.10 `has()` filters on the guests listing narrow on an *empty* value

Found while writing `GuestListingFiltersTest` (`P033.47`). About 16 filters in
`GuestsController::applyFilters()` are guarded with `$request->has('x')` and then run
`where('x', $request->x)`. **`has()` is true for a present-but-empty parameter**, so `?gender=`
becomes `where('gender', '')` and the listing returns **zero guests**.

Affected (exact-match, so an empty value empties the listing): `status`, `guest_status_id`,
`gender`, `source`, `title_id`, `country`, `nationality`, `document_type`, `language`, `with_seat`.
The `LIKE` ones (`first_name`, `last_name`, `email`, `phone`, `job_title`, `company`,
`registration_number`, `document_number`) degrade more gently — `LIKE '%%'` still drops rows where
the column is NULL.

The `filled()`-guarded filters are correct — `will_attend`, `reconfirmed_will_attend`,
`require_flights`, `require_accommodation`, `is_saudi`, `verified`, `checked`, `has_printed`,
`badge_is_collected`, `check_in`, `e_badge_sent`, `created_from`, `created_to`. That asymmetry is
why `P033.47`'s empty-parameter test covers only the `filled()` five: extending it to the `has()`
ones would assert the bug.

**Reachable from the admin.** `search-guests-by-super.tsx` pushes its whole filter form into
`router.push({query})` on every search. Next drops `undefined` but keeps `''`. Worse, the selects do
`setValue('gender', String(value?.value))` — clearing one writes the **literal string
`"undefined"`**, which reaches the API as `?gender=undefined`.

**The decision:** swap `has()` for `filled()` across the block (one-line each, but ~16 call sites
and it changes live behaviour), and separately fix `String(value?.value)` in the admin. Worth doing
together so the fix can actually be verified end to end.

### 2.11 `where_null` discards every other filter on the guests listing

Also found writing `P033.47`. `GuestsController.php:237` and `:285` are a **byte-identical
duplicated block** — the same `where_null` handling, applied twice in one pass. Each does:

```php
$guests->whereNull($whereNullField)->orWhere($whereNullField, '');
```

The `orWhere` is **ungrouped**. SQL binds AND tighter than OR, so the whole query resolves to:

```
(...every other filter... AND f IS NULL) OR (f = '')
```

Any guest whose field is an empty string matches **regardless of every other active filter** —
category, status, RSVP, and the admin's own access restriction included. `where_not_null` at `:294`
is written correctly (`whereNotNull(...)->where(..., '!=', '')`, both AND) and is not affected.

**Severity is higher than a listing glitch:** `selectIds()` reuses `applyFilters()` to power "select
all N matching" in the automation guest picker, so a bulk send could target guests the operator
never selected.

**The decision:** wrap the pair in a closure — `$guests->where(fn ($q) => $q->whereNull($f)->orWhere($f, ''))`
— and delete the duplicate block. Needs a call on intent first: should `where_null` mean *null only*,
or *null-or-empty* as written? The grouping fix is mechanical once that is settled.

---

## 3. Cosmetic / hygiene

### 3.1 `anchor` on Headless UI panels (dropdown clipping)

`P033.12`/`P033.13` fixed the freeze with `modal={false}` and deliberately did **not** add
`anchor`. Panels are clipped inside overflow containers on some screens; adding `anchor` switches
them to portal positioning, which conflicts with the `absolute` classes all 17 sites use. Worth
doing **with a browser open**, not as a blind sweep. See the correction recorded in findings §5.

### 3.2 `console.*` in the frontend app

The admin is now clean (`P033.19` removed the last one). The **frontend** still has some, including
`console.error(error)` in `one-step/step-1.tsx`. Hard rule 8. A one-commit sweep, not done here
because it is unrelated to any findings item.

### 3.3 `print-logs.tsx` pagination shape

Carries the same hand-written-key `paginate()` shape the guests listing had, but takes exactly
`{mode, page, query, status}` — so it loses nothing today. Hardening it is one line and prevents the
identical bug the day someone adds a filter.

### 3.4 `GuestsResources` still emits a null `dinner_invite`

`P033.4` removed the backend filter for a column no migration creates. The resource still returns
the key, always `null`. Harmless, but it is a response-shape change so it was left alone.

### 3.5 `CLAUDE.md` is untracked

The wrapper root has no `.git`, so the rule-5 gate update made on 2026-08-08 (`yarn production` →
`yarn build`) lives only on one machine and will not reach a clone or a teammate. Worth deciding
where that file should be version-controlled.

### 3.6 The four public-disk upload endpoints have no request validation

`BadgesController::upload`, `SponsorsController::upload`, `SpeakersController::upload` and
`CategoriesController::upload` carry **no `Validator` and no `required` rule at all**. Two
consequences, both pre-existing and untouched by `P033.25`:

- a missing `file` field reaches a `string`-typed parameter and **500s** instead of returning 422;
- `$dir` is built from `$request->moduleName.'/'.$request->inputName`, **unsanitised**, so an
  authenticated admin can write into arbitrary folders under the public disk. Traversal itself was
  tested and *is* blocked by Flysystem (`PathTraversalDetected`), but it surfaces as an unhandled
  500 rather than a clean rejection.

`P033.25` closed the executable-extension path at the service layer, which covers every caller. This
is the caller-layer half: four files, and it needs a per-endpoint decision on accepted types rather
than a blanket rule. Note `'jpeg'` in `ALLOWED_EXTENSIONS` is also dead — normalization turns
`jpeg` into `jpg` before the `in_array`, so it can never match. Cosmetic.

## 4. Verification owed

Nothing in this session was exercised in a browser — all of it is reasoned from source, with gates
green. Worth a click-test before trusting:

- a dropdown in the admin, and **Country on a phone** in the public form (`P033.12` / `P033.13`)
- a real tokenized complete-data link end to end (`P033.14`)
- the Reconfirmation filter returning different rows (`P033.17`)
- an email log row showing a real Sent/Delivered value (`P033.16`)
- a guest upload from the admin guest form — those paths 404'd for a long time (`P033.6`)

Still unanswered from the original audit: **does the admin CSP block reCAPTCHA on login** (settle in
a browser, not from source), and **is the backend's reCAPTCHA secret Google's *test* secret**.
