# NileCare — Security Findings
 
**Application:** NileCare / Egyptian Specialist Hospital  
**Assessment period:** 25–27 October 2026  
**Scope:** Web application security review  
**Report status:** Working draft — technical evidence to be added
 
## Overview
 
This document summarizes the main security issues identified during testing. The detailed requests, responses, screenshots, and reproduction notes will be added to each finding before the report is finalized.
 
| ID | Finding | Status |
|---|---|---|
| 01 | Stored Cross-Site Scripting (XSS) — Attachment Upload | Confirmed |
| 02 | Insecure Direct Object Reference (IDOR) | Confirmed |
| 03 | Information Disclosure — Doctor Details and Email Addresses | Confirmed |
| 04 | Business Logic Flaw — Client-Controlled Discount | Confirmed |
| 05 | Forgot Password — Unauthenticated Password Reset / Account Takeover | Confirmed |
| 06 | Stored Cross-Site Scripting (XSS) — Medical Report Fields | Confirmed |
 
---
 
## 01 — Stored Cross-Site Scripting (XSS) — Attachment Upload
 
**Affected functionality:** Medical-report attachment upload  
**Endpoint:** `POST /reports/upload-attachment`  
**Status:** Confirmed
 
The application accepts a file whose name contains a permitted image extension even when its final extension is `.html`. A JPEG/HTML polyglot was accepted, stored in a publicly accessible uploads directory, and served as HTML. JavaScript execution in the browser was demonstrated in the original assessment.
 
**Impact:** Same-origin JavaScript execution when a victim visits the uploaded file. This could expose authenticated users to account-level actions, depending on the victim's session and available endpoints.
 
**Technical details — add here**
 
- **Affected URL:** [ADD]
- **Prerequisites:** [ADD]
- **Steps to reproduce:** [ADD]
- **HTTP request:** [ADD]
- **HTTP response:** [ADD]
- **Payload / PoC:** [ADD]
- **Screenshots / evidence:** [ADD]
**Recommendation:** Validate the final filename extension, verify actual content, re-encode uploaded images, store attachments outside the public web root, and prevent user-controlled uploads from being served as executable HTML on the application origin.
 
---
 
## 02 — Insecure Direct Object Reference (IDOR)
 
## Vulnerability Assessment Report
 
---
 
## 1. Executive Summary
 
An **Insecure Direct Object Reference (IDOR) / Broken Access Control** vulnerability exists within the NileCare Patient Portal booking system. The application relies on a client-side modifiable session cookie (using URL-safe timed serialization without signature enforcement verification) to identify the authenticated user. By decoding the session payload, modifying identity parameters (`user_id`, `patient_name`, `patient_email`), and re-encoding the token, an attacker can manipulate session state to schedule medical appointments on behalf of arbitrary victims.
 
---
 
## 2. Vulnerability Overview
 
* **Vulnerability Type:** Insecure Direct Object Reference (IDOR) / Broken Access Control / Insecure Session Management.
* **Affected Application:** NileCare Patient Portal (https://tan-clinic-mono-flood.trycloudflare.com).
* **Affected Endpoint:** `POST /book`.
* **Severity:** High (CVSS v3.1 Score: 7.5 - `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:N/I:H/A:N`).
* **Impact:** Unauthorized appointment creation for arbitrary users, potential schedule/calendar tampering, and loss of integrity in patient booking records.
---
 
## 3. Technical Details & Root Cause Analysis
 
### Root Cause
 
The NileCare application uses a `URLSafeTimedSerializer` (zlib deflated JSON structure) to serialize user identity into the `session` cookie. The server processes booking requests at `/book` by trusting the identity stored within this session object without verifying a cryptographic signature or matching the request identity against server-side session state.
 
### Decoded Session Cookie Payload
 
Decoding the `session` cookie via tools (e.g., `[https://hackindex.io/](https://hackindex.io/)`) reveals the unverified JSON structure:
 
```json
{
  "patient_email": "kk@kk",
  "patient_name": "inAct",
  "user_id": 1322
}
 
```
 
Because the HMAC signature is either not verified or absent during request authorization, an attacker can replace `user_id`, `patient_name`, and `patient_email` with victim parameters (e.g., `ff@ff`) and submit modified booking requests.
 
---
 
## 4. Proof of Concept (PoC) Steps
 
1. **Session Interception & Decoding:**
* Log into the NileCare portal and initiate a appointment booking request using the account kk@kk.
![the first account to initiate the request](<Images/Screenshot 2026-10-29 050538.png>)
 
* Capture the request in Burp Suite and extract the `session` cookie value.
![captured the request in Burp Suite](<Images/Screenshot 2026-10-29 003322.png>)
 
* Decode the session cookie using `https://hackindex.io/` or Flask session decoding utilities, revealing the payload attributes (`patient_email`, `patient_name`, `user_id`).
![photo of the JSON Payload](<Images/Screenshot 2026-10-29 010122.png>)
 
2. **Payload Modification & Encoding:**
* Modify the payload attributes to target the victim account (`ffff` / `ff@ff`):
```json
{
  "patient_email": "ff@ff",
  "patient_name": "ffff",
  "user_id": <VICTIM_ID>
}
 
```
 
 
* Encode the modified JSON structure using the `URLSafeTimedSerializer` format.
3. **Exploitation / Request Submission:**
* Send a `POST /book` request with the modified `session` cookie:
* The server responds with `HTTP/2 302 Found` redirecting to `/dashboard`.
![photo of the request with the modified cookie](<Images/Screenshot 2026-10-29 011527.png>)
 
 
4. **Verification:**
* Accessing the victim's dashboard (`ff@ff`) confirms that the appointment for Dr. Ahmed El-Sherbiny was successfully booked under the victim's account profile.
`ff@ff before the exploitation` 
![ff@ff before the exploitation](<Images/Screenshot 2026-10-29 010027.png>)
 
`ff@ff after the exploitation`
![ff@ff after the exploitation](<Images/Screenshot 2026-10-29 011511.png>)
 
 
---
 
## 5. Impact
 
* **Data Integrity Violation:** Attackers can book unintended medical appointments, clog clinic schedules, and distort medical records for arbitrary users.
* **Resource Exhaustion:** Malicious actors can automate appointment bookings to lock out real patients from booking valid slots.
* **Unauthorized Account Manipulation:** Allows identity impersonation across action endpoints relying on the deserialized session object.
---
 
## 6. Remediation Guidelines
 
1. **Enforce Cryptographic Signing & Validation:**
* Ensure session cookies are signed using a strong, secret key (`SECRET_KEY`) and that signatures are strictly validated on every HTTP request before deserializing state.
2. **Server-Side Session Storage:**
* Transition from storing user identification data inside client-side cookies to storing session state exclusively server-side (e.g., Redis, database session store), referencing it via a cryptographically random session identifier.
3. **Strict Authorization Checks:**
* Validate that the authenticated user performing the action matches the server-side identity associated with the session before executing write operations on `/book`.
`Discovered and tested by Abdulaziz Soltan, aka k0rdz`
 
## 03 — Information Disclosure: Doctors' Details and Email Addresses
 
**Affected functionality:** Public doctors API  
**Endpoint:** `GET /api/doctors`  
**Status:** Confirmed
 
The unauthenticated endpoint returned a directory of 22 doctors with names, specialties, schedules, biographical details, and email addresses under `@nilecare.com.eg`. The information may be intended for public display in part; the exposure of email addresses and structured records should be assessed against the site's intended data-publication policy.
 
**Impact:** Bulk harvesting of staff contact details and schedules; potential targeting for phishing or impersonation.
 
**Technical details — add here**
 
- **Affected URL:** [ADD]
- **Unauthenticated HTTP request:** [ADD]
- **Response excerpt (redacted if needed):** [ADD]
- **Exposed data fields:** [ADD]
- **Screenshots / evidence:** [ADD]
**Recommendation:** Return only fields necessary for public directory functionality. Remove unnecessary staff contact details from unauthenticated API responses.
 
---
 
## 04 — Business Logic Flaw: Client-Controlled Discount
 
**Affected functionality:** Laboratory test booking  
**Endpoint:** `POST /lab-tests/book`  
**Status:** Confirmed, subject to a valid discount code
 
The server accepted a client-supplied `discount_pct` value instead of calculating the allowed discount exclusively on the server. In the documented test, the public code `NILECARE10` was accepted together with `discount_pct=90`, producing a recorded 90% discount. An invalid discount code was rejected.
 
**Impact:** Unauthorized reduction of laboratory booking prices and possible revenue loss.
 
**Technical details — add here**
 
- **Affected URL:** /lab-tests/book
- **Original booking request and expected amount:** ![anyname](<Images/disc pct (2).png>) 
- **Modified parameter / request:** ![anyname](<Images/disc pct (3).png>)
- **Server response and stored result:** ![anyname](<Images/disc pct (4).png>)
**Recommendation:** Calculate discounts and final prices on the server from trusted rules; ignore client-provided percentage values and validate promo-code eligibility and limits.
 
---
 
# 5 - Insecure Password Reset Token Exposure & Permanent Reusability
 
---
 
## 1. Executive Summary
 
A critical security vulnerability resides in the password reset mechanism of the NileCare application (`https://tan-clinic-mono-flood.trycloudflare.com`). While traffic is protected by transport-layer encryption (**HTTPS**), the application introduces severe risks by **explicitly exposing the password reset token in the HTTP response body** and utilizing **static, non-expiring, and reusable tokens** tied permanently to user accounts. This combination enables persistent Account Takeover (ATO) vectors through information disclosure, referrer header leaks, or browser history caching.
 
---
 
## 2. Vulnerability Overview
 
* **Vulnerability Type:** Sensitive Information Disclosure & Permanent/Reusable Password Reset Token (Leading to Account Takeover).
* **Affected Application:** NileCare Patient Portal (`https://tan-clinic-mono-flood.trycloudflare.com`).
* **Affected Endpoint:** `POST /forgot-password`.
* **Severity:** Critical (CVSS v3.1 Score: 8.6 - `CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:U/C:H/I:H/A:N`).
* **Impact:** Full Account Takeover (ATO), persistent unauthorized access, and complete bypass of authentication mechanisms.
---
 
## 3. Technical Details & Root Cause Analysis
 
 
* **Transport Encryption:** Because the application operates over HTTPS, the token cannot be captured via passive plain-text network sniffing (such as packet inspection on a local network).
* **The Actual Flaw (Information Disclosure & Reusability):** The vulnerability does not stem from transit sniffing, but rather from **server-side disclosure**:
1. **Response Body Leak:** When a password reset is requested for an account (`kk@kk`), the server returns the unique token and reset URL directly inside the HTTP response body.
2. **URL Parameter Exposure:** The token is passed as a GET parameter (`?token=1d4db8800978483ef04a31e92f6ed3ca`), making it vulnerable to leakage via `Referer` headers, browser history, or server access logs.
3. **Permanent Lifetime:** The token never expires and is never invalidated after use, meaning once discovered, it can be used repeatedly to perform continuous account takeovers.
---
 
 
## 4. Proof of Concept (PoC) Steps
 
1. **Triggering the Reset Request:**
* Submit a target email address (`kk@kk`) via a `POST` request to the password recovery endpoint.
2. **Observing the Information Disclosure:**
* Inspect the HTTP response body. The application explicitly prints the raw reset URL containing the static token (`1d4db8800978483ef04a31e92f6ed3ca`).
![raw token captured on Burp Suite](<Images/Screenshot 2026-10-29 044404.png>)
 
3. **Executing Account Takeover:**
* Navigate to the URL over HTTPS (`https://tan-clinic-mono-flood.trycloudflare.com/reset-password?token=...`).
* Input a new password to successfully compromise the victim's account.
![navigating to the URL](<Images/Screenshot 2026-10-29 044601.png>)
 
 
4. **Validating Persistent Reusability:**
* Attempt to reuse the exact same token URL again; confirm that **the token remains valid indefinitely**, allowing an attacker to regain entry at any time even after passwords are changed.
![first use of the reset token](<Images/Screenshot 2026-10-29 053958.png>)
 
![second use of the reset token](<Images/Screenshot 2026-10-29 054031.png>)
 
---
 
## 5. Impact
 
* **Account Takeover (ATO):** Any exposure of the response body or URL allows an attacker to instantly take over the victim's account.
* **Permanent Backdoor:** Because the token is static and non-expiring, changing the password does not revoke the token, giving an adversary permanent, long-term access.
---
 
## 6. Remediation Guidelines
 
1. **Eliminate Response Body Disclosure:**
* Never return password reset tokens or reset links directly in HTTP response bodies. Transmit reset links exclusively through secure, out-of-band communication channels (such as verified user emails).
2. **Implement Token Expiration & Single-Use Rules:**
* Generate cryptographically secure, randomized tokens with a short lifespan (e.g., 15 minutes).
* Ensure tokens are **single-use** and are automatically invalidated in the database immediately after a password reset is completed.
3. **Token Rotation:**
* Ensure requesting a new reset token or successfully changing a password invalidates all prior tokens associated with the account.
`Discovered and tested by Abdulaziz Soltan, aka k0rdz`
 
---
 
## 06 — Stored Cross-Site Scripting (XSS) — Medical Report Fields
 
**Affected functionality:** Patient dashboard → Lab Results & Medical Reports → New Report  
**Input:** "Create Medical Report" form (`POST` endpoint: /reports/create)  
**Sink:** `GET /api/v1/reports/view?token=...`  
**Status:** Confirmed
 
> **Note:** This finding is separate from Finding 01. Finding 01 abuses weak file-upload validation (HTML polyglot served from the uploads directory). This finding is caused by missing output encoding of plain-text report fields on the report view page.
 
---
 
### 1. Executive Summary
 
A stored Cross-Site Scripting (XSS) vulnerability exists in the medical report creation feature of the NileCare Patient Portal. The **Report Title**, **Laboratory / Facility**, and **Report Details / Content** fields accept HTML markup and store it unmodified. The dashboard report list displays the stored values as inert text, but the report view page renders them as raw HTML. As a result, an injected HTML element with a JavaScript event handler is rendered and executes in the application's origin whenever the report is opened and the victim interacts with the injected element.
 
---
 
### 2. Vulnerability Overview
 
* **Vulnerability Type:** Stored (Persistent) Cross-Site Scripting — CWE-79.
* **Affected Application:** NileCare Patient Portal (`https://tan-clinic-mono-flood.trycloudflare.com`).
* **Affected Functionality:** Create Medical Report form (`New Report` button on the dashboard).
* **Vulnerable Output Page:** `GET /api/v1/reports/view?token=<report token>`.
* **Affected Fields:** Report Title, Laboratory / Facility, Report Details / Content (exact parameter names: title=, lab=, report_file=).
* **Authentication Required:** Yes (any registered patient account).
* **User Interaction Required:** Yes (the payload used fires on mouse-over of the injected element).
* **Severity:** Medium (CVSS v3.1 Score: 5.4 — `CVSS:3.1/AV:N/AC:L/PR:L/UI:R/S:C/C:L/I:L/A:N`).
* **Impact:** Arbitrary JavaScript execution in the context of the NileCare origin for any user who views the malicious report.
---
 
### 3. Technical Details & Root Cause Analysis
 
The server stores user-supplied report fields without sanitizing or encoding HTML metacharacters (`<`, `>`, `"`). Output handling is inconsistent across the application:
 
* **Dashboard report list:** values are HTML-encoded and shown as literal text (the payload is visible as plain characters, no execution).
* **Report view page (`/api/v1/reports/view`):** the same values are inserted into the page without encoding. The browser parses the injected `<a>` tag, the markup disappears from the visible text, and the title/laboratory/details render as links carrying the attacker-supplied `onmouseover` handler.
The root cause is the lack of context-aware output encoding on the report view page (for example, template auto-escaping disabled or bypassed, or content inserted via an unsafe sink such as `innerHTML`), combined with no server-side input validation on the report fields.
 
**Payloads entered:**
 
Report Title:
```html
\<a onmouseover="alert(document.cookie)"\>xxs here\</a\>
```
 
Laboratory / Facility:
```html
\<a onmouseover="alert(document.cookie)"\>and xxs here\</a\>
```
 
Report Details / Content:
```html
\<a onmouseover="alert(document.cookie)"\>and another xxs here\</a\>
```
 
**Observation on `document.cookie`:** the alert dialog displayed an empty value, which indicates the session cookie is not readable from JavaScript (likely flagged `HttpOnly`). The PoC therefore demonstrates arbitrary script execution, not cookie theft; the impact below is rated accordingly.
 
**Additional observation:** the report view URL uses a `token` parameter such as `eyJ1c2VyX2lkIjogOTMyLCAicmVwb3J0X2lkIjogNjY2NH0=`, which is plain Base64 for `{"user_id": 932, "report_id": 6664}`. The token is unsigned and predictable, which may allow report view links to be forged for other users' reports and would widen the reach of this XSS. This was not tested as part of this finding.
 
---
 
### 4. Proof of Concept (PoC) Steps
 
1. **Open the report creation form:**
   * Log in as a patient (test account `lalala@la.la`), open the **Dashboard**, and click **New Report** under *Lab Results & Medical Reports*.
![Dashboard with the New Report button highlighted](<Images/xss-report-01-dashboard-new-report.png>)
 
2. **Inject the payload into the form fields:**
   * In the **Create Medical Report** dialog, enter the payloads listed above into **Report Title**, **Laboratory / Facility**, and **Report Details / Content**, then click **Save Report**.
![Create Medical Report form filled with the XSS payloads](<Images/xss-report-02-create-report-form.png>)
 
3. **Confirm the payload is stored:**
   * The new report appears in the dashboard list dated 2026-10-29. The list view shows the payload as literal text (encoded), confirming it was stored but not executed here.
![Stored report in the dashboard list showing the payload as text](<Images/xss-report-03-report-list.png>)
 
4. **Trigger the payload:**
   * Click **View** on the report. The application opens `/api/v1/reports/view?token=...`, where the title, laboratory, and details are rendered as HTML. Hovering over the injected link executes the JavaScript, and the browser displays an alert dialog from `tan-clinic-mono-flood.trycloudflare.com`.
![Alert dialog triggered on the report view page](<Images/xss-report-04-alert-on-view-page.png>)
 
---
 
### 5. Impact
 
* **Arbitrary JavaScript execution** in the NileCare origin when a victim opens the malicious report and interacts with the injected element.
* **Actions on behalf of the victim:** script running in the victim's session can send authenticated requests (e.g., booking appointments, changing profile data) and read any data shown on the page, including patient name, email, and phone number.
* **Phishing / content manipulation:** injected content can redirect users or overlay fake forms on a trusted medical portal.
* **Scope of exposure:** as demonstrated, the payload executed in the session of the account that created the report (self-XSS). Cross-user impact depends on whether other users (for example doctors or staff) can open a patient's report view page, or whether view links can be shared or forged (see the token observation above). [ADD result if tested]
* **Session theft limited:** `document.cookie` returned no readable value in testing, so direct cookie theft via this payload was not demonstrated.
---
 
### 6. Remediation Guidelines
 
1. **Apply context-aware output encoding everywhere report data is rendered:**
   * Ensure the report view page HTML-encodes title, laboratory, and details (enable/restore template auto-escaping; avoid `|safe`, `Markup()`, or `innerHTML` for user-supplied data). The dashboard list already encodes correctly and can be used as the reference behavior.
2. **Validate and sanitize input server-side:**
   * Treat these fields as plain text; reject or strip HTML markup. If rich text is required, sanitize with a maintained allow-list sanitizer rather than blacklisting tags or attributes.
3. **Add defense in depth:**
   * Deploy a strict Content-Security-Policy (no `unsafe-inline` scripts, which also blocks inline event handlers).
   * Keep session cookies `HttpOnly`, `Secure`, and `SameSite`.
4. **Review all other pages** that display user-supplied report data (including API responses rendered client-side) for the same issue.
 
