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

**Affected functionality:** Medical-report viewing  
**Endpoint:** `GET /api/v1/reports/view?token=...`  
**Status:** Requires further verification

The report-view token contains readable Base64-encoded identifiers. The original testing established that the token format is not cryptographically protected, but **did not prove access to another user's medical report**. Attempts to change the `user_id` were rejected; cross-user `report_id` access remained untested or inconclusive.

**Potential impact if confirmed:** Unauthorized access to another patient's medical report or sensitive healthcare information.

**Technical details — add here**

- **Affected URL / object reference:** [ADD]
- **Authorized test accounts:** [ADD]
- **Object ownership comparison:** [ADD]
- **Steps to reproduce:** [ADD]
- **HTTP requests and responses:** [ADD]
- **Evidence of cross-user access:** [ADD]
- **Screenshots:** [ADD]

**Recommendation:** Enforce server-side object-level authorization for every report request and use authenticated, signed, short-lived references where applicable.

---

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
