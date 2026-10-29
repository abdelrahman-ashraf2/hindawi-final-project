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

- **Affected URL:** [ADD]
- **Original booking request and expected amount:** [ADD]
- **Modified parameter / request:** [ADD]
- **Server response and stored result:** [ADD]
- **Steps to reproduce:** [ADD]
- **Screenshots / evidence:** [ADD]

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
