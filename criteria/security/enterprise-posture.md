# Enterprise Security Posture Reference Checklist

Canonical reference for evaluating enterprise security posture across applications
and infrastructure. Covers trust boundaries, access control, audit logging, network
security, data protection, configuration hardening, and compliance touchpoints.
Complements the inlined criteria in `review-security.md`.

---

## 1. Input Validation and Trust Boundaries

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 1.1 | All external input is validated on the server side (never rely solely on client-side validation) | **HIGH** |
| 1.2 | Input validation uses allowlists (not denylists) wherever possible | **HIGH** |
| 1.3 | Numeric inputs are validated for range, type, and signedness | **MEDIUM** |
| 1.4 | String inputs are validated for length, character set, and format | **MEDIUM** |
| 1.5 | File uploads are validated for type, size, and content (not just extension) | **HIGH** |
| 1.6 | Structured data (JSON, XML, YAML) is validated against a schema before processing | **MEDIUM** |
| 1.7 | Trust boundaries are explicitly identified and enforced (client/server, service-to-service, internal/external) | **HIGH** |
| 1.8 | Data crossing trust boundaries is re-validated, even from internal services | **MEDIUM** |
| 1.9 | Content-Type headers are validated and enforced on both request and response | **MEDIUM** |
| 1.10 | Deserialization of untrusted data uses safe methods only (no `pickle`, no `eval`, use `yaml.safe_load`) | **CRITICAL** |

### What to Look For

```regex
# Missing server-side validation indicators
(?i)(req|request)\.(body|query|params|args)\.\w+  # Direct use without validation

# Schema validation libraries (positive indicators)
(?i)(joi|yup|zod|cerberus|marshmallow|pydantic|json.?schema|ajv)

# Allowlist patterns (positive indicators)
(?i)(allowed_values|whitelist|allowlist|valid_types|permitted_params)

# Denylist patterns (weaker - may need allowlist instead)
(?i)(blacklist|blocklist|denylist|forbidden|banned)
```

---

## 2. Authorization and Access Control

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 2.1 | Every API endpoint and page has an explicit authorization check | **CRITICAL** |
| 2.2 | Authorization is enforced server-side, not via client-side UI hiding | **CRITICAL** |
| 2.3 | Role-Based Access Control (RBAC) roles follow least-privilege principle | **HIGH** |
| 2.4 | Attribute-Based Access Control (ABAC) policies are centrally defined and auditable | **HIGH** |
| 2.5 | IDOR protections are in place: resource access verifies ownership or permission | **CRITICAL** |
| 2.6 | Horizontal privilege escalation is prevented (user A cannot access user B's resources) | **CRITICAL** |
| 2.7 | Vertical privilege escalation is prevented (regular user cannot perform admin actions) | **CRITICAL** |
| 2.8 | API keys and service accounts follow least-privilege (scoped to minimum necessary permissions) | **HIGH** |
| 2.9 | Authorization decisions are logged for audit purposes | **MEDIUM** |
| 2.10 | Default-deny policy: access is denied unless explicitly granted | **HIGH** |
| 2.11 | Multi-tenancy isolation is enforced at the data layer (queries are tenant-scoped) | **CRITICAL** |
| 2.12 | Admin interfaces are restricted by network, IP, or additional authentication factor | **HIGH** |

### What to Look For

```regex
# Missing auth middleware
(?i)(app\.(get|post|put|delete|patch)|router\.(get|post|put|delete|patch))\s*\(\s*["'/]
# Verify: auth middleware is present in the route chain

# IDOR patterns
(?i)(findById|get_object_or_404|findOne|findByPk)\s*\(\s*(req|request|params)
# Verify: ownership check follows the lookup

# Multi-tenancy
(?i)(tenant_id|org_id|organization_id|account_id)
# Verify: all database queries include tenant scoping

# Role checks (positive indicators)
(?i)(hasRole|hasPermission|isAuthorized|@PreAuthorize|@Secured|@login_required|@permission_required|requireAuth|requireRole)
```

---

## 3. Security Logging and Audit Trail

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 3.1 | Authentication events are logged: login success, login failure, logout, password change, MFA events | **HIGH** |
| 3.2 | Authorization failures are logged with user identity, resource, and action | **HIGH** |
| 3.3 | Administrative actions are logged: user creation, role changes, config changes, data deletion | **HIGH** |
| 3.4 | Data access to sensitive records is logged (read audit trail for PII, financial data) | **MEDIUM** |
| 3.5 | Log entries include: timestamp (UTC), user/session ID, source IP, action, resource, outcome | **MEDIUM** |
| 3.6 | Sensitive data is excluded from logs: passwords, tokens, PII, credit card numbers | **HIGH** |
| 3.7 | Logs are protected against injection (newline stripping, structured logging) | **MEDIUM** |
| 3.8 | Logs are forwarded to a centralized, tamper-evident logging system | **MEDIUM** |
| 3.9 | Log retention meets compliance requirements (minimum 90 days hot, 1 year cold for SOC 2) | **MEDIUM** |
| 3.10 | Alerting is configured for: brute force attempts, privilege escalation, unusual access patterns | **HIGH** |
| 3.11 | Audit logs are immutable (append-only) and protected from deletion by application users | **HIGH** |
| 3.12 | Clock synchronization (NTP) is configured for consistent timestamps across services | **LOW** |

### What to Look For

```regex
# Logging framework usage (positive indicators)
(?i)(logger|logging|log4j|winston|bunyan|pino|zerolog|zap|slog|serilog|structlog)

# Security event logging (verify these exist)
(?i)(login|logout|auth|failed|denied|forbidden|unauthorized|privilege|escalat|admin).*\b(log|audit|event|track)\b

# Sensitive data in logs (negative pattern)
(?i)(log|logger|console)\.\w+\(.*\b(password|token|secret|ssn|credit.?card|cvv|authorization)\b

# Structured logging (positive indicator)
(?i)(structlog|json.?log|structured|log\.WithField|log\.With\()
```

---

## 4. HTTP Security

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 4.1 | `Strict-Transport-Security` header is set with `max-age >= 31536000` and `includeSubDomains` | **HIGH** |
| 4.2 | `Content-Security-Policy` header is set with restrictive policy (no `unsafe-inline`, no `unsafe-eval` in production) | **HIGH** |
| 4.3 | `X-Content-Type-Options: nosniff` header is set | **MEDIUM** |
| 4.4 | `X-Frame-Options: DENY` or `SAMEORIGIN` is set (or equivalent CSP `frame-ancestors`) | **MEDIUM** |
| 4.5 | `Referrer-Policy` is set to `strict-origin-when-cross-origin` or stricter | **LOW** |
| 4.6 | `Permissions-Policy` restricts unnecessary browser features (camera, microphone, geolocation) | **LOW** |
| 4.7 | CORS is configured with explicit allowed origins (not wildcard `*` for credentialed requests) | **HIGH** |
| 4.8 | CORS `Access-Control-Allow-Credentials` is not combined with `Access-Control-Allow-Origin: *` | **CRITICAL** |
| 4.9 | CSRF protection is enabled for all state-changing operations (token-based or SameSite cookies) | **HIGH** |
| 4.10 | Rate limiting is implemented on authentication endpoints, API endpoints, and form submissions | **HIGH** |
| 4.11 | Rate limiting returns `429 Too Many Requests` with `Retry-After` header | **LOW** |
| 4.12 | HTTP methods are restricted to those actually needed per endpoint (no blanket `app.all()`) | **MEDIUM** |
| 4.13 | Response bodies do not leak server version, framework version, or stack traces | **MEDIUM** |
| 4.14 | `Set-Cookie` flags include `Secure`, `HttpOnly`, and `SameSite=Lax` (or `Strict`) | **HIGH** |
| 4.15 | API versioning is implemented to enable deprecation of insecure endpoints | **LOW** |

### What to Look For

```regex
# Security headers (verify these are set)
(?i)(Strict-Transport-Security|Content-Security-Policy|X-Content-Type-Options|X-Frame-Options|Referrer-Policy|Permissions-Policy)

# CORS misconfiguration
(?i)Access-Control-Allow-Origin\s*[:=]\s*["']?\*
(?i)(cors|CORS)\s*\(\s*\)   # Default permissive CORS

# CSRF protection (positive indicators)
(?i)(csrf|xsrf|csrftoken|anti.?forgery|csurf|CSRFProtect)

# Rate limiting (positive indicators)
(?i)(rate.?limit|throttle|RateLimit|express.?rate|slowapi|throttling|bucket)

# Cookie security
(?i)(httponly|secure|samesite)\s*[:=]\s*(false|0|none)
```

---

## 5. Network and Communication Security

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 5.1 | All external communication uses TLS 1.2 or higher | **CRITICAL** |
| 5.2 | TLS certificates are valid, not self-signed in production, and from a trusted CA | **HIGH** |
| 5.3 | TLS certificate verification is not disabled (`verify=False`, `InsecureSkipVerify`, `rejectUnauthorized: false`) | **CRITICAL** |
| 5.4 | Weak cipher suites are disabled (RC4, DES, 3DES, export ciphers, NULL ciphers) | **HIGH** |
| 5.5 | Service-to-service communication uses mTLS or equivalent authentication | **HIGH** |
| 5.6 | Internal APIs are not exposed to the public internet without authentication | **CRITICAL** |
| 5.7 | DNS resolution uses DNSSEC or DNS-over-HTTPS where available | **LOW** |
| 5.8 | WebSocket connections use `wss://` (TLS) and validate the Origin header | **HIGH** |
| 5.9 | gRPC services use TLS channel credentials, not insecure channels in production | **HIGH** |
| 5.10 | Network segmentation isolates sensitive services (database, cache, message queue) from public-facing services | **HIGH** |
| 5.11 | Egress filtering restricts outbound connections to known destinations | **MEDIUM** |
| 5.12 | Certificate pinning is implemented for high-security mobile or IoT clients | **MEDIUM** |
| 5.13 | Protocol downgrade attacks are prevented (HSTS preload, no fallback to HTTP) | **HIGH** |

### What to Look For

```regex
# TLS verification disabled
(?i)(verify\s*=\s*False|InsecureSkipVerify\s*[:=]\s*true|rejectUnauthorized\s*[:=]\s*false|NODE_TLS_REJECT_UNAUTHORIZED\s*=\s*["']?0)

# Insecure protocols
(?i)(http://|ftp://|telnet://|ws://)[^"'\s]*\.(com|org|net|io|dev)
# Exclude: localhost, 127.0.0.1, test fixtures

# Weak TLS versions
(?i)(SSLv2|SSLv3|TLSv1\.0|TLSv1\.1|ssl\.PROTOCOL_TLSv1\b)

# Insecure gRPC
(?i)(grpc\.insecure_channel|ManagedChannelBuilder\.forAddress.*usePlaintext)

# mTLS indicators (positive)
(?i)(mutual.?tls|mtls|client.?cert|client_certificate|tls\.RequireAndVerifyClientCert)
```

---

## 6. Data Protection

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 6.1 | Data at rest is encrypted (database encryption, disk encryption, or field-level encryption for sensitive fields) | **HIGH** |
| 6.2 | Encryption keys are stored in a key management service (KMS), not in application code or config | **CRITICAL** |
| 6.3 | PII is identified and classified according to a data classification policy | **HIGH** |
| 6.4 | PII is encrypted or tokenized at the field level in databases | **HIGH** |
| 6.5 | PII is masked or redacted in logs, error messages, and non-production environments | **HIGH** |
| 6.6 | Data retention policies are implemented and enforced (automated deletion of expired data) | **MEDIUM** |
| 6.7 | Backups are encrypted and access-controlled | **HIGH** |
| 6.8 | Data export/download endpoints enforce authorization and audit logging | **HIGH** |
| 6.9 | Sensitive data is not stored in browser local storage, session storage, or cookies (use server-side sessions) | **HIGH** |
| 6.10 | Database queries returning sensitive data use column-level selection (no `SELECT *` on tables with PII) | **MEDIUM** |
| 6.11 | File storage (S3, GCS, Azure Blob) uses server-side encryption and private ACLs | **HIGH** |
| 6.12 | Temporary files containing sensitive data are securely deleted after use | **MEDIUM** |
| 6.13 | Data in transit between services is encrypted (see Section 5) | **HIGH** |
| 6.14 | Cryptographic key rotation is automated on a defined schedule | **MEDIUM** |

### What to Look For

```regex
# PII field names
(?i)(ssn|social_security|date_of_birth|dob|national_id|passport|driver.?license|credit.?card|card_number|pan|cvv|phone|email|address|salary|medical|health|diagnosis)

# Unencrypted sensitive storage
(?i)(localStorage|sessionStorage)\.(setItem|getItem)\(.*\b(token|password|ssn|secret)\b

# SELECT * on potentially sensitive tables
(?i)SELECT\s+\*\s+FROM\s+(users|customers|patients|employees|accounts|payments|orders)

# Cloud storage ACLs
(?i)(public-read|public-read-write|AllUsers|allAuthenticatedUsers)
(?i)(BlockPublicAccess|block_public_access)\s*[:=]\s*(false|False)

# KMS / key management (positive indicators)
(?i)(kms|key.?management|vault|aws.?kms|cloud.?kms|azure.?key.?vault|HashiCorp.?Vault)
```

---

## 7. Configuration Security

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 7.1 | Debug mode is disabled in production (`DEBUG=False`, no verbose error pages) | **HIGH** |
| 7.2 | Default credentials are changed or removed before deployment | **CRITICAL** |
| 7.3 | Admin panels and management interfaces are not exposed on public ports | **HIGH** |
| 7.4 | Error responses do not reveal internal paths, stack traces, or technology details | **MEDIUM** |
| 7.5 | Server/framework version headers are suppressed (`Server`, `X-Powered-By`) | **LOW** |
| 7.6 | Directory listing is disabled on web servers | **MEDIUM** |
| 7.7 | Unnecessary HTTP methods are disabled (TRACE, OPTIONS if not needed for CORS) | **LOW** |
| 7.8 | Configuration is externalized from code (environment variables, config service, sealed secrets) | **HIGH** |
| 7.9 | Configuration files with secrets are not committed to version control | **CRITICAL** |
| 7.10 | Infrastructure as Code (Terraform, CloudFormation, Ansible) does not contain hardcoded secrets | **CRITICAL** |
| 7.11 | Container images use minimal base images (distroless, Alpine) and run as non-root | **MEDIUM** |
| 7.12 | Container images are scanned for vulnerabilities before deployment | **HIGH** |
| 7.13 | Kubernetes pods do not run as privileged or with host networking | **HIGH** |
| 7.14 | Kubernetes secrets are encrypted at rest in etcd (EncryptionConfiguration) | **HIGH** |
| 7.15 | Feature flags and kill switches exist for security-sensitive features | **MEDIUM** |

### What to Look For

```regex
# Debug mode
(?i)(DEBUG|debug)\s*[:=]\s*(True|true|1|"true"|'true')

# Default credentials
(?i)(admin|root|default|test)[:@](admin|root|password|123456|default|changeme)

# Version exposure
(?i)(X-Powered-By|Server)\s*[:=]
(?i)app\.disable\s*\(\s*["']x-powered-by["']\s*\)  # Positive: disabled

# Container security
(?i)(privileged\s*[:=]\s*true|runAsRoot|hostNetwork\s*[:=]\s*true|hostPID\s*[:=]\s*true)
(?i)(USER\s+root|--privileged)

# Non-root container (positive indicator)
(?i)(USER\s+\d{4,}|USER\s+(?!root)\w+|runAsNonRoot\s*[:=]\s*true|readOnlyRootFilesystem\s*[:=]\s*true)
```

---

## 8. Compliance Considerations

This section provides touchpoints for common compliance frameworks. A full compliance
audit requires dedicated assessment; these checkpoints identify code-level indicators
relevant to each framework.

### SOC 2 (Type II)

| # | Touchpoint | Relevant Sections |
|---|---|---|
| 8.1 | Access control policies are implemented and enforced | Sections 2, 4 |
| 8.2 | Security events are logged and monitored | Section 3 |
| 8.3 | Data encryption in transit and at rest | Sections 5, 6 |
| 8.4 | Change management: code changes go through review and approval | CI/CD pipeline review |
| 8.5 | Incident response: alerting and escalation paths are defined | Section 3.10 |
| 8.6 | Vendor management: third-party dependencies are tracked and assessed | OWASP A06 |
| 8.7 | Availability: health checks, circuit breakers, and failover mechanisms exist | Architecture review |

### PCI-DSS v4.0

| # | Touchpoint | Relevant Sections |
|---|---|---|
| 8.8 | Cardholder data is identified, classified, and minimized (Req 3) | Section 6 |
| 8.9 | PAN is encrypted or tokenized at rest and in transit (Req 3, 4) | Sections 5, 6 |
| 8.10 | Access to cardholder data is restricted by business need-to-know (Req 7) | Section 2 |
| 8.11 | Unique IDs are assigned to each person with access (Req 8) | Section 2 |
| 8.12 | All access to cardholder data is logged (Req 10) | Section 3 |
| 8.13 | Vulnerability scans and penetration tests are scheduled (Req 11) | OWASP A06 |
| 8.14 | Secure development lifecycle practices are followed (Req 6) | All sections |
| 8.15 | WAF or equivalent protection is deployed for public-facing applications (Req 6.4) | Section 4 |

### HIPAA (Technical Safeguards)

| # | Touchpoint | Relevant Sections |
|---|---|---|
| 8.16 | Access control: unique user identification, emergency access procedure (164.312(a)) | Section 2 |
| 8.17 | Audit controls: record and examine activity in systems with ePHI (164.312(b)) | Section 3 |
| 8.18 | Integrity: mechanisms to authenticate ePHI and detect tampering (164.312(c)) | Section 6 |
| 8.19 | Transmission security: encryption of ePHI in transit (164.312(e)) | Section 5 |
| 8.20 | PHI is encrypted at rest using AES-256 or equivalent | Section 6 |
| 8.21 | Minimum necessary: access to PHI is limited to what is needed for the task | Section 2 |
| 8.22 | BAA (Business Associate Agreement) compliance for third-party services handling PHI | Vendor review |
| 8.23 | Breach notification mechanisms are in place | Section 3.10 |

### FedRAMP (Moderate Baseline)

| # | Touchpoint | Relevant Sections |
|---|---|---|
| 8.24 | FIPS 140-2/140-3 validated cryptographic modules are used (SC-13) | Sections 5, 6 |
| 8.25 | Multi-factor authentication is required for privileged access (IA-2) | Section 2.12 |
| 8.26 | Continuous monitoring: automated vulnerability scanning and reporting (CA-7) | OWASP A06 |
| 8.27 | Audit logging meets NIST 800-53 AU controls: AU-2, AU-3, AU-6, AU-8, AU-9, AU-12 | Section 3 |
| 8.28 | Boundary protection: network segmentation and access control (SC-7) | Section 5.10 |
| 8.29 | Session management: session lock, session termination (AC-11, AC-12) | Section 4.14 |
| 8.30 | Least privilege enforced for all accounts and services (AC-6) | Section 2 |
| 8.31 | Configuration management: baseline configurations are documented and enforced (CM-2, CM-6) | Section 7 |
| 8.32 | System and information integrity: malicious code protection, software updates (SI-2, SI-3) | OWASP A06 |

### Cross-Framework Code Indicators

```regex
# FIPS mode indicators
(?i)(FIPS|fips.?mode|fips.?140|FIPS_mode_set)

# Data classification tags
(?i)(classification|data.?class|sensitivity|confidential|internal|public|restricted|pii|phi|pci)

# Compliance annotations
(?i)(@compliant|@pci|@hipaa|@sox|@gdpr|compliance.?check|audit.?required)

# Data residency / sovereignty
(?i)(data.?residency|data.?sovereignty|region.?lock|geo.?restrict)
```

---

## Review Procedure Summary

When evaluating enterprise security posture:

1. **Map trust boundaries**: identify where external input enters, where services
   communicate, and where data crosses security domains.
2. **Verify authorization coverage**: every endpoint, every data access path,
   every administrative function must have explicit access control.
3. **Audit the logging pipeline**: confirm security events are captured, sensitive
   data is excluded, and logs are forwarded and retained.
4. **Inspect HTTP hardening**: check security headers, CORS, CSRF, cookie flags,
   and rate limiting.
5. **Validate communication security**: TLS configuration, certificate handling,
   service-to-service authentication.
6. **Assess data protection**: encryption at rest and in transit, PII handling,
   key management, storage ACLs.
7. **Review configuration hygiene**: debug mode, default credentials, version
   exposure, container security, infrastructure as code.
8. **Check compliance touchpoints**: map findings to relevant framework requirements
   based on the organization's compliance obligations.
9. **Classify each finding** using the severity levels in each section.
10. **Prioritize remediation**: CRITICAL and HIGH findings first, with emphasis
    on items that appear across multiple compliance frameworks.
