# Wave 1 progress — 128-pif-psf-2026

<!-- progress:start -->

**Overall:** `░░░░░░░░░░░░░░░░░░░░` 0% — **0 / 24** closed (0 done, 0 closed without code)

| Step | Closed | Progress |
|---|---|---|
| 1. Close the critical holes | 0 / 4 | `░░░░░░░░░░` 0% |
| 2. Captcha and abuse controls | 0 / 3 | `░░░░░░░░░░` 0% |
| 3. Stored XSS and uploads | 0 / 2 | `░░░░░░░░░░` 0% |
| 4. Admin BFF write guard | 0 / 1 | `░░░░░░░░░░` 0% |
| 5. Disclosure | 0 / 2 | `░░░░░░░░░░` 0% |
| 6. Error handling and exports | 0 / 3 | `░░░░░░░░░░` 0% |
| 7. Headers, cookies and proxy trust | 0 / 4 | `░░░░░░░░░░` 0% |
| 8. Data integrity and triage | 0 / 3 | `░░░░░░░░░░` 0% |
| 9. Verify | 0 / 2 | `░░░░░░░░░░` 0% |

_Refreshed 2026-09-21 by `python3 /Users/admin/Projects/ALT/operational-tasks/wave1_progress.py`_

<!-- progress:end -->

## How to use

- `[ ]` open · `[x]` done · `[-]` closed without a code change (n/a, accepted risk, deferred) — write the reason in the [TASK.md](TASK.md) Log.
- This file is updated **centrally**, from the operational-tasks session, after each item is verified in the code. Project sessions record their work in the [TASK.md](TASK.md) Log instead. Full detail for each item is in [ITEMS.md](ITEMS.md).
- Central refresh command: `python3 /Users/admin/Projects/ALT/operational-tasks/wave1_progress.py`
- Items marked **Day 0** are the quick wins to do first.

## Step 1 — Close the critical holes

- [ ] **B3 · C01** — Build the admin reset and invite link from config('app.admin_url') and drop back_link from the request contract, bundling C01-3.1 (the config key), C01-3.7/3.9 (three admin senders plus ?lang= on r…
- [ ] **B10 · H01** — Move the whole forgot-password branch onto an AdminPasswordResetRequested event handled by a ShouldQueue listener with unconditional dispatch and a uniform response, plus H01.2 (no 500/200 split on…
- [ ] **MX-WHERE-NULL · WHERE-NULL-SCOPE** — Group `$guests->whereNull($f)->orWhere($f, '')` in a closure so the admin-scope restriction applied afterwards binds to BOTH branches.
- [ ] **B15 · H02.1** — Delete BOTH forced-500 debug hooks — RWYI82IWDG in InvitationsController::verifyInvitation and the sibling IUFRP3GYZ4ISPUEU in GuestsController's verify-status-token path — and port tests/Feature/I…

## Step 2 — Captcha and abuse controls

- [ ] **B7 · C02** — Port the rewritten app/Rules/ReCaptcha.php WITH its prerequisites, per project: the services.recaptcha config block, messages.recaptcha_required in EN and AR, GOOGLE_RECAPTCHA_SECRET pinned in phpu…
- [ ] **B11 · C02-2.2-checkunique** — Write a fresh fix — no fix exists in pep either: replace the client-supplied column name in checkUnique with a per-project allow-list derived from that project's actual frontend callers.
- [ ] **B12 · C02-2.3-otp** — Add `->where('email', $request->email)` to the OTP lookup, normalising both sides with trim + lowercase.

## Step 3 — Stored XSS and uploads

- [ ] **B4 · PHASE-1** — Render category share text as a React child inside a <div className="whitespace-pre-wrap wrap-break-word">{shareText}</div> instead of dangerouslySetInnerHTML, on BOTH the share page and the succes…
- [ ] **B13 · M04-SVG** — Two different fixes under one id. In 129-pif-gamf and the three v2 targets, add the mimes rule to the conference upload (AdminConferenceController.php:276 in 129 takes pep's one-word change verbatim).

## Step 4 — Admin BFF write guard

- [ ] **B6 · M03-BFF** — Port isSelfOriginatedWrite into pages/api/proxy/[...path].ts — read Sec-Fetch-Site first, fall back to Origin then Referer, compare host-to-host against the request's own Host (never an env var), r…

## Step 5 — Disclosure

- [ ] **B5 · C06** — Narrow AdminsSelectResources::toArray() to id plus trim(first_name.' '.last_name), exactly as pep commit d9f6476 did — deliberately WITHOUT gating the route.
- [ ] **B17 · H04** — Strip app and env from GET /api/health, keeping timestamp (the mobile contract names this route as how a client learns server time), and add the regression test asserting their absence.

## Step 6 — Error handling and exports

- [ ] **B9 · PHASE-0** — Port the Handler::render rewrite with P0.1-P0.8 in the documented branch order (HttpResponseException, Throttle, Authentication, Validation, ModelNotFound, NotFound, generic 4xx, generic 500), keep…
- [ ] **L9 · P0.14** — Get a written confirmation per project that the mobile client implements the 401-to-OTP re-login handler its own contract documents, BEFORE the PHASE-0 backend deploy.
- [ ] **B8 · EXPORT-FORMULA** — One change, two ordered steps, one deploy: (1) add `: bool` to every bindValue override, (2) introduce app/Exports/Concerns/TextSafeValueBinder.php and swap the parent, (3) port ExportClassesAreLoa…

## Step 7 — Headers, cookies and proxy trust

- [ ] **B20 · M04-API** — Add the SecurityHeaders global middleware to the API origin (closed CSP, Permissions-Policy, nosniff, X-Frame-Options, Referrer-Policy), keeping pep's deliberate omission of the `sandbox` directive…
- [ ] **B18 · M01** — Set the session cookie's Secure flag explicitly, keeping the env('APP_ENV') !== 'local' exemption, and document the key as ABSENT (commented out) in the env examples — never as a bare SESSION_SECUR…
- [ ] **B19 · H03.1** — Repair the 429 so it keeps Retry-After and reports the limiter that actually blocked, rather than the outer throttle decoration's ceiling.
- [ ] **B16 · H03.4** — Add config/trustedproxy.php and declare $headers in TrustProxies.php, leaving $proxies unset, plus utils/forwarded-for.ts and a `headers:` option on each Axios.get in getServerSideProps.

## Step 8 — Data integrity and triage

- [ ] **B21 · PHASE-6.6-refcheck** — Handle the RESTRICT foreign keys and the dangling guest_status_ids JSON before a guest status is deleted: return a 409 with a message naming what still references it, instead of letting the driver…
- [ ] **B22 · PHASE-4-TRIAGE** — Adopt the method, not just the fixes: verify each precondition in the target project before porting, tier the evidence (code-verified / harness-observed / needs-production, BRIEF-5), combine the th…
- [ ] **L3 · PHASE-3.4** — Add App\Models\PersonalAccessToken with the strict id|hash lookup and Sanctum::usePersonalAccessTokenModel, after tracing every token consumer — in 129-pif-gamf that means MobileQrController.php:23…

## Step 9 — Verify

- [ ] **L4 · PHASE-6.5-mobile-demo** — Confirm MOBILE_DEMO_EMAIL and MOBILE_DEMO_OTP are blank in every 129-pif-gamf environment and add an `if (! app()->environment('production'))` guard around both bypass blocks at MobileAuthControlle…
- [ ] **L14 · PHASE-6.7-smoke** — Write post-deploy-security-check.sh and wire it into the pipeline: read-only probes asserting the Secure cookie flag, the header set, /health's shape, APP_DEBUG behaviour and the _ignition 404.
