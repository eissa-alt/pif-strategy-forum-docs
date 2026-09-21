# Task 042 — Deal signing slots, modelled on GFAI's bilateral booking

- **Status:** `done (code)` — merged to `main` and pushed; guest page walked in a real browser; admin screens not yet opened in one
- **Opened:** 2026-09-21
- **Owner:** Imtinan Almutairi
- **Sub-app(s):** backend + admin + frontend
- **Branch(es):** `dev` → `main` (fast-forward)

## Goal

"Improve the system like booking rooms in GFAI, but it will be deal, not booking room."
Each agreement used to carry a free-text *proposed signing time*, and the team typed one
`scheduled_at` per submission after accepting it. Nothing stopped two agreements getting
the same hour, and three agreements shared one time.

Now the guest **picks a slot per agreement** from the ones the team opened. Only free ones
are offered. Ported from GFAI (`Gfai-v2/gfeai-v2-*`, bilateral booking: rooms, a time-slot
catalogue, applied slots, holding requests, a schedule grid, template settings).

## Owner decisions, taken at kickoff

| Question | Answer |
|---|---|
| Who picks a slot? | **Each agreement** (a submission with 2 agreements books 2 slots) |
| Is a picked slot held while the team reviews? | **Yes, from the moment it is submitted**, as on GFAI |
| Where do signings happen? | **One signing stage**, modelled as venues so a second is an admin row, not code |

The durable rules are in [D49](../../decisions/LEDGER.md).

## Backend

Migration `2026_09_22_000001_create_deal_signing_slots`. The prefix is a day ahead of the
work; see [Commits](#commits).

- `deal_venues`: one **Signing Stage** created by the migration.
- `deal_time_slots`: the catalogue.
- `deal_slots`: venue × day × window. Slots **copy** the catalogue's times, so a catalogue
  edit cannot move a signing both parties were already told.
- `deal_items.slot_id`: **null on delete, never cascade**, so removing a slot cannot delete
  an agreement.
- `deal_settings`: one row of templates, plus the SMTP account.
- `deal_emails`: the send log.

Booking and review:

- `DealSlotAllocator`: available days and times, and allocation (first free venue by
  order).
- The submit books each agreement **under a row lock with a locking re-check**. A plain
  read is served from the REPEATABLE READ snapshot and misses another guest's insert; GFAI
  found this. A slot lost in the meantime throws `DealSlotTakenException`. The whole
  submission rolls back, **token included**, and the response is
  `409 {data: slot_taken, item_index}`.
- **Review:** accept and reject email from Deals > Settings. Re-accepting a rejected or
  returned deal re-takes its slots and **is refused (409)** if another deal took one since.
- **Emails:** `DealMailer` → `DealEmail` → `SendDealEmailNotification`, marked sent by
  `AfterEmailSentListener` (`notification-deal-email`). A step with no template sends
  nothing. Template variables:
  - `{{ deal_reference }}`
  - `{{ deal_signings }}`: each agreement's title, date, time and venue
  - `{{ deal_contact_name }}`
  - `{{ deal_agreements_count }}`
  - `{{ deal_review_note }}`
- **Admin API** under `admin/deals/*`, on the existing `deals` feature: reading on `view`,
  every change on `schedule` (the action that already meant "sets the confirmed slot").
  No new RBAC. It covers venues, time slots, slots (apply / move / close / delete held
  → 409), dates, schedule grid and settings.
- **Guest availability** is behind the deal link's token: `GET deal-slots/days`, `GET
  deal-slots/times`.

## Admin

Deals has a sub-menu:

| Screen | Purpose |
|---|---|
| **Submissions** | The listing |
| **Signing schedule** | The day as a grid: venues × windows; free, pending and accepted |
| **Time slots** | The catalogue, plus applying it to days in one action. With one venue it is pre-selected |
| **Venues** | The signing stage, and any others |
| **Settings** | The three templates and the SMTP account |

Client requests the same day:

- **Where and when, visibly.** The listing's **Signing slot** column shows date, time and
  venue per agreement, replacing "scheduled at". Deals from before slots fall back to it.
- **Accept and reject outside "see more", with a confirm button.** ✓ and ✗ sit on the row;
  a booked grid cell opens the booking itself. Both open `DealDecisionModal`, which shows
  who, every agreement with where and when, what the decision will do, and an optional
  note. **Nothing is saved before "Confirm acceptance" / "Confirm rejection".** Only the
  decision that changes something is offered, and only with the `review` action.
- **The Delete row action is removed.** The endpoint remains behind its permission.
- "See more" keeps its review section: it is the only place to mark a deal **returned**.
  Its manual date and location box now appears only for deals without slots.

## Frontend

`components/join/forms/psf/deals/signing-slot-picker.tsx` replaces the free-text time in
every agreement:

- Day cards (weekday and date), then time buttons.
- A time already chosen for another agreement in the form is shown but disabled.
- A 409 clears that one time, refreshes what is free, and names the agreement to fix.
- Arabic dates are Gregorian with Latin digits (`ar-u-ca-gregory-nu-latn`). `ar-SA` alone
  renders Hijri.
- Times read **"10:00 to 10:15" / "10:00 إلى 10:15"**, not with a dash, at client request.

## Commits

- backend `b3eb2a5`, `4ba6e76`
- admin `34a00f8`, `08212f4`
- frontend `9701903`, `3ea1813`

**Date correction:** backend `81998ef`, admin `1600973` and frontend `67e588d` correct the
code comments, which had dated these decisions 2026-09-22. They were made on 2026-09-21.
The migration filename keeps its `2026_09_22` prefix: it had already run under that name
locally, and renaming would make `migrate` try to recreate the tables.

## Gates

- Backend: pint, phpstan, **741 tests** (+31: `DealSlotBookingTest`,
  `AdminDealScheduleTest`, and the submission tests now book real slots).
- Both Next apps: `type-check` + `lint` + `build`. Builds were run in isolated copies,
  because the owner's `next dev` servers share `.next`.
- Admin: `check:rbac` (38 links).

## Verification

- **MySQL, rolled back:** A books 10:00; B is refused with 409 and keeps a usable link;
  B books 10:20; A's used link can no longer read availability. The row locks run
  cleanly on MySQL, which SQLite ignores.
- **Headless Chrome, EN + AR,** against temporary servers started and stopped for the
  purpose. Step 1 → Next → one agreement → a day → the six times; the email is locked.
  **It found two bugs no test could** (frontend `3ea1813`):
  1. "Field is required" fired on the time the moment a day was picked. Clearing it went
     through onChange validation.
  2. The locked email looked editable. See [Task 041](../041-deals-per-guest-link/TASK.md).

## Owed

- ⚠️ **The admin screens have not been opened in a browser.** There was no admin session
  to drive.
- **Deploy:** `php artisan migrate`, then build both Next apps.
- **Before any deal link goes out in production:** open slots (Deals > Time slots →
  apply to the signing days). With none, the guest sees "no signing slots are open" and
  cannot submit.
- Choose the three templates in Deals > Settings. Optionally rename the stage and give it
  a location for the acceptance email.
- **Moving a booked slot's time in the admin does not email the guest.** Same as GFAI.
- Local only: a demo schedule (6 windows × 5, 6 and 7 Oct) exists in the owner's local DB.
  It is data, not code; remove it from the Signing schedule screen.
