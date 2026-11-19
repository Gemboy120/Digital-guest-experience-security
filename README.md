# Cybersecurity Team Sprint Report — Digital Guest Experience Platform

**Project:** Digital Guest Experience Platform (Sprint Group 4)  
**Project Stage:** Pre-build security requirements, risk identification, and architecture guidance *(excludes live penetration testing prior to MVP build)*  
**Status:** Complete (All 5 Deliverables Merged)

## Team Members & Assigned Roles

| Member Name | Assigned Deliverable | Focus Area |
| :--- | :--- | :--- |
| **Muhyideen Husseini (`@MOH_Sec`)** *(Coordinator)* | Deliverable 1 & Master Report Compilation | Platform Security Risk Assessment & Cross-Team Alignment |
| **Oyetoro Oyeleke Joseph** | Deliverable 2 | Authentication & Access Control Specification |
| **Obasi Anselem** & **Muhammad Salihu Fatima Binta** | Deliverable 3 | Guest Data Protection & Nigeria NDPA 2023 Checklist |
| **Patience Ishigbuzo** | Deliverable 4 & Deliverable 5 | Paystack Payment Security & Developer Security Testing Checklist |

---

## Part 1: Team Objective
Identify the Digital Guest Experience Platform's major architectural, operational, integration, and compliance risks prior to development, and define actionable, developer-ready security requirements to protect guest accounts, identity documents, Paystack payment workflows, staff operations, and digital room credentials.

## Part 2: Activities Completed
* Reviewed the Product Requirements Document (PRD) across all 12 functional modules and non-functional requirements (multi-tenancy, 99.5% uptime, Paystack integration, pre-arrival ID verification, digital room access, and NDPA 2023 compliance).
* Allocated the 5 Cybersecurity sprint deliverables across sub-team members and aligned with Product Management, Backend, and Frontend leads.
* Completed **Deliverable 1 (Basic Security Risk Assessment)**, **Deliverable 2 (Authentication & Access Control Specification)**, and **Deliverable 3 (Guest Data Protection & Nigeria NDPA 2023 Checklist)**.
* Completed **Deliverable 4 (Payment Security Recommendations — Paystack Integration)** and **Deliverable 5 (Developer Security Testing Checklist)**.

---

## Part 3: Sprint Deliverables

### Deliverable 1: Basic Security Risk Assessment
**Scope:** Pre-development architectural, operational, and integration risk assessment based on the PRD.  
**Scoring Scale:** **Critical** (immediate severe impact; block pilot if unmitigated) | **High** (major operational/financial/compliance impact; core architecture priority) | **Medium** (moderate impact; standard hardening/rate-limiting).

| Risk ID | Risk & Affected Module | Severity | Threat Scenario | Required Security Control (Mitigation) |
| :--- | :--- | :--- | :--- | :--- |
| **RSK-01** | Multi-Hotel Data Isolation (Backend & Module 9) | **Critical** | Flawed tenant checks allow Hotel A staff to view or alter Hotel B guest records, IDs, or revenue data. | Enforce mandatory `hotel_id` scoping on every database query and API middleware; bind staff tokens to their assigned `hotel_id`; test cross-tenant isolation. |
| **RSK-02** | Third-Party PMS Integration (Module 12) | **High** | A compromised hotel PMS injects malicious payloads or corrupts room/price sync data. | Validate/sanitize all incoming PMS sync payloads; issue unique, revocable API keys per hotel; restrict API scopes to rooms and reservations. |
| **RSK-03** | Offline & Fallback Mode Abuse (Section 8) | **High** | Power/network outages force manual key overrides that bypass ID checks and audit logs. | Require supervisor PIN approval for manual overrides; store encrypted, tamper-evident local logs that auto-sync once online; flag offline check-ins for review. |
| **RSK-04** | Broken Digital Room Access (Module 5) | **Critical** | Predictable QR/room credentials, IDOR bugs, or delayed revocation allow unauthorized room entry. | Issue cryptographically random, single-stay credentials bound to `guest_id`, `booking_id`, and `room_id`; auto-revoke immediately at checkout or cancellation. |
| **RSK-05** | Inventory Locking & API DoS (Modules 2 & 3) | **Medium** | Bots flood booking endpoints to hold rooms without paying via Paystack, blocking real guests. | Apply rate limiting and bot protection on search/booking routes; auto-release unpaid room holds after 10–15 minutes. |
| **RSK-06** | Privileged Staff Misuse (Modules 4 & 9) | **High** | Over-privileged staff or shared front-desk logins expose guest IDs and payment records. | Enforce strict RBAC (Front Desk, Concierge, Housekeeping, Manager, Admin); ban shared logins; require MFA for staff/admins and immutable audit logs. |
| **RSK-07** | ID & Signature Exposure (Module 4) | **Critical** | Public storage buckets, guessable URLs, or indefinite retention expose guest IDs and breach NDPA 2023. | Encrypt files at rest/in transit; use private buckets with short-lived signed URLs; automate deletion/archiving post-stay per NDPA 2023. |
| **RSK-08** | Paystack Webhook Spoofing/Replay (Module 3) | **Critical** | Forged or replayed Paystack webhooks confirm unpaid bookings or trigger duplicate bookings/refunds. | Verify Paystack HMAC-SHA512 signatures on every webhook; confirm status via Paystack API; enforce idempotency keys. |
| **RSK-09** | Account Takeover (Module 1) | **High** | Weak passwords, unthrottled logins, or non-expiring sessions compromise guest or staff accounts. | Hash passwords with Argon2id/bcrypt; rate-limit login/OTP routes; enforce short-lived tokens and automatic session expiry. |
| **RSK-10** | Loyalty & Promo Manipulation (Module 8) | **Medium** | Race conditions or weak validation allow double point redemption or expired promo abuse. | Wrap point accrual/redemption in atomic DB transactions; validate promo dates and usage limits server-side. |
| **RSK-11** | Availability & Backup Integrity (99.5% Uptime) | **High** | Outages, DDoS, or ransomware halt check-ins and room access across all onboarded hotels. | Encrypt automated backups in isolated storage; deploy DDoS protection, automated failover, and uptime alerts. |

---

### Deliverable 2: Authentication & Access Control Recommendations
**Scope:** Pre-build security requirements for login, sessions, RBAC, API authorization, and digital room access (based on OWASP ASVS).

#### 2.1 Roles & Permission Model (Default-Deny, Property-Scoped)

| Role | Permitted Actions & Restrictions |
| :--- | :--- |
| **Guest** | Manage own profile, bookings, pre-arrival check-in, concierge requests, and own active room key. |
| **Receptionist** | Manage reservations, check-in/check-out, and ID verification for assigned property only; no role management. |
| **Concierge** | View/update concierge requests for assigned property; view minimal guest details needed to fulfil requests. |
| **Housekeeping & Maintenance** | View/update assigned room tasks only; no access to payment details or unrelated guest profiles. |
| **Manager** | View property operational reports and manage approved workflows; cannot modify admin security/role settings. |
| **System Administrator** | Manage accounts, roles, property configuration, and security settings (requires strong MFA + audit logging). |

#### 2.2 Core Authentication & Session Rules (`AUTH-01` to `AUTH-07`)
* **Registration & Enumeration (`AUTH-01`):** Verify email/phone identifiers, normalize inputs, and return generic messages on login/reset to block account enumeration.
* **Password Policy (`AUTH-02`):** Hash passwords with Argon2id (or bcrypt fallback) with unique salts; block breached passwords; never log passwords or OTPs.
* **Brute-Force Defense (`AUTH-03`):** Rate-limit login, registration, OTP, and reset routes by account + IP using progressive throttling (avoid permanent lockouts).
* **Multi-Factor Authentication (`AUTH-04`):** Mandate MFA (passkeys/TOTP) for all staff, managers, and admins; enforce single-use, short-lived OTPs for guests; require step-up MFA for role changes or key revocation.
* **Password Reset (`AUTH-05`):** Use single-use, hashed reset tokens (15–30 min expiry); notify account owner and revoke all active sessions upon reset.
* **Session Lifecycle (`AUTH-06`):** Store web tokens in `Secure`, `HttpOnly`, `SameSite` cookies and mobile refresh tokens in Android Keystore / iOS Keychain; rotate refresh tokens with reuse detection; enforce inactivity/absolute timeouts and "logout all devices."
* **Re-Auth & CSRF (`AUTH-07`):** Require recent authentication for sensitive profile/security changes and enforce CSRF protection on cookie-authenticated routes.

#### 2.3 API Authorization & Digital Room Access Controls
1. **Backend Enforcement Order:** Validate token $\rightarrow$ Confirm active user $\rightarrow$ Load server-side roles $\rightarrow$ Verify `hotel_id` scope $\rightarrow$ Verify object ownership/task assignment (prevent IDOR on routes like `GET /bookings/{id}`) $\rightarrow$ Check resource state $\rightarrow$ Execute & write audit log. Frontend UI hiding is never a security boundary.
2. **Digital Room Key Lifecycle:** `Pending` $\rightarrow$ `Eligible` (paid + ID verified) $\rightarrow$ `Issued` $\rightarrow$ `Active` (within local stay window) $\rightarrow$ `Expired/Revoked`. Issue unique credentials per guest/stay/device over TLS; never expose lock-vendor secrets to mobile clients; revoke immediately on checkout, cancellation, lost device, or emergency override.
3. **Audit Logging & Incident Response:** Log logins, MFA events, resets, role changes, denied access, and key issuance/revocation (with UTC timestamps and correlation IDs, zero secrets). On compromise, suspend account, revoke all tokens, and revoke digital keys with lock-vendor confirmation.

#### 2.4 Developer Acceptance Criteria (`AC-01` to `AC-12`)

| ID | Security Requirement | Verification Check |
| :--- | :--- | :--- |
| **AC-01** | Passwords stored only as Argon2id/bcrypt hashes | Inspect DB user record; confirm no plaintext password. |
| **AC-02** | Login failures throttled without leaking account existence | Test repeated bad logins on valid/invalid emails; confirm throttling & generic errors. |
| **AC-03** | Staff/admin endpoints require MFA | Call staff route before MFA completion; confirm `401`/`403` denial. |
| **AC-04** | Object-level ownership enforced (IDOR protection) | Request Guest B's booking ID using Guest A's token; confirm denial. |
| **AC-05** | Role boundaries enforced across staff functions | Call housekeeping/maintenance routes using a Concierge token; confirm denial. |
| **AC-06** | Strict multi-hotel property isolation | Query Property B data using Property A staff credentials; confirm denial. |
| **AC-07** | Privilege escalation blocked | Attempt self-assignment of admin/manager role via API; confirm rejection. |
| **AC-08** | Reset tokens are single-use, time-bound, and revoke sessions | Test expired/reused reset tokens; confirm prior sessions invalidate. |
| **AC-09** | Logout/disablement immediately revokes credentials | Replay access/refresh token after logout or disablement; confirm failure. |
| **AC-10** | Digital keys blocked for unpaid/cancelled/unassigned stays | Attempt key issuance on ineligible bookings; confirm rejection. |
| **AC-11** | Digital keys auto-expire at checkout & support instant revocation | Verify lock-vendor and backend state after checkout and manual revoke. |
| **AC-12** | Sensitive events logged without exposing secrets | Trigger role change and key revoke; confirm audit log has no tokens/secrets. |

---

### Deliverable 3: Guest Data Protection & Nigeria NDPA 2023 Checklist
**Scope:** Privacy, consent, identity document protection, data lifecycle, and regulatory compliance controls under the Nigeria Data Protection Act (NDPA) 2023.

#### 3.1 Guest Data Protection & Privacy Controls
* **Zero Raw Card Data Storage:** Never accept, store, or process raw payment card numbers internally; delegate all card handling to PCI-DSS compliant Paystack checkout and store only transaction references.
* **Lawful Consent & Transparency:** Present a clear privacy notice at registration explaining what guest data is collected, why it is processed, and retention periods; capture explicit, timestamped opt-in consent.
* **AI Personalisation & Marketing Opt-Out:** Provide simple in-app controls allowing guests to opt out of AI-driven stay recommendations and promotional SMS/email notifications at any time without degrading core booking or check-in functionality.
* **Data Integrity & Least-Privilege Access:** Validate guest profile fields server-side, restrict ID/signature viewing strictly to authorized Front Desk and Manager roles (`hotel_id` scoped), and disable unnecessary file downloads.
* **Data Subject Rights (Access, Correction & Deletion):** Provide workflows for guests to view, export, update, or request deletion of their personal profile and historical stay records (subject to statutory hotel/tax record-keeping rules).

#### 3.2 Specific NDPA 2023 Requirements for Guest ID Documents

| NDPA 2023 Requirement Area | Platform Implementation Mandate |
| :--- | :--- |
| **Lawful Basis & Sensitive Data** | Rely on a valid legal basis (statutory hotel/KYC compliance or explicit consent) and apply heightened encryption and access protections to government IDs, NINs, and digital signatures. |
| **Data Minimization & Retention** | Collect only the minimum ID fields required for check-in; enforce automated deletion or secure purging of uploaded ID images and signatures on a strict post-checkout schedule. |
| **Data Protection Impact Assessment (DPIA)** | Conduct and document a formal DPIA prior to onboarding hotels at scale for systematic ID document collection or automated identity verification. |
| **Technical & Organizational Safeguards** | Encrypt ID files and signatures at rest (AES-256) and in transit (TLS 1.2+), store in private cloud buckets served via short-lived signed URLs, and log every staff view event. |
| **72-Hour Breach Notification** | Maintain an incident response playbook to report any unauthorized ID or PII breach to the Nigeria Data Protection Commission (NDPC) within 72 hours and notify affected guests where high risk exists. |
| **Third-Party & Cross-Border Rules** | Execute binding Data Processing Agreements (DPAs) with cloud hosting, SMS/email, and KYC vendors, and ensure approved cross-border transfer safeguards if data is hosted outside Nigeria. |
| **Major Importance Thresholds** | Track guest volume across onboarded hotels to register as a Data Controller/Processor of Major Importance with NDPC, appoint a Data Protection Officer (DPO), and file annual Compliance Audit Returns (CAR) where applicable. |

#### 3.3 Operational Best Practices to Reduce Identity Risk
* **Verify Without Long-Term Storage:** Where feasible, integrate NIMC-licensed verification APIs to confirm guest identity via pass/fail tokens rather than retaining raw ID card photos or PDFs.
* **Ban Insecure Collection Channels:** Strictly prohibit collecting or sharing guest IDs over personal WhatsApp chats, unencrypted email threads, or shared public folders.
* **Field Masking & Redaction:** Mask non-essential sensitive digits (e.g., partial NIN/passport numbers) in standard staff views when full visibility is not legally required.
* **Front-Desk Privacy Training:** Mandate onboarding and periodic NDPA 2023 data-handling training for hotel receptionists and managers, with documented completion logs.

---

### Deliverable 4: Payment Security Recommendations (Paystack Integration)
**Scope:** Payment lifecycle integrity, Paystack webhook verification, idempotency, reconciliation, and refund authorization controls for room bookings and concierge orders.

#### 4.1 Core Paystack Security & Integrity Controls
* **Server-Side Transaction Initialization & Verification:** Initialize Paystack transactions exclusively from the backend using server-validated room/service prices (never trust client-submitted amounts). Never treat a frontend callback alone as conclusive proof of payment; verify transaction status server-side via Paystack's verification API before confirming any booking.
* **Webhook Signature Authentication (HMAC-SHA512):** Validate Paystack's `x-paystack-signature` header using HMAC-SHA512 against the raw request payload and secret key on every incoming webhook event; reject unsigned or invalid requests immediately.
* **Strict Idempotency & Duplicate Prevention:** Assign a unique idempotency key and transaction reference to every payment initiation and webhook event so double-clicks, page refreshes, network retries, or repeated Paystack webhook deliveries never trigger duplicate bookings or double debits.
* **Zero Raw Card Data Storage (PCI-DSS Scope Reduction):** Delegate all card entry and processing to Paystack's hosted/PCI-DSS compliant checkout; store only Paystack transaction references, payment status, timestamps, and hotel/booking metadata.
* **Controlled Refunds & Separation of Duties:** Require dual authorization / manager-level approval for booking refunds or reversals (the staff member initiating a refund cannot self-approve it). Block duplicate refunds or refunds exceeding the original paid amount, and link every reversal to its original ledger entry.
* **Traceable Audit Trail & Reconciliation:** Maintain append-only financial transaction records (no silent editing or deletion of posted entries) and run automated daily reconciliation between internal booking payment logs and Paystack settlement reports.

---

### Deliverable 5: Developer Security Testing Checklist
**Scope:** Pre-launch functional security, authentication, API isolation, payment integrity, and negative edge-case verification across Development, Staging, and UAT environments *(excludes formal third-party penetration testing prior to MVP build)*.

#### 5.1 Pre-Launch Security & Payment Verification Matrix

| Test ID | Domain | Test Scenario | Expected Outcome |
| :--- | :--- | :--- | :--- |
| **TST-01** | Auth & Sessions | Submit weak passwords, repeated invalid logins, or expired/reused OTPs and password reset tokens. | Weak passwords rejected; progressive throttling blocks brute force without account enumeration; expired/reused tokens invalidated and prior sessions revoked. |
| **TST-02** | RBAC & Tenant Isolation | Attempt to access Property B records using Property A staff credentials, or call admin/housekeeping endpoints as a Guest/Concierge. | Backend enforces default-deny `hotel_id` scoping and role boundaries; all cross-tenant and unauthorized role requests return `403 Forbidden`. |
| **TST-03** | Object Ownership (IDOR) | Authenticated as Guest A, request or modify Guest B's booking ID, concierge request, or uploaded ID signed URL. | Backend rejects unauthorized object access (`403`/`404`) and blocks expired or tampered signed URLs. |
| **TST-04** | Digital Room Keys | Attempt to issue a digital key for an unpaid/cancelled stay, or use a key after checkout or manual staff revocation. | Key issuance blocked for ineligible stays; active keys expire automatically at checkout and revoke immediately on command. |
| **TST-05** | Input & API Hardening | Submit SQL injection, XSS payloads, oversized payloads, or unauthenticated state-changing requests without CSRF/TLS. | Malicious inputs sanitized/rejected; HTTPS enforced; CSRF protection blocks forged browser requests; minimal error details returned. |
| **TST-06** | Paystack Webhook Integrity | Send forged webhook signatures, out-of-sequence events, or replay the same successful Paystack webhook 5 times. | Forged signatures rejected; duplicate webhook replays ignored via idempotency keys (only one booking confirmation posted). |
| **TST-07** | Payment Edge Cases & Dubious Debits | Simulate rapid double-clicks on "Pay", zero/negative amounts, network timeout after bank debit, and duplicate refund requests. | Zero/negative amounts rejected; idempotency blocks double charges; timeout triggers server-side Paystack status enquiry; duplicate/over-limit refunds blocked. |
| **TST-08** | Audit Logs & Backups | Trigger role changes, ID views, key revocations, and failed logins; inspect application logs and test encrypted database backup restoration. | Immutable audit records capture actor, `hotel_id`, and UTC timestamp with zero raw passwords, tokens, or card secrets; backup restores cleanly. |

#### 5.2 Defect Severity & Go-Live Release Gate

| Severity | Definition | Release Gate Policy |
| :--- | :--- | :--- |
| **Critical** | Financial loss, duplicate debits, cross-hotel data breach, unauthorized room access, or broken webhook verification. | Mandatory blocker — must be resolved and retested before pilot or production launch. |
| **High** | Major failure in RBAC, MFA enforcement, ID document encryption/retention, or audit logging. | Mandatory blocker — must be resolved before launch. |
| **Medium / Low** | Minor validation, rate-limit tuning, or UI error-message improvements with an approved workaround. | Requires documented PM/Security sign-off and scheduled fix window. |

---

## Part 4: Tools & Frameworks Used
* **Collaboration & Documentation:** Google Docs, GitHub, WhatsApp, Slack
* **Standards & Frameworks:** OWASP ASVS, OWASP Cheat Sheet Series (Auth, Authorization, Session Management, Password Storage, MFA), Qualitative Risk Modeling, Nigeria Data Protection Act (NDPA) 2023

## Part 5: Challenges Encountered
* Defining concrete API, database, and lock-vendor controls prior to finalized Backend schemas required documenting explicit assumptions for developer confirmation.
* Balancing offline/power-outage hotel workflows with strict ID verification and auditability.

## Part 6: Recommendations
1. Mandate server-side `hotel_id` tenant isolation and object-level ownership checks across all APIs prior to pilot launch.
2. Enforce MFA for all staff/admin accounts and prohibit shared front-desk logins.
3. Verify all Paystack payment status changes via HMAC-SHA512 signed webhooks with idempotency keys.
4. Store guest IDs in encrypted private storage using short-lived signed URLs with automated post-stay deletion per NDPA 2023.

## Next Steps
1. Review Backend's database ER diagram, API endpoint specs, and Paystack webhook handler implementation against Deliverables 1–4.
2. Confirm operational parameters with the PM (primary login identifier, session timeout durations, lock-vendor revocation API, and post-checkout ID retention window) and execute Deliverable 5 test cases (`TST-01` to `TST-08`) in Staging prior to pilot launch.
