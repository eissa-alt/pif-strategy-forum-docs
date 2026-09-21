# Task 041 — Deals asked per guest, not a category

- **Status:** `done (code)` — merged to `main` and pushed; verified on MySQL and walked in a real browser
- **Opened:** 2026-09-21
- **Owner:** Imtinan Almutairi
- **Sub-app(s):** backend + admin + frontend
- **Branch(es):** `dev` → `main` (fast-forward)

## Goal

The client's decision ([D48](../../decisions/LEDGER.md)): **there is no deals category.**
Deals are asked of the private category's guests. The team sends a guest a single-use link
from a **Deal action in the guest listing**, the way complete-data is sent, and the way
speaker onboarding is sent on Tourise (`notify-onboarding` / `complete-onboarding`, which
this mirrors).

This reshapes [Task 035](../035-psf-2027-deals/TASK.md). Its deals tree was built on
2026-09-06/07 as a category flow (`psf-deals-2027`, `POST /api/deals`); the tree, the
admin listing and the review are kept.

## How it works

1. **Admin:** the guest listing's 🤝 action, `PATCH admin/guests/notify-deal/{id}`
   (`DealsController@notify`). It mails the category's `deal_email` template and logs
   "Deal Requested".
2. **Email:** `{{ deal_url }}` is minted as the template renders (`App\Services\DealLink`,
   a twin of `ReconfirmationLink`). One row per email in `deal_tokens`; re-minting on
   every send revives a used link.
3. **Guest:** `/{lang}/complete-deal/{category}/{token}` verifies server-side
   (`GET verify-deal-token`). It answers `submitted`, not 404, for a used link. Then the
   same two step components the category flow used, so the client-reviewed questions are
   unchanged.
4. **Submit:** `POST complete-deal` writes the deal tree with `deals.guest_id` and burns
   the token **inside the write's transaction, by a conditional update**. A validation
   failure leaves the link usable, and two tabs cannot both get through.

## Decisions

- **Gated three ways.** The category's `with_request_deal` switch decides where the icon
  appears (the seeder turns it on for the private category), `deal_email` is the template,
  and the admin needs the `guests_listing` `notify` action. The endpoint re-checks the
  switch: a hidden button is not a guard.
- **The icon sits beside Edit, on any status but rejected or cancelled.** It first sat
  with the accepted/invited actions. Moved at client request so it is always in the same
  place: the team decides who signs a deal, not the registration status.
- **The old flow is gone, not parallel.** `POST /api/deals`, the `psf-deals-2027` form
  shape (both front ends) and its step wiring are removed.
  `2026_09_21_000004_retire_deals_categories` **switches off** any category still on that
  shape. It finds it by shape, because it existed under more than one slug. It does **not
  delete** it: invitations and an invitation collection hold it by `RESTRICT` foreign keys,
  and the admin has no category delete. The owner agreed ("make it not active instead of
  deleting").
- **An agreed point of contact's email is the guest's own.** The page prefills the contact
  block from the guest and shows the email **locked** (read-only, not disabled, so it is
  still submitted). The API stores `guests.email` for `contact_agreed: yes` whatever is
  posted. A locked email skips the company-domain rule: the private category allows a
  personal address, and a rule the guest cannot satisfy would leave them stuck.
- **One form across both steps, held by the page.** Nothing is written to the `user_data`
  cookie, which every join form reads.
- **Steps are lazy-loaded** as the join forms load theirs: first load 402 → 207 kB.
- **Mobile unaffected.** `docs/mobile/` has no deal routes.

## Commits

- backend `1ec18c6`, `08ff1c4`
- admin `30c143e`
- frontend `60458a5`, `269d351`

## Gates

- Backend: **710 tests**, phpstan.
- Both Next apps: `type-check` + `lint` + `build`.
- Admin: `check:rbac`.

## Verification

- **MySQL, end to end, rolled back.** Mint, open, submit two agreements, read the tree
  back, resubmit (410), reopen (`submitted`), old endpoint (404). SQLite alone could not
  prove the `guest_id` foreign key.
- **Headless Chrome, EN + AR** (in [Task 042](../042-deal-signing-slots/TASK.md)). This
  found that the locked email *looked* editable: `.custom-input` paints its white
  background in unlayered CSS, which beats a plain Tailwind utility. Fixed in frontend
  `3ea1813` with `bg-[#f3f4f6]!`.

## Deploy

1. `php artisan migrate` (four migrations): `deal_tokens`, the category switch and
   template, `deals.guest_id`, and the retirement of the old deals category.
2. In the admin, open the private category, switch on **Deals Signing Ceremony** and
   choose a template that contains the **Deal Link** variable. **The migration does not
   switch it on.**
