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

- patch: `scratchpad/P033.26-render-assets.patch`
- new config file: `scratchpad/P033.26-config-assets.php`

⚠️ **Scratchpad is session-scoped and will be cleaned up.** If this is not restored soon, treat the
work as gone and redo it from this note.

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

---

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
