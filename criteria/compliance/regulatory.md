# Regulatory Compliance Reference Checklist

Canonical reference for evaluating code-level compliance with major regulatory
frameworks. Covers GDPR, HIPAA, PCI-DSS v4.0, SOC 2, FedRAMP, export controls,
and state privacy laws. Each regulation section provides concrete code-level
checkpoints, not just policy-level guidance.

Complements the inlined criteria in `review-compliance.md`. The
`compliance-reviewer` agent uses this file as a lookup when evaluating regulatory
posture in source code, configuration, and infrastructure definitions.

---

## 1. GDPR (General Data Protection Regulation)

The GDPR applies to any organization that processes personal data of individuals
in the European Economic Area (EEA), regardless of where the organization is
located. It imposes strict requirements on data collection, processing, storage,
and transfer.

### 1.1 Data Collection and Lawful Basis

**Regulation reference**: GDPR Art. 6 (Lawful basis), Art. 7 (Consent), Art. 13-14 (Information obligations)

| # | Checkpoint | Severity |
|---|---|---|
| 1.1.1 | Every data collection point (form, API endpoint, SDK event) has a documented lawful basis (consent, contract, legitimate interest, legal obligation, vital interest, public task) | **HIGH** |
| 1.1.2 | Consent collection is explicit, granular, and records a timestamp, the specific consent text shown, and the user's affirmative action | **HIGH** |
| 1.1.3 | Pre-checked consent boxes or implied consent (e.g., "by continuing to use this site you agree") are not used | **MEDIUM** |
| 1.1.4 | Consent is as easy to withdraw as to give (a visible "withdraw consent" path exists) | **HIGH** |
| 1.1.5 | Data collection forms only request fields that are necessary for the stated purpose (data minimization, Art. 5(1)(c)) | **MEDIUM** |
| 1.1.6 | Privacy notice is presented at the point of data collection, not buried in terms of service | **MEDIUM** |
| 1.1.7 | Third-party tracking scripts (analytics, advertising) are not loaded until consent is obtained | **HIGH** |
| 1.1.8 | Cookie consent banners implement actual blocking of non-essential cookies before consent | **HIGH** |
| 1.1.9 | Server-side data collection (logging, analytics events) does not capture personal data beyond what is necessary | **MEDIUM** |
| 1.1.10 | The lawful basis for each data processing activity is documented in a Record of Processing Activities (ROPA) or equivalent | **MEDIUM** |

#### What to Look For

```regex
# Consent collection patterns
(?i)(consent|gdpr|cookie.?consent|privacy.?consent|opt.?in)

# Tracking scripts loaded unconditionally (negative pattern)
(?i)<script[^>]*src=[^>]*(google.?analytics|gtag|facebook|pixel|segment|mixpanel|amplitude|hotjar)

# Pre-checked consent (negative pattern)
(?i)(checked|selected|default)\s*[:=]\s*(true|"true"|'true').*consent

# Data minimization indicators
(?i)(optional|required)\s*[:=].*\b(phone|address|dob|gender|age)\b
```

### 1.2 Data Subject Rights

**Regulation reference**: GDPR Art. 15 (Access), Art. 16 (Rectification), Art. 17 (Erasure), Art. 18 (Restriction), Art. 20 (Portability), Art. 21 (Objection)

| # | Checkpoint | Severity |
|---|---|---|
| 1.2.1 | A mechanism exists for users to request and receive all personal data held about them (right of access, Art. 15) | **HIGH** |
| 1.2.2 | Data export is available in a structured, commonly used, machine-readable format such as JSON or CSV (right to portability, Art. 20) | **MEDIUM** |
| 1.2.3 | A user can request deletion of their account and all associated personal data (right to erasure, Art. 17) | **HIGH** |
| 1.2.4 | Deletion cascades to all systems: primary database, backups (within retention period), caches, search indexes, analytics, third-party services, and logs | **HIGH** |
| 1.2.5 | Deletion requests are processed within 30 days (Art. 12(3)) or the system tracks and enforces this deadline | **MEDIUM** |
| 1.2.6 | Soft-delete implementations eventually hard-delete data (not retained indefinitely in a "deleted" state) | **MEDIUM** |
| 1.2.7 | Users can correct inaccurate personal data (right to rectification, Art. 16) | **MEDIUM** |
| 1.2.8 | Users can object to processing based on legitimate interest (right to object, Art. 21) | **MEDIUM** |
| 1.2.9 | Automated decision-making with legal effects provides for human review (Art. 22) | **HIGH** |
| 1.2.10 | Data subject requests are authenticated before fulfillment to prevent unauthorized access to another person's data | **HIGH** |

#### What to Look For

```regex
# Account deletion / data erasure
(?i)(delete.?account|remove.?account|erase.?data|gdpr.?delete|purge.?user|forget.?me|right.?to.?erasure)

# Soft delete without hard delete
(?i)(soft.?delete|is_deleted|deleted_at|archived)\b
# Verify: a scheduled job or process eventually hard-deletes

# Data export
(?i)(export.?data|download.?data|data.?portability|personal.?data.?export)

# Data subject request handling
(?i)(dsar|data.?subject|subject.?access|data.?request)
```

### 1.3 Data Protection Impact Assessments (DPIAs)

**Regulation reference**: GDPR Art. 35

| # | Checkpoint | Severity |
|---|---|---|
| 1.3.1 | High-risk processing activities (profiling, large-scale monitoring, sensitive data processing) have a documented DPIA | **HIGH** |
| 1.3.2 | DPIAs are referenced in code comments or documentation near the relevant processing logic | **LOW** |
| 1.3.3 | New features involving personal data processing include DPIA consideration in design documents or ADRs | **MEDIUM** |

### 1.4 Data Processing Agreements (DPAs)

**Regulation reference**: GDPR Art. 28

| # | Checkpoint | Severity |
|---|---|---|
| 1.4.1 | Third-party services that process personal data on behalf of the organization have DPAs in place | **HIGH** |
| 1.4.2 | DPA references are documented in infrastructure or configuration (e.g., comments noting the DPA for a cloud service, analytics provider, email service) | **MEDIUM** |
| 1.4.3 | Sub-processors used by third-party services are tracked | **MEDIUM** |
| 1.4.4 | Data sent to third-party services is limited to what is necessary for the service's purpose | **MEDIUM** |

### 1.5 Cross-Border Data Transfers

**Regulation reference**: GDPR Art. 44-49, Chapter V

| # | Checkpoint | Severity |
|---|---|---|
| 1.5.1 | Cloud infrastructure regions are documented, and data residency is intentional (not accidental based on provider defaults) | **HIGH** |
| 1.5.2 | Transfers to countries outside the EEA rely on an adequate legal mechanism: adequacy decision, Standard Contractual Clauses (SCCs), Binding Corporate Rules (BCRs), or derogations | **HIGH** |
| 1.5.3 | US-based services rely on the EU-US Data Privacy Framework (DPF) certification and this is documented | **MEDIUM** |
| 1.5.4 | Infrastructure-as-code configurations specify regions explicitly (not relying on provider defaults that may change) | **MEDIUM** |
| 1.5.5 | CDN and edge computing configurations do not inadvertently cache personal data in non-adequate regions | **MEDIUM** |
| 1.5.6 | Backup and disaster recovery locations are within adequate transfer regions or covered by SCCs | **MEDIUM** |

#### What to Look For

```regex
# Cloud region configuration
(?i)(region|location|zone)\s*[:=]\s*["']?(us-|eu-|ap-|sa-|af-|me-)

# Data residency
(?i)(data.?residen|data.?sovereign|geo.?restrict|region.?lock)

# Transfer mechanisms
(?i)(standard.?contractual|SCC|binding.?corporate|BCR|adequacy|data.?privacy.?framework|privacy.?shield)
```

---

## 2. HIPAA (Health Insurance Portability and Accountability Act)

HIPAA applies to covered entities (healthcare providers, health plans, healthcare
clearinghouses) and their business associates that handle Protected Health
Information (PHI). The Security Rule (45 CFR Part 164, Subpart C) defines
technical safeguards that are directly relevant to code review.

### 2.1 PHI Identification

**Regulation reference**: 45 CFR 160.103 (Definition of PHI), HIPAA Safe Harbor de-identification method (164.514(b))

| # | Checkpoint | Severity |
|---|---|---|
| 2.1.1 | All 18 HIPAA identifiers are cataloged in the data model: names, geographic data, dates (except year), phone numbers, fax numbers, email addresses, SSN, medical record numbers, health plan beneficiary numbers, account numbers, certificate/license numbers, vehicle identifiers, device identifiers, URLs, IP addresses, biometric identifiers, full-face photos, any other unique identifying number | **HIGH** |
| 2.1.2 | Database schemas and data models clearly mark PHI fields (through naming convention, annotations, or documentation) | **HIGH** |
| 2.1.3 | PHI fields are not included in general-purpose search indexes without access control | **HIGH** |
| 2.1.4 | PHI is not stored in browser local storage, session storage, or cookies | **CRITICAL** |
| 2.1.5 | PHI is not included in URLs (query parameters, path segments) where it appears in server logs, browser history, and referrer headers | **HIGH** |
| 2.1.6 | De-identification follows the HIPAA Safe Harbor method (all 18 identifiers removed) or Expert Determination method (164.514(a)) | **MEDIUM** |
| 2.1.7 | Test environments do not contain real PHI (use synthetic or de-identified data) | **CRITICAL** |
| 2.1.8 | Error messages, stack traces, and debug output do not expose PHI | **HIGH** |

#### What to Look For

```regex
# PHI field names in data models
(?i)(patient.?name|medical.?record|mrn|diagnosis|treatment|prescription|health.?plan|beneficiary|ssn|social.?security|date.?of.?birth|dob)\b

# PHI in URLs (negative pattern)
(?i)(patient|diagnosis|ssn|mrn|medical)\s*[:=]\s*(req|request)\.(params|query|url)

# PHI in logs (negative pattern)
(?i)(log|logger|console)\.\w+\(.*\b(patient|diagnosis|ssn|mrn|medical|phi)\b

# PHI in client storage (negative pattern)
(?i)(localStorage|sessionStorage)\.(setItem|getItem)\(.*\b(patient|medical|health|diagnosis)\b
```

### 2.2 Encryption Requirements

**Regulation reference**: 45 CFR 164.312(a)(2)(iv) (Encryption at rest), 164.312(e)(1) (Transmission security)

| # | Checkpoint | Severity |
|---|---|---|
| 2.2.1 | PHI at rest is encrypted using AES-128 or AES-256 (or equivalent NIST-approved algorithm) | **CRITICAL** |
| 2.2.2 | PHI in transit uses TLS 1.2 or higher with strong cipher suites | **CRITICAL** |
| 2.2.3 | Encryption keys for PHI are managed in a dedicated key management system (KMS), not stored alongside the encrypted data | **HIGH** |
| 2.2.4 | Database-level encryption (TDE) or field-level encryption is enabled for databases containing PHI | **HIGH** |
| 2.2.5 | Backup media containing PHI is encrypted | **HIGH** |
| 2.2.6 | Encryption key rotation is implemented on a defined schedule | **MEDIUM** |
| 2.2.7 | File storage services (S3, GCS, Azure Blob) containing PHI use server-side encryption with customer-managed keys | **HIGH** |
| 2.2.8 | Mobile applications and portable devices encrypt PHI at the application level (not relying solely on device encryption) | **HIGH** |

### 2.3 Access Controls

**Regulation reference**: 45 CFR 164.312(a)(1) (Access control), 164.312(d) (Authentication)

| # | Checkpoint | Severity |
|---|---|---|
| 2.3.1 | Unique user identification: each user has a unique identifier for tracking (164.312(a)(2)(i)) | **HIGH** |
| 2.3.2 | Emergency access procedure: a documented break-glass mechanism exists for emergency PHI access (164.312(a)(2)(ii)) | **MEDIUM** |
| 2.3.3 | Automatic logoff: sessions expire after a period of inactivity (164.312(a)(2)(iii)) | **MEDIUM** |
| 2.3.4 | Role-based access control restricts PHI access to the minimum necessary for job function (Minimum Necessary Rule, 164.502(b)) | **HIGH** |
| 2.3.5 | Multi-factor authentication is required for remote access to systems containing PHI | **HIGH** |
| 2.3.6 | Access to PHI is controlled at the row level or record level (not just table-level permissions) | **MEDIUM** |
| 2.3.7 | API endpoints serving PHI enforce authentication and authorization checks | **CRITICAL** |
| 2.3.8 | Administrative access to systems containing PHI requires additional authentication (not the same credentials as regular access) | **HIGH** |

### 2.4 Business Associate Agreements (BAAs)

**Regulation reference**: 45 CFR 164.502(e), 164.504(e)

| # | Checkpoint | Severity |
|---|---|---|
| 2.4.1 | All third-party services that receive, store, or process PHI have a BAA in place | **CRITICAL** |
| 2.4.2 | Cloud provider BAA is executed (AWS, Azure, GCP all offer BAAs but they must be explicitly signed) | **HIGH** |
| 2.4.3 | Third-party SaaS tools used for PHI (logging, monitoring, communication) are HIPAA-eligible and have BAAs | **HIGH** |
| 2.4.4 | BAA coverage is documented in infrastructure configuration or architecture documentation | **MEDIUM** |
| 2.4.5 | Services without BAAs do not receive PHI (verified by data flow analysis) | **HIGH** |

### 2.5 Audit Logging

**Regulation reference**: 45 CFR 164.312(b) (Audit controls)

| # | Checkpoint | Severity |
|---|---|---|
| 2.5.1 | All access to PHI is logged: who accessed it, what was accessed, when, and the outcome | **HIGH** |
| 2.5.2 | Audit logs themselves do not contain PHI (log the record ID, not the record contents) | **HIGH** |
| 2.5.3 | Audit logs are tamper-evident (append-only, forwarded to a separate system, or cryptographically chained) | **HIGH** |
| 2.5.4 | Audit log retention meets HIPAA requirements (minimum 6 years for policies and procedures; audit logs should align with organizational policy, typically 6-7 years) | **MEDIUM** |
| 2.5.5 | Audit logs are regularly reviewed (automated alerting for anomalous access patterns) | **MEDIUM** |
| 2.5.6 | Failed access attempts to PHI are logged and trigger alerts above a threshold | **MEDIUM** |

#### What to Look For

```regex
# Audit logging for PHI access (positive indicators)
(?i)(audit.?log|access.?log|phi.?access|hipaa.?audit|access.?trail)

# Logging PHI content (negative pattern)
(?i)(log|audit|track)\.(info|debug|warn|error)\(.*\b(patient.?name|ssn|diagnosis|mrn|medical.?record)\b

# BAA references (positive indicators)
(?i)(baa|business.?associate|hipaa.?eligible|hipaa.?compliant)

# Session timeout configuration
(?i)(session.?timeout|idle.?timeout|inactivity.?timeout)\s*[:=]\s*\d+
```

---

## 3. PCI-DSS v4.0 (Payment Card Industry Data Security Standard)

PCI-DSS v4.0 applies to all entities that store, process, or transmit cardholder
data (CHD) or sensitive authentication data (SAD). The standard defines 12
requirements across 6 control objectives. This section focuses on the
requirements most relevant to code and application security.

### 3.1 Cardholder Data Scoping

**Regulation reference**: PCI-DSS v4.0 Req. 3 (Protect Stored Account Data)

| # | Checkpoint | Severity |
|---|---|---|
| 3.1.1 | The cardholder data environment (CDE) is clearly defined: which systems store, process, or transmit CHD | **HIGH** |
| 3.1.2 | Primary Account Number (PAN) is never stored in full after authorization unless there is a documented business need | **CRITICAL** |
| 3.1.3 | Sensitive authentication data (full track, CVV/CVC, PIN) is never stored after authorization, even if encrypted (Req. 3.3.1) | **CRITICAL** |
| 3.1.4 | PAN is rendered unreadable anywhere it is stored: truncation, tokenization, one-way hashing, or strong cryptography (Req. 3.5.1) | **CRITICAL** |
| 3.1.5 | PAN is not stored in log files, debug output, or error messages (Req. 3.4.1) | **CRITICAL** |
| 3.1.6 | PAN is not stored in cookies, hidden form fields, or URL parameters | **CRITICAL** |
| 3.1.7 | Data retention policies exist and are enforced: CHD is deleted when no longer needed for a business or legal purpose (Req. 3.2.1) | **HIGH** |
| 3.1.8 | Display of PAN is masked (show first 6 and last 4 digits at most) except for users with a documented business need (Req. 3.4.1) | **HIGH** |
| 3.1.9 | Tokenization is used to avoid bringing CHD into the application scope where possible | **MEDIUM** |
| 3.1.10 | Scope reduction through a payment gateway or processor (e.g., Stripe, Braintree) is documented and verified | **MEDIUM** |

#### What to Look For

```regex
# Credit card number patterns in code (negative indicators)
(?i)(card.?number|pan|credit.?card|cc_num|ccnum|card_num)\s*[:=]

# CVV/CVC storage (CRITICAL - never store after auth)
(?i)(cvv|cvc|cvv2|cvc2|security.?code|card.?verification)\s*[:=]

# PAN in logs
(?i)(log|logger|console)\.\w+\(.*\b(card.?number|pan|credit.?card|cc_num)\b

# PAN masking patterns (positive indicators)
(?i)(mask|truncat|redact)\w*\(.*\b(card|pan|account)\b

# Tokenization (positive indicators)
(?i)(tokenize|payment.?token|card.?token|vault.?token|stripe.?token)
```

### 3.2 Encryption Requirements

**Regulation reference**: PCI-DSS v4.0 Req. 3.5 (PAN is secured wherever stored), Req. 4 (Protect CHD with strong cryptography during transmission)

| # | Checkpoint | Severity |
|---|---|---|
| 3.2.1 | PAN at rest is encrypted with AES-256 or equivalent (Req. 3.5.1) | **CRITICAL** |
| 3.2.2 | Encryption keys are managed per Req. 3.6: generated using strong cryptographic methods, stored securely, rotated on a defined schedule, and split-knowledge/dual-control procedures apply | **HIGH** |
| 3.2.3 | Key-encrypting keys (KEKs) are at least as strong as the data-encrypting keys (DEKs) they protect | **HIGH** |
| 3.2.4 | CHD in transit over public networks uses TLS 1.2+ with strong cipher suites (Req. 4.2.1) | **CRITICAL** |
| 3.2.5 | CHD is not sent over end-user messaging (email, chat, SMS) (Req. 4.2.2) | **HIGH** |
| 3.2.6 | Wireless networks transmitting CHD use industry best practices for strong encryption (not WEP) | **HIGH** |
| 3.2.7 | Disk-level encryption alone is not sufficient for PAN protection; additional file/column/field encryption is required on removable media and non-production use (Req. 3.5.1.1) | **MEDIUM** |

### 3.3 Access Control

**Regulation reference**: PCI-DSS v4.0 Req. 7 (Restrict access by business need), Req. 8 (Identify users and authenticate)

| # | Checkpoint | Severity |
|---|---|---|
| 3.3.1 | Access to CHD is restricted by role-based access control (RBAC) to only those with documented business need (Req. 7.2) | **HIGH** |
| 3.3.2 | All user accounts have unique IDs (no shared or generic accounts for CDE access) (Req. 8.2.1) | **HIGH** |
| 3.3.3 | Multi-factor authentication is required for all access into the CDE (Req. 8.4.2) | **HIGH** |
| 3.3.4 | Multi-factor authentication is required for all remote network access (Req. 8.4.3) | **HIGH** |
| 3.3.5 | Password/passphrase complexity meets Req. 8.3.6: minimum 12 characters (or 8 if the system does not support 12), containing both numeric and alphabetic characters | **MEDIUM** |
| 3.3.6 | Accounts are locked after no more than 10 invalid login attempts (Req. 8.3.4) | **MEDIUM** |
| 3.3.7 | Session idle timeout is set to 15 minutes or less for CDE access (Req. 8.2.8) | **MEDIUM** |
| 3.3.8 | Application and system accounts used by automated processes have restricted privileges and are not used interactively (Req. 8.6.1) | **MEDIUM** |
| 3.3.9 | Default passwords on systems and applications are changed before deployment (Req. 2.2.2) | **CRITICAL** |

### 3.4 Monitoring and Logging

**Regulation reference**: PCI-DSS v4.0 Req. 10 (Log and monitor all access)

| # | Checkpoint | Severity |
|---|---|---|
| 3.4.1 | Audit trails record all individual user access to CHD (Req. 10.2.1) | **HIGH** |
| 3.4.2 | Logs capture: user identification, type of event, date/time, success or failure, origination of event, identity or name of affected data/system/resource (Req. 10.2.1.1-10.2.1.7) | **HIGH** |
| 3.4.3 | All administrative actions are logged (Req. 10.2.1.2) | **HIGH** |
| 3.4.4 | Invalid logical access attempts are logged (Req. 10.2.1.4) | **MEDIUM** |
| 3.4.5 | Changes to identification and authentication mechanisms are logged (Req. 10.2.1.5) | **MEDIUM** |
| 3.4.6 | Logs are reviewed at least daily using automated mechanisms (Req. 10.4.1) | **MEDIUM** |
| 3.4.7 | Audit trail history is retained for at least 12 months, with at least 3 months immediately available for analysis (Req. 10.5.1) | **HIGH** |
| 3.4.8 | Time synchronization (NTP/PTP) is configured on all CDE systems (Req. 10.6) | **MEDIUM** |
| 3.4.9 | Logs are protected from unauthorized modification (Req. 10.3.2) | **HIGH** |
| 3.4.10 | A mechanism exists to detect and alert on security-relevant events (Req. 10.4.1.1) | **MEDIUM** |

#### What to Look For

```regex
# Audit logging for payment data (positive indicators)
(?i)(payment.?audit|transaction.?log|card.?access.?log|pci.?audit)

# Time synchronization
(?i)(ntp|chrony|timesyncd|w32time|PTP|precision.?time)

# Log retention configuration
(?i)(retention|rotate|expire|ttl|max.?age).*\b(log|audit)\b
```

---

## 4. SOC 2 (Service Organization Control 2)

SOC 2 audits evaluate an organization's information systems against the Trust
Services Criteria (TSC) defined by the AICPA. While SOC 2 is an audit framework
rather than a regulation, code-level indicators directly support or undermine the
five trust service categories.

### Trust Service Criteria Touchpoints in Code

| # | Trust Criteria | Code-Level Checkpoint | Severity |
|---|---|---|---|
| 4.1 | **Security (CC6)**: Logical access controls | Role-based access control is implemented; all endpoints require authentication; principle of least privilege is enforced | **HIGH** |
| 4.2 | **Security (CC6)**: Encryption | Data at rest and in transit is encrypted using industry-standard algorithms | **HIGH** |
| 4.3 | **Security (CC7)**: System monitoring | Security events are logged, monitored, and alert on anomalies; intrusion detection is in place | **HIGH** |
| 4.4 | **Security (CC8)**: Change management | Code changes go through documented review and approval; CI/CD pipelines enforce gates | **MEDIUM** |
| 4.5 | **Availability (A1)**: System availability | Health checks, circuit breakers, retry logic, and failover mechanisms exist; SLA monitoring is implemented | **MEDIUM** |
| 4.6 | **Availability (A1)**: Backup and recovery | Backup procedures are automated and tested; disaster recovery plans reference the code and infrastructure | **MEDIUM** |
| 4.7 | **Processing Integrity (PI1)**: Data processing accuracy | Input validation, data quality checks, reconciliation mechanisms, and idempotency patterns exist | **MEDIUM** |
| 4.8 | **Confidentiality (C1)**: Data classification | Sensitive data is identified, classified, and handled according to classification (encryption, access control, masking) | **HIGH** |
| 4.9 | **Confidentiality (C1)**: Data disposal | Retention policies are enforced; data is securely deleted when no longer needed | **MEDIUM** |
| 4.10 | **Privacy (P1-P8)**: Privacy practices | Consent management, data subject rights, privacy notices, data minimization (overlaps with GDPR Section 1) | **HIGH** |

### SOC 2 Evidence in Code

Auditors look for evidence that controls are designed and operating effectively.
The following code-level artifacts support SOC 2 evidence collection:

| Evidence Type | What Auditors Look For | Code-Level Indicator |
|---|---|---|
| Access control configuration | Who can access what, and is it least-privilege | RBAC definitions, IAM policies, middleware authorization checks |
| Change management records | Are changes reviewed and approved before deployment | PR review requirements, CI/CD approval gates, branch protection rules |
| Monitoring and alerting | Are anomalies detected and responded to | Alerting rules, monitoring dashboards as code, incident response runbooks |
| Encryption configuration | Is data encrypted appropriately | TLS configuration, database encryption settings, KMS references |
| Backup configuration | Are backups automated, tested, and encrypted | Backup job definitions, restore test scripts, encryption configuration |
| Vulnerability management | Are vulnerabilities identified and remediated | Dependency scanning in CI, CVE tracking, patch cadence |

### What to Look For

```regex
# SOC 2 relevant patterns
(?i)(soc.?2|trust.?service|aicpa|service.?organization)

# Change management indicators
(?i)(required.?reviewers|branch.?protection|approval.?required|code.?review.?required|CODEOWNERS)

# Availability patterns
(?i)(health.?check|readiness.?probe|liveness.?probe|circuit.?breaker|retry.?policy|failover)

# Data classification
(?i)(classification|sensitivity|confidential|internal|public|restricted|pii.?tag|data.?class)
```

---

## 5. FedRAMP (Federal Risk and Authorization Management Program)

FedRAMP requires cloud service providers serving federal agencies to implement
NIST SP 800-53 security controls. The following NIST 800-53 control families are
most relevant to code review. Control identifiers reference NIST 800-53 Rev. 5.

### 5.1 Access Control (AC)

**Control reference**: NIST 800-53 AC-2, AC-3, AC-5, AC-6, AC-7, AC-11, AC-12, AC-17

| # | Checkpoint | Control | Severity |
|---|---|---|---|
| 5.1.1 | Account management: provisioning, deprovisioning, and review of user accounts is implemented (not just manual) | AC-2 | **HIGH** |
| 5.1.2 | Separation of duties is enforced: no single user can approve and deploy their own changes | AC-5 | **HIGH** |
| 5.1.3 | Least privilege: application service accounts and user roles have minimum necessary permissions | AC-6 | **HIGH** |
| 5.1.4 | Unsuccessful login attempts are limited (lockout or progressive delay after threshold) | AC-7 | **MEDIUM** |
| 5.1.5 | Session lock: sessions are locked after a period of inactivity (configurable, typically 15 minutes for Moderate baseline) | AC-11 | **MEDIUM** |
| 5.1.6 | Session termination: sessions are terminated after a defined maximum duration or on user logout | AC-12 | **MEDIUM** |
| 5.1.7 | Remote access: all remote access uses encrypted channels (VPN, SSH, TLS) with MFA | AC-17 | **HIGH** |
| 5.1.8 | Access enforcement: every request is checked against the access control policy (no default-allow) | AC-3 | **HIGH** |

### 5.2 Audit and Accountability (AU)

**Control reference**: NIST 800-53 AU-2, AU-3, AU-6, AU-8, AU-9, AU-12

| # | Checkpoint | Control | Severity |
|---|---|---|---|
| 5.2.1 | Auditable events are defined and implemented: login/logout, failed access, privilege escalation, data access, configuration changes | AU-2 | **HIGH** |
| 5.2.2 | Audit records contain: what happened, when, where, source, outcome, and identity of subjects/objects | AU-3 | **HIGH** |
| 5.2.3 | Audit records are reviewed and analyzed (automated correlation, SIEM integration) | AU-6 | **MEDIUM** |
| 5.2.4 | Timestamps use a reliable time source (NTP synchronized, UTC) | AU-8 | **MEDIUM** |
| 5.2.5 | Audit records are protected from unauthorized access, modification, and deletion | AU-9 | **HIGH** |
| 5.2.6 | Audit record generation: the system generates audit records for the defined auditable events | AU-12 | **HIGH** |

### 5.3 Identification and Authentication (IA)

**Control reference**: NIST 800-53 IA-2, IA-5, IA-8

| # | Checkpoint | Control | Severity |
|---|---|---|---|
| 5.3.1 | Multi-factor authentication for privileged accounts | IA-2(1) | **HIGH** |
| 5.3.2 | Multi-factor authentication for non-privileged accounts accessing the system remotely | IA-2(2) | **HIGH** |
| 5.3.3 | Authenticator management: passwords are hashed using approved algorithms (bcrypt, scrypt, Argon2, PBKDF2 with sufficient iterations) | IA-5 | **HIGH** |
| 5.3.4 | Password complexity and length requirements are enforced (minimum 12 characters for Moderate baseline) | IA-5(1) | **MEDIUM** |
| 5.3.5 | Federated identity: external user authentication uses SAML, OIDC, or equivalent approved protocol | IA-8 | **MEDIUM** |

### 5.4 System and Communications Protection (SC)

**Control reference**: NIST 800-53 SC-7, SC-8, SC-12, SC-13, SC-28

| # | Checkpoint | Control | Severity |
|---|---|---|---|
| 5.4.1 | Boundary protection: network segmentation isolates the system from external and less-trusted networks | SC-7 | **HIGH** |
| 5.4.2 | Transmission confidentiality: all data in transit is encrypted using FIPS 140-2/140-3 validated modules | SC-8 | **CRITICAL** |
| 5.4.3 | Cryptographic key management: keys are generated, distributed, stored, rotated, and destroyed per organizational policy | SC-12 | **HIGH** |
| 5.4.4 | Cryptographic protection: FIPS-validated cryptographic modules are used (not custom or non-validated implementations) | SC-13 | **CRITICAL** |
| 5.4.5 | Protection of information at rest: data at rest is encrypted using FIPS-validated cryptography | SC-28 | **HIGH** |

### 5.5 System and Information Integrity (SI)

**Control reference**: NIST 800-53 SI-2, SI-3, SI-4, SI-5, SI-10

| # | Checkpoint | Control | Severity |
|---|---|---|---|
| 5.5.1 | Flaw remediation: a process exists for identifying, reporting, and correcting software flaws (vulnerability scanning, dependency updates) | SI-2 | **HIGH** |
| 5.5.2 | Malicious code protection: anti-malware or equivalent controls for container image scanning, dependency scanning | SI-3 | **HIGH** |
| 5.5.3 | System monitoring: intrusion detection, anomaly detection, and security event monitoring are implemented | SI-4 | **HIGH** |
| 5.5.4 | Security alerts: the system receives and acts on security advisories (e.g., CVE alerts for dependencies) | SI-5 | **MEDIUM** |
| 5.5.5 | Input validation: the system validates all inputs for type, length, range, and format | SI-10 | **HIGH** |

#### FedRAMP-Specific What to Look For

```regex
# FIPS mode indicators
(?i)(FIPS|fips.?mode|FIPS.?140|FIPS_mode_set|GOFIPS|OPENSSL_FIPS)

# FIPS-validated algorithms
(?i)(AES-128|AES-256|SHA-256|SHA-384|SHA-512|RSA-2048|RSA-4096|ECDSA|ECDH)

# Non-FIPS algorithms (may need replacement in FedRAMP context)
(?i)(MD5|SHA-1|DES|3DES|RC4|Blowfish)\b
# Note: SHA-1 and MD5 are not approved for digital signatures in FIPS

# Network segmentation indicators
(?i)(security.?group|network.?policy|NetworkPolicy|firewall.?rule|nacl|nsg|calico|cilium)

# NIST control references (positive documentation indicator)
(?i)(NIST|800-53|SP.?800|AC-\d|AU-\d|IA-\d|SC-\d|SI-\d)
```

---

## 6. Export Controls

Export control regulations restrict the transfer of certain technologies,
including cryptographic software, across national borders. These regulations
apply to both the distribution of software and the use of cryptographic
algorithms within software.

### 6.1 EAR (Export Administration Regulations)

**Regulation reference**: 15 CFR Parts 730-774, ECCN 5D002

| # | Checkpoint | Severity |
|---|---|---|
| 6.1.1 | Software using non-standard or proprietary encryption algorithms is classified under ECCN 5D002 and may require an export license | **HIGH** |
| 6.1.2 | Publicly available open-source software with encryption is exempt from EAR licensing if proper notification is filed with BIS (Bureau of Industry and Security) and NSA (15 CFR 740.13(e)) | **MEDIUM** |
| 6.1.3 | The project's encryption capabilities are documented for export classification purposes | **MEDIUM** |
| 6.1.4 | Software is not distributed to embargoed countries (Cuba, Iran, North Korea, Syria, Crimea) without proper license | **CRITICAL** |
| 6.1.5 | App store distribution: the Apple App Store and Google Play encryption declarations are completed accurately | **MEDIUM** |

### 6.2 ITAR (International Traffic in Arms Regulations)

**Regulation reference**: 22 CFR Parts 120-130

| # | Checkpoint | Severity |
|---|---|---|
| 6.2.1 | If the software is specifically designed for military applications or defense articles, it falls under ITAR, not EAR | **CRITICAL** |
| 6.2.2 | ITAR-controlled technical data is not accessible to foreign nationals without authorization ("deemed export") | **CRITICAL** |
| 6.2.3 | ITAR-controlled software is not hosted on foreign servers or in cloud regions outside the US without authorization | **CRITICAL** |
| 6.2.4 | Access controls prevent foreign national access to ITAR-controlled repositories | **CRITICAL** |

### 6.3 Cryptography Classifications

| Cryptography Type | Export Control Status | Notes |
|---|---|---|
| Mass-market encryption (TLS, HTTPS, SSH) | Generally exempt under EAR 740.17(b) | Must file annual classification request or self-classification |
| Standard open-source encryption | Exempt under 15 CFR 740.13(e) | Notify BIS and NSA via email with URL to public source |
| Proprietary encryption algorithms | Requires ECCN classification, may need license | Custom encryption is a red flag for both export control and security |
| Key length > 56 bits symmetric / > 512 bits asymmetric | Subject to EAR if not mass-market or open-source | Most modern algorithms exceed these thresholds |
| Encryption for military/intelligence use | ITAR Category XIII | Requires State Department license |

### What to Look For

```regex
# Cryptographic algorithm usage
(?i)(AES|RSA|ECC|ECDSA|ECDH|ChaCha20|Poly1305|Curve25519|Ed25519|SHA-256|SHA-512|HMAC|PBKDF2|scrypt|argon2|bcrypt)

# Custom/proprietary encryption (red flag)
(?i)(custom.?encrypt|proprietary.?cipher|home.?brew.?crypto|my.?encrypt)

# Encryption libraries
(?i)(openssl|boringssl|libsodium|nacl|ring|crypto|cryptography|javax\.crypto|System\.Security\.Cryptography|mbedtls)

# Export control markers
(?i)(export.?control|ear|itar|eccn|munitions|defense.?article|deemed.?export)

# Embargoed country references
(?i)(cuba|iran|north.?korea|syria|crimea).*\b(block|restrict|embargo|sanction)\b
```

---

## 7. CCPA / State Privacy Laws

The California Consumer Privacy Act (CCPA), as amended by the CPRA, and similar
state privacy laws (Virginia VCDPA, Colorado CPA, Connecticut CTDPA, Utah UCPA,
Texas TDPSA, Oregon OCPA, Montana MCDPA, Iowa ICDPA, Indiana ICDPA, Tennessee
TIPA) impose consumer privacy requirements that affect application code.

### 7.1 CCPA/CPRA (California)

**Regulation reference**: Cal. Civ. Code 1798.100-1798.199

| # | Checkpoint | Severity |
|---|---|---|
| 7.1.1 | A "Do Not Sell or Share My Personal Information" link is visible on the website/app if the business sells or shares personal information | **HIGH** |
| 7.1.2 | Global Privacy Control (GPC) signals are detected and honored (CCPA 1798.135(e)) | **HIGH** |
| 7.1.3 | Consumers can request access to their personal information collected in the prior 12 months | **HIGH** |
| 7.1.4 | Consumers can request deletion of their personal information | **HIGH** |
| 7.1.5 | Consumers can opt out of the sale or sharing of their personal information | **HIGH** |
| 7.1.6 | The right to opt out applies to cross-context behavioral advertising (CPRA addition) | **MEDIUM** |
| 7.1.7 | Sensitive personal information (SSN, financial account, geolocation, racial/ethnic origin, biometric data, health data, sex life/orientation) has additional protections and a "Limit Use" option | **HIGH** |
| 7.1.8 | Data retention periods are disclosed and enforced for each category of personal information (CPRA addition) | **MEDIUM** |
| 7.1.9 | Service provider and contractor contracts include CCPA-required provisions | **MEDIUM** |
| 7.1.10 | Privacy notices are updated at least annually | **LOW** |

### 7.2 Cross-State Compliance Checkpoints

The following checkpoints address requirements common across multiple state
privacy laws:

| # | Checkpoint | Applicable Laws | Severity |
|---|---|---|---|
| 7.2.1 | Right to access personal information | CCPA, VCDPA, CPA, CTDPA, all others | **HIGH** |
| 7.2.2 | Right to delete personal information | CCPA, VCDPA, CPA, CTDPA, all others | **HIGH** |
| 7.2.3 | Right to data portability | CCPA, VCDPA, CPA, CTDPA | **MEDIUM** |
| 7.2.4 | Right to opt out of targeted advertising | CCPA (sale/share), VCDPA, CPA, CTDPA | **HIGH** |
| 7.2.5 | Right to opt out of profiling | VCDPA, CPA, CTDPA | **MEDIUM** |
| 7.2.6 | Right to correct inaccurate information | CCPA (CPRA), VCDPA, CPA, CTDPA | **MEDIUM** |
| 7.2.7 | Universal opt-out mechanisms (GPC) recognized | CCPA, CPA, CTDPA, Montana, Texas | **MEDIUM** |
| 7.2.8 | Data protection assessments for high-risk processing | VCDPA, CPA, CTDPA | **MEDIUM** |
| 7.2.9 | Non-discrimination for exercising privacy rights | All state privacy laws | **HIGH** |
| 7.2.10 | Consent required for processing sensitive data | VCDPA, CPA, CTDPA (opt-in); CCPA (opt-out/limit) | **HIGH** |

### What to Look For

```regex
# CCPA/privacy signals
(?i)(do.?not.?sell|do.?not.?share|privacy.?rights|ccpa|cpra|consumer.?privacy|opt.?out)

# Global Privacy Control
(?i)(global.?privacy.?control|gpc|Sec-GPC|navigator\.globalPrivacyControl)

# Targeted advertising / cross-context behavioral tracking
(?i)(targeted.?advertis|cross.?context|behavioral.?advertis|retarget|lookalike|audience.?segment)

# Sensitive data categories
(?i)(precise.?geolocation|biometric|genetic|racial|ethnic|sexual.?orientation|health.?data|financial.?account)
```

---

## 8. Cross-Regulation Code Indicators

Many code patterns are relevant to multiple regulatory frameworks. This section
provides a consolidated view of cross-cutting concerns.

### Data Classification Tags

| Classification | Applicable Regulations | Code Treatment Required |
|---|---|---|
| **Public** | None | No special handling |
| **Internal** | SOC 2 (C1) | Access control; no external exposure |
| **Confidential** | SOC 2 (C1), FedRAMP (SC-28) | Encryption at rest and in transit; access logging |
| **PII** | GDPR, CCPA, SOC 2 (P1) | Consent, access rights, deletion, encryption, minimization |
| **PHI** | HIPAA, SOC 2 (C1) | All PII requirements plus BAA, audit logging, encryption |
| **CHD** | PCI-DSS | Encryption, masking, tokenization, minimal storage, CDE isolation |
| **CUI** | FedRAMP, NIST 800-171 | FIPS encryption, access control, audit logging |
| **ITAR** | ITAR | US-person access only, no foreign hosting |

### Cross-Regulation Encryption Requirements

| Regulation | Encryption at Rest | Encryption in Transit | Algorithm Standard |
|---|---|---|---|
| GDPR | "Appropriate technical measures" (Art. 32) | "Appropriate technical measures" | Industry standard (no specific algorithm mandated) |
| HIPAA | "Reasonable and appropriate" (addressable) | Required (164.312(e)) | NIST-approved (AES, 3DES deprecated) |
| PCI-DSS | Required for PAN (Req. 3.5.1) | Required for CHD over public networks (Req. 4.2.1) | Strong cryptography (AES-256 recommended) |
| SOC 2 | Trust Services Criteria dependent | Trust Services Criteria dependent | Industry standard |
| FedRAMP | Required (SC-28) | Required (SC-8) | FIPS 140-2/3 validated modules mandatory |

### Cross-Regulation Logging Requirements

| Regulation | What to Log | Retention | Tamper Protection |
|---|---|---|---|
| GDPR | Processing activities, consent, data subject requests | Per retention policy | Not explicitly required (accountability principle) |
| HIPAA | All PHI access (164.312(b)) | 6 years (policies); organizational policy for logs | Recommended |
| PCI-DSS | All CHD access, admin actions, security events (Req. 10) | 12 months (3 months online) | Required (Req. 10.3.2) |
| SOC 2 | Security events, access changes, anomalies (CC7) | Per organizational policy (typically 1 year+) | Expected |
| FedRAMP | AU-2 event list (access, changes, failures) | Per organizational policy (typically 1-3 years) | Required (AU-9) |

### What to Look For

```regex
# Data classification annotations in code
(?i)(@pii|@phi|@pci|@confidential|@sensitive|@restricted|@internal|@public)
(?i)(data.?class|classification)\s*[:=]\s*["']?(pii|phi|pci|confidential|sensitive|restricted|internal|public)

# Consent tracking
(?i)(consent.?record|consent.?log|consent.?timestamp|consent.?version|consent.?audit)

# Data retention enforcement
(?i)(retention.?policy|retention.?period|data.?ttl|purge.?schedule|cleanup.?job|auto.?delete|expire.?data)

# Data subject request handling
(?i)(dsar|data.?subject.?request|privacy.?request|erasure.?request|access.?request|portability.?request)
```

---

## Review Procedure Summary

When evaluating regulatory compliance in code:

1. **Determine applicable regulations**: Based on the project's industry, data
   types, user base, and deployment geography, identify which regulations apply.
   Not all regulations apply to every project. GDPR applies if EU users exist;
   HIPAA applies if PHI is handled; PCI-DSS applies if cardholder data is
   processed; FedRAMP applies if serving US federal agencies.

2. **Identify sensitive data types**: Catalog what personal data, health data,
   financial data, or classified data the application handles. Map fields and
   data flows to regulation-specific requirements.

3. **Evaluate data collection practices**: Check consent mechanisms, data
   minimization, lawful basis documentation, and privacy notices (GDPR Section 1,
   CCPA Section 7).

4. **Verify data subject rights**: Confirm that access, portability, erasure,
   rectification, and opt-out mechanisms are implemented and functional (GDPR
   Section 1.2, CCPA Section 7).

5. **Check encryption**: Verify encryption at rest and in transit for all
   regulated data types. Confirm algorithm strength and key management practices
   (HIPAA Section 2.2, PCI-DSS Section 3.2, FedRAMP Section 5.4).

6. **Review access controls**: Confirm RBAC, least privilege, MFA, session
   management, and account lifecycle management (HIPAA Section 2.3, PCI-DSS
   Section 3.3, FedRAMP Section 5.1).

7. **Audit logging completeness**: Verify that all regulated events are logged,
   logs are tamper-evident, retention meets requirements, and sensitive data is
   excluded from logs (HIPAA Section 2.5, PCI-DSS Section 3.4, FedRAMP
   Section 5.2).

8. **Evaluate cross-border transfers**: For GDPR, confirm legal transfer
   mechanisms are in place and documented. For ITAR, confirm US-only hosting
   and access (GDPR Section 1.5, Export Controls Section 6).

9. **Check export controls**: Identify cryptographic capabilities and verify
   export classification compliance (Section 6).

10. **Classify each finding** using the severity tables in each section.
    Prioritize CRITICAL and HIGH findings. Map findings to specific regulation
    articles and requirements for actionable remediation.
