# NileCare — Security Findings

**Application:** NileCare / Egyptian Specialist Hospital  
**Assessment period:** 25–27 October 2026  
**Scope:** Web application security review  
**Report status:** Working draft — technical evidence to be added

## Overview

This document summarizes the main security issues identified during testing. The detailed requests, responses, screenshots, and reproduction notes will be added to each finding before the report is finalized.

| ID | Finding | Status |
|---|---|---|
| 01 | Stored Cross-Site Scripting (XSS) | Confirmed 
| 02 | Insecure Direct Object Reference (IDOR) | confirmed 
| 03 | Information Disclosure — Doctor Details and Email Addresses | Confirmed 
| 04 | Business Logic Flaw — Client-Controlled Discount | Confirmed 
| 05 | Forgot Password — Unauthenticated Password Reset / Account Takeover | confirmed 

---

## 01 — Stored Cross-Site Scripting (XSS)

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

# Vulnerability Assessment Report

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

* Decode the session cookie using `[https://hackindex.io/](https://hackindex.io/)` or Flask session decoding utilities, revealing the payload attributes (`patient_email`, `patient_name`, `user_id`).




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


```http
POST /book HTTP/2
Host: tan-clinic-mono-flood.trycloudflare.com
Cookie: session=<MODIFIED_COOKIE_VALUE>
Content-Type: application/x-www-form-urlencoded

name=inAct&email=kk%40kk&phone=0100101010&doctor_id=4&date=2026-12-12&time=11%3A00+AM&notes=

```


* The server responds with `HTTP/2 302 Found` redirecting to `/dashboard`.




4. **Verification:**
* Accessing the victim's dashboard (`ff@ff`) confirms that the appointment for Dr. Ahmed El-Sherbiny / Dr. Hossam Farouk was successfully booked under the victim's account profile.





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

- **Affected URL:** [ADD]
- **Original booking request and expected amount:** [ADD]
- **Modified parameter / request:** [ADD]
- **Server response and stored result:** [ADD]
- **Steps to reproduce:** [ADD]
- **Screenshots / evidence:** [ADD]

**Recommendation:** Calculate discounts and final prices on the server from trusted rules; ignore client-provided percentage values and validate promo-code eligibility and limits.

---

## 05 — Forgot Password: Unauthenticated Password Reset

**Affected functionality:** Forgot-password / password-reset flow  
**Endpoint:** `POST /forgot-password` (entry point; reset action not yet documented)  
**Status:** Reported for follow-up — **not verified in the source report**

**Claim to document:** It may be possible to change another user's password without authenticating as that user. The supplied recon file does not establish this result: it explicitly states that the complete password-reset flow was not exercised and that reset-token security remained unknown. This finding should only be marked **Confirmed** once an authorized test proves the claim.

**Potential impact if confirmed:** Unauthorized password change and account takeover.

**Technical details — add here**

- **Exact vulnerable endpoint:** [ADD]
- **Affected user/account used for authorized test:** [ADD]
- **Authentication state during test:** [ADD]
- **Password-reset request:** [ADD]
- **Password-reset response:** [ADD]
- **Evidence that password changed without authorization:** [ADD]
- **Steps to reproduce:** [ADD]
- **Screenshots / evidence:** [ADD]

**Recommendation:** Bind reset tokens to the intended account, make them cryptographically random, short-lived, and single-use; validate them server-side and revoke active sessions after password changes where appropriate.

---

## Evidence Checklist

For each confirmed issue, attach the original HTTP request, server response, clear reproduction steps, screenshots or screen recordings, the expected versus actual behavior, and a short impact statement. Use only authorized test accounts and redact cookies, passwords, and patient information before publication.

**Note:** This is a shortened working draft based on the supplied recon report. Entries labeled *requires verification* are intentionally not presented as confirmed vulnerabilities.
