# Task 040 — Security port, wave 1 (from 123-pif-pep-v2)

- **Status:** `todo`
- **Opened:** 2026-09-21
- **Owner:** —
- **Sub-app(s):** backend + admin + frontend
- **Branch(es):** `security/wave-1` in each code repo, created from `dev` — see [Branch rules](#branch-rules)
- **Project phase:** ongoing — live (v2 template)

> Created uncommitted on 2026-09-21 by the cross-project security review. Leave this folder uncommitted while you work — the user commits it.

## Start here — new session prompt

Open this repo in VS Code, start a fresh session, and send:

```text
Read docs/tasks/040-security-port-wave-1/TASK.md and docs/tasks/040-security-port-wave-1/ITEMS.md.
This is wave 1 of a cross-project security port. The fixes come from
123-pif-pep-v2 (reference code at /Users/admin/Projects/ALT/123-pif-pep-v2/pif-pep-v2-repos).
Start in Plan Mode. Before changing anything, verify each item against the
CURRENT code — line numbers in ITEMS.md were verified on 2026-09-20/21 and
may have drifted. Work the Plan steps in order, one step at a time.
Follow the Branch rules in TASK.md: work on security/wave-1 and DO NOT COMMIT —
stop after each step so the user can review the changes. Never merge or
push. Log each step in TASK.md. Stop and ask before touching any .env*
file.
```

## Branch rules

The work stays **off `dev`** until the user decides it is safe to merge.

- Every code change goes on **`security/wave-1`**, in each code repo you touch: `pif-psf-2026-backend`, `pif-psf-2026-admin`, `pif-psf-2026-frontend`. The task files themselves live in the docs repo, which stays on `main`.
- **Do not commit.** Finish one step, leave its changes uncommitted, and stop, so the user can review the diff. The user then commits it, or tells you to commit that step on `security/wave-1` using the commit format in CLAUDE.md. Do not start the next step until the previous one is committed.
- If the branch does not exist yet in a repo: `git fetch`, then `git switch -c security/wave-1 origin/dev`.
- **Never** commit to, merge into, or push `dev` or `main`. **Do not push `security/wave-1` either.** The user merges it into `dev` and pushes when it is safe.
- Before starting each step — once the previous step is committed — bring the branch up to date: on `security/wave-1`, run `git fetch` then `git merge origin/dev`.
- Once a step is committed, record its commit SHA in each repo in the Log. That lets the user merge up to any single step on its own — for example the Day 0 items ahead of everything else.
- Leave the docs changes (this `TASK.md` and `PROGRESS.md`) uncommitted — the user commits them.
- You do not need to tick `PROGRESS.md` or run any progress script: progress is updated centrally after the work is verified in the code.

## Goal

Close the security gaps that 123-pif-pep-v2's pentest remediation fixed and this project still carries — above all the unauthenticated admin-takeover path through the password-reset link (C01) and the guest-listing scope leak. Where 122-gfeai-v2 or 127-tourise has already landed an item, replay its diff: the affected code is identical across the three v2 projects.

## Scope

- **In:** the 24 items in the Plan below. Full detail for each is in [ITEMS.md](ITEMS.md).
- **Out:** `C02-3.4` is already removed here. Non-security porting-matrix items belong to wave 2.

## Plan

This describes the plan. Progress is tracked centrally in [PROGRESS.md](PROGRESS.md) once the work is verified.

### Step 1 — Close the critical holes

C01 goes backend-first and needs `ADMIN_PANEL_URL` set on every box in the same change window. B15 is test-only here.

- **B3 · C01** _(day)_ — Build the admin reset and invite link from config('app.admin_url') and drop back_link from the request contract, bundling C01-3.1 (the config key), C01-3.7/3.9 (three admin senders plus ?lang= on r…
- **B10 · H01** _(day)_ — Move the whole forgot-password branch onto an AdminPasswordResetRequested event handled by a ShouldQueue listener with unconditional dispatch and a uniform response, plus H01.2 (no 500/200 split on…
- **MX-WHERE-NULL · WHERE-NULL-SCOPE** _(hours)_ — Group `$guests->whereNull($f)->orWhere($f, '')` in a closure so the admin-scope restriction applied afterwards binds to BOTH branches.
- **B15 · H02.1** _(minutes)_ — Delete BOTH forced-500 debug hooks — RWYI82IWDG in InvitationsController::verifyInvitation and the sibling IUFRP3GYZ4ISPUEU in GuestsController's verify-status-token path — and port tests/Feature/I…

### Step 2 — Captcha and abuse controls

Enable enforcement in the dev environment first, before staging and production.

- **B7 · C02** _(day)_ — Port the rewritten app/Rules/ReCaptcha.php WITH its prerequisites, per project: the services.recaptcha config block, messages.recaptcha_required in EN and AR, GOOGLE_RECAPTCHA_SECRET pinned in phpu…
- **B11 · C02-2.2-checkunique** _(hours)_ — Write a fresh fix — no fix exists in pep either: replace the client-supplied column name in checkUnique with a per-project allow-list derived from that project's actual frontend callers.
- **B12 · C02-2.3-otp** _(hours)_ — Add `->where('email', $request->email)` to the OTP lookup, normalising both sides with trim + lowercase.

### Step 3 — Stored XSS and uploads

- **B4 · PHASE-1** _(minutes)_ — Render category share text as a React child inside a <div className="whitespace-pre-wrap wrap-break-word">{shareText}</div> instead of dangerouslySetInnerHTML, on BOTH the share page and the succes…
- **B13 · M04-SVG** _(hours)_ — Two different fixes under one id. In 129-pif-gamf and the three v2 targets, add the mimes rule to the conference upload (AdminConferenceController.php:276 in 129 takes pep's one-word change verbatim).

### Step 4 — Admin BFF write guard

Land the null-body correction (`047d31f`) in the same release.

- **B6 · M03-BFF** _(hours)_ — Port isSelfOriginatedWrite into pages/api/proxy/[...path].ts — read Sec-Fetch-Site first, fall back to Origin then Referer, compare host-to-host against the request's own Host (never an env var), r…

### Step 5 — Disclosure

- **B5 · C06** _(minutes)_ — Narrow AdminsSelectResources::toArray() to id plus trim(first_name.' '.last_name), exactly as pep commit d9f6476 did — deliberately WITHOUT gating the route.
- **B17 · H04** _(minutes)_ — Strip app and env from GET /api/health, keeping timestamp (the mobile contract names this route as how a client learns server time), and add the regression test asserting their absence.

### Step 6 — Error handling and exports

Re-baseline 5xx alerting BEFORE the PHASE-0 deploy; `WhatsAppWebhookController.php:57` aborts 503 on an unauthenticated webhook, so P0.4 and P0.5 must land together. The export binder must ship both halves in ONE deploy.

- **B9 · PHASE-0** _(day)_ — Port the Handler::render rewrite with P0.1-P0.8 in the documented branch order (HttpResponseException, Throttle, Authentication, Validation, ModelNotFound, NotFound, generic 4xx, generic 500), keep…
- **L9 · P0.14** — Get a written confirmation per project that the mobile client implements the 401-to-OTP re-login handler its own contract documents, BEFORE the PHASE-0 backend deploy.
- **B8 · EXPORT-FORMULA** _(hours)_ — One change, two ordered steps, one deploy: (1) add `: bool` to every bindValue override, (2) introduce app/Exports/Concerns/TextSafeValueBinder.php and swap the parent, (3) port ExportClassesAreLoa…

### Step 7 — Headers, cookies and proxy trust

The trusted-proxy code is inert, so it can go on the branch freely; setting its value is a separate day, in a maintenance window.

- **B20 · M04-API** _(hours)_ — Add the SecurityHeaders global middleware to the API origin (closed CSP, Permissions-Policy, nosniff, X-Frame-Options, Referrer-Policy), keeping pep's deliberate omission of the `sandbox` directive…
- **B18 · M01** _(minutes)_ — Set the session cookie's Secure flag explicitly, keeping the env('APP_ENV') !== 'local' exemption, and document the key as ABSENT (commented out) in the env examples — never as a bare SESSION_SECUR…
- **B19 · H03.1** _(minutes)_ — Repair the 429 so it keeps Retry-After and reports the limiter that actually blocked, rather than the outer throttle decoration's ceiling.
- **B16 · H03.4** _(day)_ — Add config/trustedproxy.php and declare $headers in TrustProxies.php, leaving $proxies unset, plus utils/forwarded-for.ts and a `headers:` option on each Axios.get in getServerSideProps.

### Step 8 — Data integrity and triage

- **B21 · PHASE-6.6-refcheck** _(hours)_ — Handle the RESTRICT foreign keys and the dangling guest_status_ids JSON before a guest status is deleted: return a 409 with a message naming what still references it, instead of letting the driver…
- **B22 · PHASE-4-TRIAGE** _(hours)_ — Adopt the method, not just the fixes: verify each precondition in the target project before porting, tier the evidence (code-verified / harness-observed / needs-production, BRIEF-5), combine the th…
- **L3 · PHASE-3.4** — Add App\Models\PersonalAccessToken with the strict id|hash lookup and Sanctum::usePersonalAccessTokenModel, after tracing every token consumer — in 129-pif-gamf that means MobileQrController.php:23…

### Step 9 — Verify

- **L4 · PHASE-6.5-mobile-demo** — Confirm MOBILE_DEMO_EMAIL and MOBILE_DEMO_OTP are blank in every 129-pif-gamf environment and add an `if (! app()->environment('production'))` guard around both bypass blocks at MobileAuthControlle…
- **L14 · PHASE-6.7-smoke** — Write post-deploy-security-check.sh and wire it into the pipeline: read-only probes asserting the Secure cookie flag, the header set, /health's shape, APP_DEBUG behaviour and the _ignition 404.

## Decisions

- Finished projects (101-leap-dinner-v2, 113-pif-directors-gathering, 126-pif-hakathon) are sources only; nothing in this task changes them.
- Durable decisions → promote to `decisions/LEDGER.md`.

## Log

- 2026-09-21 — opened by the cross-project security review; 24 items selected for this project.

## Definition of Done

- [ ] [PROGRESS.md](PROGRESS.md) at 100% — every item `[x]` done, or `[-]` closed with its reason in the Log
- [ ] All code committed on `security/wave-1` in the relevant repos — **not merged and not pushed**; the user merges into `dev` and pushes when it is safe
- [ ] Every step reviewed by the user before it was committed
- [ ] Each step's commit SHA per repo recorded in the Log
- [ ] EN + AR translations in the same commit (if any user-facing strings)
- [ ] Quality gate green (backend `pint --test` + `php artisan test`; Next apps `yarn type-check` + `yarn production`)
- [ ] Mobile contract checked where `routes/api.php` or status codes changed
- [ ] DevOps items for this project handed over with a named owner
- [ ] Post-deploy security smoke run (L14) where it applies
- [ ] This TASK.md set to `done`; index row added

## References

- Cross-project security review: `/Users/admin/Projects/ALT/operational-tasks/SECURITY_PORT_REVIEW_2026-09-21.md`
- Cross-project porting matrix: `/Users/admin/Projects/ALT/operational-tasks/PORTING_MATRIX_2026-09-20.md`
- pep-v2 source documents: `/Users/admin/Projects/ALT/123-pif-pep-v2/pif-pep-v2-repos/docs/security/`
