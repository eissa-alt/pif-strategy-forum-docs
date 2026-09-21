# Task 035 — PSF 2027 Deals Signing Ceremony module

- **Status:** `done (code)` — built 2026-09-06/07 as a category flow, then reshaped: per-guest link ([041](../041-deals-per-guest-link/TASK.md)) and signing slots ([042](../042-deal-signing-slots/TASK.md))
- **Opened:** 2026-09-01
- **Owner:** Imtinan Almutairi
- **Sub-app(s):** backend | admin | frontend | docs
- **Branch(es):** —

> **Update 2026-09-21: this is built, and the flow has moved on.** Everything below is the
> original spec, kept as written.
>
> - **Built 2026-09-06/07 as a category.** Form shape `psf-deals-2027`, `POST /api/deals`,
>   the four-table deals tree, and the admin listing with review. Backend `fb8d897` …
>   `d1ea8a9`, admin `4746f18` / `77e1049`, frontend `0b3ee72` …
> - **Reshaped 2026-09-21 into a per-guest link** ([Task 041](../041-deals-per-guest-link/TASK.md),
>   ledger D48). There is no deals category any more. The private category's guests are
>   sent a single-use link from a Deal action in the guest listing. The category route and
>   shape are removed, and the old category is switched off by migration.
> - **Signing slots added the same day** ([Task 042](../042-deal-signing-slots/TASK.md),
>   ledger D49). Each agreement books a slot the team opened, instead of typing a time.
>   The admin gains a schedule, time slots, venues and email settings, and decides from
>   the row with a confirm step.
>
> The original banner said "Nothing here has been built"; it was left stale for two weeks
> after the build.

## Goal

Implement the `DEALS REGISTRATION PAGE 2027` tab of
`PSF 2027 Registration Sheet Requirement_13082026 - CAD comments.xlsx`: a private flow where a
PIF/PC contact registers one or more deal-signing ceremonies, which are then reviewed by staff
and scheduled.

## Why this is not a guest form shape

The registration shapes fit `FORM_SHAPES_CONFIG` because they produce **one `guests` row**.
A deal does not:

- it has **two signing parties**, each with name, company & position, email, phone and a **logo**;
- each party brings **up to 3 attendees** (name, company & position, email);
- it carries a **monetary value in SAR**, a title, a description and a proposed signing time;
- one contact may announce **several deals** ("How many deals would you like to announce?");
- it has a **review lifecycle** (accepted / rejected / returned) with notification emails and a
  final email carrying the exact time slot and location;
- it needs **its own dashboard**, not the guests listing.

Forcing this into a guest shape would satisfy the form rows and none of B24–B32.

## What the sheet specifies

### 1st Registration Page
| Row | Field | Notes |
|---|---|---|
| 1 | Planning to schedule a deal signing ceremony? | yes / no — a `no` presumably ends the flow |
| 2 | Agree to be main point of contact? | yes / no, plus name, company & position, email (**company domain only**), phone |

### 2nd Registration Page
| Row | Field | Notes |
|---|---|---|
| 1 | How many deals to announce? | number — drives how many deal blocks are rendered |
| 2 | Type of deal | MoU / Agreement / Joint Venture / Partnership |
| 3 | Agreement details | title, description, **monetary value in SAR**, proposed signing time (subject to availability) |
| 4 | Signing party 1 (PIF/PCs) | name, company & position, email (company domain), phone, **company logo** |
| 4.1 | Party 1 attendees | up to 3 × (name, company & position, email) |
| 5 | Signing party 2 (PIF/PCs) | same as 4 |
| 5.1 | Party 2 attendees | up to 3 × (name, company & position, email) |

> **Sheet note carried verbatim** for row 3: *"Disclaimer: the data will be for internal use, no
> announcement to be made using the provided number"* — this disclaimer must appear next to the
> monetary-value field.

### 3rd Registration Page
Message with a **reference number / barcode**, stating the submission is under review.

### Post-review
- Outcomes: **accepted / rejected / returned**.
- If accepted: a confirmation email reflecting the outcome and next steps.
- A final email with the **exact signing time slot and location**.

### Dashboard
- Deal status counts: accepted · rejected · pending · returned.
- Deal overview: party names, deal title, monetary value, number of ceremonies.

## Suggested shape (not locked — decide at kickoff)

```
deals                     id, reference_number, contact_* (name/company/position/email/phone),
                          deals_count, status(pending|accepted|rejected|returned),
                          reviewed_by, reviewed_at, review_note,
                          scheduled_at, location, created_at
deal_items                deal_id, type(mou|agreement|joint_venture|partnership),
                          title, description, monetary_value_sar, proposed_signing_time
deal_parties              deal_item_id, ordinal(1|2), name, company_position, email, phone, logo
deal_party_attendees      deal_party_id, name, company_position, email      (max 3, enforced)
```

## Open decisions (resolve before writing code — one at a time, per `ai/AI_RULES.md`)

1. **Is a deal linked to a `guests` row?** The main contact and the attendees are people who will
   attend. Do they become guests (badges, check-in, seating) or stay deal-only records?
2. **Reference number / barcode format** — reuse `generateUniqueRegistrationNumber()`, or a
   separate `DEAL-XXXX` series? Is "barcode" a QR like the guest one, or a printed code?
3. **"Returned" semantics** — can the contact re-open and resubmit through the original link, and
   does that reset the status? (Compare the existing `complete-data` tokenized flow.)
4. **Monetary value** — currency is fixed to SAR, but what type? Suggest `decimal(18,2)`; confirm
   whether it must be masked in exports given the internal-use disclaimer.
5. **Who reviews?** A new RBAC feature id (`deals`) with `view` / `review` actions, or reuse an
   existing one? Note `yarn check:rbac` must stay green.
6. **Emails** — three templates (accepted, rejected/returned, final schedule). Do they go through
   the existing automation/`EmailsTemplates` machinery so staff can edit them?
7. **Logo upload** — `deal_parties.logo` is company branding, so probably the **public** disk,
   unlike the private registrant files. Confirm; it also needs the PDF/image decision that Task 034
   settled for `company_profile`.
8. **Does the deals flow reuse the private-link/invitation mechanism** (`categories.visibility =
   private` + `Invitation`), or does it get its own token?

## Definition of Done

- [ ] Open decisions above answered and recorded here as LOCKED
- [ ] Migrations + models + resources
- [ ] Public flow (3 pages) in `-frontend`
- [ ] Admin CRUD, review actions and dashboard in `-admin`
- [ ] Three email templates, EN + AR
- [ ] RBAC feature registered; `yarn check:rbac` green
- [ ] `composer qa` + both Next apps `yarn type-check` && `yarn build` green
- [ ] Mobile contract checked (`../../mobile/`) if `routes/api.php` changes
