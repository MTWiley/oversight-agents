# Audit Readiness Reference Checklist

Canonical reference for evaluating a project's readiness for compliance audits.
Covers dependency management, change documentation, security documentation,
compliance automation, evidence collection, and third-party risk management.
This checklist ensures that auditors can find the evidence they need and that
compliance controls are demonstrably operational.

Complements the inlined criteria in `review-compliance.md`. The
`compliance-reviewer` agent uses this file as a lookup when evaluating audit
readiness posture.

---

## 1. Dependency Management

Dependency management is a foundational audit control. Auditors verify that the
organization knows what software it uses, where it comes from, whether it is
vulnerable, and whether builds are reproducible. Gaps here affect every
compliance framework: SOC 2 (CC8.1 change management, CC7.1 vulnerability
management), PCI-DSS (Req. 6.3 security vulnerabilities), FedRAMP (SI-2 flaw
remediation, CM-2 baseline configuration), and HIPAA (164.308(a)(5)(ii)(B)
protection from malicious software).

### 1.1 Reproducible Builds

| # | Checkpoint | Severity |
|---|---|---|
| 1.1.1 | Lock files are committed to version control: `package-lock.json`, `yarn.lock`, `pnpm-lock.yaml`, `poetry.lock`, `Pipfile.lock`, `go.sum`, `Gemfile.lock`, `Cargo.lock`, `composer.lock`, `mix.lock` | **HIGH** |
| 1.1.2 | Lock files are not in `.gitignore` | **HIGH** |
| 1.1.3 | CI/CD uses deterministic install commands: `npm ci` (not `npm install`), `pip install --require-hashes`, `bundle install --frozen`, `cargo build --locked` | **MEDIUM** |
| 1.1.4 | Build tool versions are pinned: `.tool-versions`, `.node-version`, `.python-version`, `.go-version`, `rust-toolchain.toml` | **MEDIUM** |
| 1.1.5 | Container base images are pinned to a specific version or digest, not `:latest` | **MEDIUM** |
| 1.1.6 | The same commit produces the same artifact regardless of build time or environment | **MEDIUM** |
| 1.1.7 | Build scripts do not fetch external resources at build time without version pinning (no `curl | bash` without checksum verification) | **MEDIUM** |
| 1.1.8 | Monorepo workspace dependencies use exact versions or workspace references, not floating ranges | **LOW** |

#### What to Look For

```regex
# Lock files in .gitignore (negative pattern)
(?i)(package-lock|yarn\.lock|pnpm-lock|poetry\.lock|Pipfile\.lock|go\.sum|Gemfile\.lock|Cargo\.lock|composer\.lock)

# Deterministic install commands (positive indicators)
(?i)(npm ci|pip install.*--require-hashes|bundle install.*--frozen|cargo build.*--locked|pnpm install.*--frozen-lockfile)

# Non-deterministic install (negative patterns)
(?i)(npm install\b(?!.*--ci)|pip install\b(?!.*--require-hashes)(?!.*-r requirements))
# Note: npm install (without ci) resolves versions; pip install without constraints is non-deterministic

# Mutable image tags (negative pattern)
(?i)FROM\s+\S+:(latest|main|master|dev|nightly)\b

# curl pipe to shell (negative pattern)
(?i)curl\s+.*\|\s*(bash|sh|zsh)
```

### 1.2 Trusted Sources

| # | Checkpoint | Severity |
|---|---|---|
| 1.2.1 | Dependencies are installed from official registries (npmjs.com, PyPI, crates.io, Maven Central, NuGet, RubyGems) | **HIGH** |
| 1.2.2 | Dependencies from Git URLs, private registries, or local paths are documented with justification | **MEDIUM** |
| 1.2.3 | Private registries use authentication and HTTPS | **HIGH** |
| 1.2.4 | Dependency integrity is verified: checksums or signatures are validated during install (npm uses `integrity` field in lockfile; pip supports `--require-hashes`; Go uses `go.sum`) | **MEDIUM** |
| 1.2.5 | No dependencies are fetched from personal GitHub accounts, Gists, or unverified sources without review | **HIGH** |
| 1.2.6 | Dependency sources are consistent across environments (dev, CI, production all use the same registry) | **MEDIUM** |
| 1.2.7 | Registry mirrors or proxies (Artifactory, Nexus, Verdaccio) are configured to proxy only approved upstream registries | **MEDIUM** |
| 1.2.8 | Typosquatting risk is mitigated: dependency names are verified against the intended package | **MEDIUM** |

#### What to Look For

```regex
# Git URL dependencies (review required)
(?i)"[^"]+"\s*:\s*"(git\+|git://|https?://github\.com|https?://gitlab\.com)[^"]*"
(?i)git\s*=\s*"https?://

# Private registry configuration
(?i)(registry|repository)\s*[:=]\s*["']?https?://(?!registry\.(npmjs|yarnpkg)\.com|pypi\.org|crates\.io|repo1\.maven|rubygems\.org|nuget\.org)

# Local path dependencies (may be intentional in monorepos)
(?i)(file:|path\s*[:=]\s*["']?\.\./)

# Integrity/hash verification
(?i)(integrity|hash|checksum|sha256|sha512)\s*[:=]
```

### 1.3 Vulnerability Scanning

| # | Checkpoint | Severity |
|---|---|---|
| 1.3.1 | Automated dependency vulnerability scanning is configured (Dependabot, Snyk, Renovate, Trivy, Grype, `npm audit`, `pip-audit`, `cargo audit`, `bundler-audit`) | **HIGH** |
| 1.3.2 | Vulnerability scanning runs on every PR or merge request, not just periodic manual scans | **HIGH** |
| 1.3.3 | Critical and high-severity vulnerabilities in direct dependencies block merges or deployments | **HIGH** |
| 1.3.4 | Transitive dependency vulnerabilities are scanned (not just direct dependencies) | **MEDIUM** |
| 1.3.5 | Vulnerability scan results are stored as artifacts or reported to a centralized dashboard for audit trail | **MEDIUM** |
| 1.3.6 | A defined SLA exists for vulnerability remediation: CRITICAL within 24-72 hours, HIGH within 1-2 weeks, MEDIUM within 30 days | **MEDIUM** |
| 1.3.7 | Known-risk exceptions (vulnerabilities accepted as non-exploitable in context) are documented with justification and expiration | **MEDIUM** |
| 1.3.8 | Container image vulnerability scanning is integrated into the CI/CD pipeline | **HIGH** |
| 1.3.9 | Vulnerability scanning covers all ecosystems used in the project (e.g., both npm and pip if the project uses JavaScript and Python) | **MEDIUM** |
| 1.3.10 | Automated dependency update PRs are enabled (Dependabot, Renovate) to reduce the window of exposure | **MEDIUM** |

#### What to Look For

```regex
# Vulnerability scanning tools (positive indicators)
(?i)(dependabot|snyk|renovate|trivy|grype|npm.?audit|pip.?audit|cargo.?audit|bundler.?audit|safety|ossaudit)

# Dependabot configuration
(?i)\.github/dependabot\.(yml|yaml)

# Renovate configuration
(?i)(renovate\.(json|json5)|\.renovaterc)

# Snyk configuration
(?i)(\.snyk|snyk\.(json|yml|yaml))

# Vulnerability exception/ignore files
(?i)(\.snyk|\.trivyignore|\.grype\.yaml|audit-ci\.json)
```

### 1.4 SBOM (Software Bill of Materials) Generation

| # | Checkpoint | Severity |
|---|---|---|
| 1.4.1 | An SBOM is generated as part of the build process | **MEDIUM** |
| 1.4.2 | The SBOM uses a standard format: SPDX (ISO/IEC 5962:2021) or CycloneDX (OASIS standard) | **LOW** |
| 1.4.3 | The SBOM includes all direct and transitive dependencies with version numbers and license information | **MEDIUM** |
| 1.4.4 | The SBOM is stored as a build artifact and associated with the corresponding release | **LOW** |
| 1.4.5 | The SBOM is updated on every release (not just generated once and forgotten) | **MEDIUM** |
| 1.4.6 | For container images, the SBOM includes both OS packages and application dependencies | **LOW** |
| 1.4.7 | The SBOM generation tool is integrated into CI/CD, not run manually | **LOW** |

**Context**: SBOM generation is increasingly a compliance requirement. US Executive
Order 14028 (2021) requires SBOMs for software sold to the federal government.
The EU Cyber Resilience Act (CRA) will require SBOMs for products sold in the EU.
NTIA minimum elements for SBOMs include: supplier name, component name, version,
dependency relationship, unique identifier, and timestamp.

**SBOM Generation Tools**:

| Tool | Format Support | Ecosystems | Integration |
|---|---|---|---|
| `syft` (Anchore) | SPDX, CycloneDX | All major + container images | CLI, GitHub Actions |
| `cdxgen` | CycloneDX | All major | CLI, GitHub Actions |
| `trivy` | SPDX, CycloneDX | All major + container images | CLI, GitHub Actions |
| `spdx-sbom-generator` | SPDX | Go, Java, Node, Python, Ruby, Rust | CLI |
| `cyclonedx-cli` | CycloneDX | Cross-platform converter | CLI |

#### What to Look For

```regex
# SBOM generation (positive indicators)
(?i)(sbom|software.?bill.?of.?materials|spdx|cyclonedx|syft|cdxgen)

# SBOM in CI
(?i)(generate.*sbom|sbom.*generat|sbom.*output|sbom.*artifact)

# SPDX/CycloneDX file patterns
(?i)\.(spdx|cdx)\.(json|xml|yaml|yml|rdf)$
(?i)(bom\.(json|xml)|sbom\.(json|xml|spdx))
```

---

## 2. Change Documentation

Change documentation provides the audit trail that demonstrates controlled,
reviewed, and authorized changes to the system. This is central to SOC 2 (CC8.1
changes are authorized, tested, and approved), PCI-DSS (Req. 6.5.1 change
control processes), FedRAMP (CM-3 configuration change control), and ISO 27001
(A.14.2 change management).

### 2.1 Changelog Quality

| # | Checkpoint | Severity |
|---|---|---|
| 2.1.1 | A CHANGELOG.md or equivalent exists and is maintained | **MEDIUM** |
| 2.1.2 | The changelog follows a consistent format: [Keep a Changelog](https://keepachangelog.com) or equivalent | **LOW** |
| 2.1.3 | Each release entry documents: version number, date, and categorized changes (Added, Changed, Deprecated, Removed, Fixed, Security) | **MEDIUM** |
| 2.1.4 | Security-relevant changes are explicitly called out in the changelog under a "Security" category | **MEDIUM** |
| 2.1.5 | The changelog links to relevant issues, PRs, or tickets for each change | **LOW** |
| 2.1.6 | Breaking changes are clearly marked and include migration guidance | **MEDIUM** |
| 2.1.7 | The changelog is updated as part of the release process (automated or enforced by CI check) | **LOW** |
| 2.1.8 | Version numbers follow Semantic Versioning (semver) or a documented versioning scheme | **LOW** |

#### What to Look For

```regex
# Changelog files
(?i)^CHANGELOG(\.md|\.txt|\.rst)?$
(?i)^CHANGES(\.md|\.txt|\.rst)?$
(?i)^HISTORY(\.md|\.txt|\.rst)?$
(?i)^RELEASE.?NOTES(\.md|\.txt|\.rst)?$

# Keep a Changelog format indicators
(?i)## \[\d+\.\d+\.\d+\].*\d{4}-\d{2}-\d{2}
(?i)### (Added|Changed|Deprecated|Removed|Fixed|Security)

# Conventional Commits (supports automated changelog generation)
(?i)^(feat|fix|docs|style|refactor|perf|test|build|ci|chore|revert)(\(.+\))?!?:
```

### 2.2 Traceability to Requirements

| # | Checkpoint | Severity |
|---|---|---|
| 2.2.1 | Commits reference issue/ticket numbers (e.g., `PROJ-123`, `#456`, `closes #789`) | **MEDIUM** |
| 2.2.2 | Pull requests link to the issue or requirement they implement | **MEDIUM** |
| 2.2.3 | Commit messages are descriptive (not just "fix", "update", "changes") | **LOW** |
| 2.2.4 | Squash commits preserve the original ticket references in the squash message | **LOW** |
| 2.2.5 | Architecture Decision Records (ADRs) exist for significant design decisions | **MEDIUM** |
| 2.2.6 | Security-relevant changes reference the security requirement or threat model entry they address | **MEDIUM** |
| 2.2.7 | Regulatory-relevant changes reference the specific regulation or control they implement or satisfy | **MEDIUM** |
| 2.2.8 | A requirements traceability matrix (RTM) exists for regulated systems, mapping requirements to implementation and tests | **HIGH** (for FedRAMP, HIPAA, PCI-DSS) |

#### What to Look For

```regex
# Issue/ticket references in commits
(?i)([A-Z]+-\d+|#\d+|closes?\s+#\d+|fixes?\s+#\d+|resolves?\s+#\d+)

# Descriptive commit messages (positive indicators)
(?i)^(feat|fix|docs|refactor|test|perf|ci|build|chore)\s*(\(.+\))?\s*:\s*.{10,}

# Poor commit messages (negative indicators)
(?i)^(fix|update|changes|wip|temp|test|stuff|misc|oops|typo)\s*$

# ADR files
(?i)(adr|decision)[-_]?\d+

# Requirements traceability
(?i)(traceability|requirements?.?matrix|RTM|requirement.?mapping)
```

### 2.3 Code Review Records

| # | Checkpoint | Severity |
|---|---|---|
| 2.3.1 | All code changes go through pull request / merge request review before merging to the main branch | **HIGH** |
| 2.3.2 | Branch protection rules require at least one approving review | **HIGH** |
| 2.3.3 | Branch protection rules prevent self-approval (the PR author cannot be the sole approver) | **HIGH** |
| 2.3.4 | CODEOWNERS file exists to route reviews to appropriate team members | **MEDIUM** |
| 2.3.5 | Review comments are substantive (not just "LGTM") for security-sensitive or compliance-relevant changes | **MEDIUM** |
| 2.3.6 | Code review approval records are preserved and accessible to auditors (PR history in GitHub/GitLab, not in ephemeral tools) | **HIGH** |
| 2.3.7 | Emergency changes (hotfixes) follow a documented expedited review process with retrospective review | **MEDIUM** |
| 2.3.8 | Force pushes to protected branches are disabled or restricted to break-glass scenarios with audit logging | **HIGH** |
| 2.3.9 | CI checks must pass before a PR can be merged (status checks required) | **MEDIUM** |
| 2.3.10 | Stale review approvals are dismissed on new commits (re-review required after changes) | **MEDIUM** |

#### What to Look For

```regex
# Branch protection configuration
(?i)(branch.?protection|protected.?branch|require.?review|required.?reviewers|dismiss.?stale)

# CODEOWNERS file
(?i)^\.?github/CODEOWNERS$
(?i)^CODEOWNERS$

# CI required checks
(?i)(required.?status|required.?checks|status.?checks.?required)

# Force push restrictions
(?i)(force.?push|allow.?force|restrict.?force)
```

---

## 3. Security Documentation

Security documentation demonstrates to auditors that the organization has a
defined security program. Missing security documentation is a gap across SOC 2
(CC1.4 communication), PCI-DSS (Req. 12 information security policy), FedRAMP
(PL-1 planning policy), and ISO 27001 (Clause 7.5 documented information).

### 3.1 SECURITY.md and Vulnerability Disclosure

| # | Checkpoint | Severity |
|---|---|---|
| 3.1.1 | A `SECURITY.md` file exists at the repository root or in `.github/` | **HIGH** |
| 3.1.2 | The SECURITY.md specifies how to report security vulnerabilities (email address, encrypted communication option, bug bounty platform) | **HIGH** |
| 3.1.3 | The SECURITY.md specifies the expected response timeline (e.g., acknowledgment within 48 hours, resolution target within 90 days) | **MEDIUM** |
| 3.1.4 | The SECURITY.md lists supported versions and which versions receive security updates | **MEDIUM** |
| 3.1.5 | A `security.txt` file exists at `.well-known/security.txt` for web applications (RFC 9116) | **LOW** |
| 3.1.6 | PGP/GPG key or encrypted communication channel is offered for sensitive vulnerability reports | **LOW** |
| 3.1.7 | GitHub Security Advisories or equivalent platform feature is configured for coordinated disclosure | **LOW** |
| 3.1.8 | The vulnerability disclosure process includes a commitment to not pursue legal action against good-faith reporters | **LOW** |

#### SECURITY.md Template

```markdown
# Security Policy

## Supported Versions

| Version | Supported          |
| ------- | ------------------ |
| 2.x.x   | :white_check_mark: |
| 1.x.x   | :x:                |

## Reporting a Vulnerability

Please report security vulnerabilities by emailing security@example.com.

- You will receive an acknowledgment within 48 hours.
- We will provide an initial assessment within 5 business days.
- We target resolution of confirmed vulnerabilities within 90 days.
- We do not pursue legal action against good-faith security researchers.

For sensitive reports, use our PGP key: [link to key]

## Security Update Process

Security updates are released as patch versions. Subscribe to our
security advisory feed for notifications.
```

### 3.2 Security Policy

| # | Checkpoint | Severity |
|---|---|---|
| 3.2.1 | A security policy document exists (may be internal, referenced from SECURITY.md) | **HIGH** |
| 3.2.2 | The security policy defines roles and responsibilities for security incidents | **MEDIUM** |
| 3.2.3 | The security policy defines acceptable use of cryptography (algorithms, key lengths, key management) | **MEDIUM** |
| 3.2.4 | The security policy defines access control requirements (MFA, password complexity, session timeouts) | **MEDIUM** |
| 3.2.5 | The security policy defines data classification levels and handling requirements | **HIGH** |
| 3.2.6 | The security policy is reviewed and updated at least annually | **MEDIUM** |
| 3.2.7 | The security policy references applicable regulatory requirements (HIPAA, PCI-DSS, FedRAMP, GDPR) | **MEDIUM** |

### 3.3 Incident Response References

| # | Checkpoint | Severity |
|---|---|---|
| 3.3.1 | An incident response plan exists and is referenced from the project documentation | **HIGH** |
| 3.3.2 | The incident response plan defines severity levels and escalation paths | **MEDIUM** |
| 3.3.3 | On-call rotation or incident response contact information is documented and accessible | **MEDIUM** |
| 3.3.4 | Incident response runbooks exist for common scenarios (data breach, service outage, credential compromise, dependency vulnerability) | **MEDIUM** |
| 3.3.5 | Post-incident review (blameless postmortem) process is documented and evidence of past reviews exists | **LOW** |
| 3.3.6 | Incident response plan includes regulatory notification requirements: GDPR 72-hour breach notification (Art. 33), HIPAA 60-day breach notification (164.408), PCI-DSS incident response (Req. 12.10.1) | **HIGH** |
| 3.3.7 | The incident response plan is tested at least annually (tabletop exercise, simulation, or actual incident review) | **MEDIUM** |

#### What to Look For

```regex
# Security documentation files
(?i)^SECURITY(\.md|\.txt|\.rst)?$
(?i)^\.well-known/security\.txt$
(?i)^\.github/SECURITY\.md$

# Incident response references
(?i)(incident.?response|incident.?plan|runbook|playbook|escalation|on.?call|pager.?duty|opsgenie)

# Security policy references
(?i)(security.?policy|information.?security|infosec.?policy|acceptable.?use)
```

---

## 4. Compliance Automation

Compliance automation reduces the burden of manual evidence collection and
ensures that controls are continuously enforced, not just checked during audit
windows. Auditors increasingly expect automated controls as evidence of
operational effectiveness.

### 4.1 License Checks in CI

| # | Checkpoint | Severity |
|---|---|---|
| 4.1.1 | License scanning runs as a required CI check on PRs | **HIGH** |
| 4.1.2 | A license allowlist or blocklist is defined and enforced | **MEDIUM** |
| 4.1.3 | License check failures block merges | **HIGH** |
| 4.1.4 | License exceptions are documented with justification and approval | **MEDIUM** |
| 4.1.5 | License scanning covers all package ecosystems used in the project | **MEDIUM** |
| 4.1.6 | License scanning includes vendored code, not just package-manager dependencies | **MEDIUM** |
| 4.1.7 | License scan results are stored as build artifacts for audit trail | **LOW** |

**Reference**: See `criteria/compliance/license-compatibility.md` Sections 8-9 for
detailed tool configuration and CI integration examples.

### 4.2 Vulnerability Scanning Gates

| # | Checkpoint | Severity |
|---|---|---|
| 4.2.1 | Dependency vulnerability scanning runs on every PR | **HIGH** |
| 4.2.2 | CRITICAL vulnerability findings block merges | **HIGH** |
| 4.2.3 | Container image scanning runs before image promotion to production registry | **HIGH** |
| 4.2.4 | SAST (Static Application Security Testing) is integrated into CI (Semgrep, CodeQL, Bandit, ESLint security rules, SonarQube) | **HIGH** |
| 4.2.5 | DAST (Dynamic Application Security Testing) runs against staging environments on a scheduled basis | **MEDIUM** |
| 4.2.6 | Vulnerability findings are tracked in an issue tracker, not just CI logs | **MEDIUM** |
| 4.2.7 | Remediation SLAs are defined and tracked: CRITICAL 24-72h, HIGH 1-2 weeks, MEDIUM 30 days, LOW 90 days | **MEDIUM** |
| 4.2.8 | Accepted-risk exceptions include justification, approver, and expiration date | **MEDIUM** |
| 4.2.9 | Infrastructure-as-code scanning is integrated (Checkov, tfsec, KICS, Bridgecrew) | **MEDIUM** |
| 4.2.10 | Secret scanning is integrated (GitLeaks, TruffleHog, GitHub secret scanning) | **HIGH** |

#### What to Look For

```regex
# SAST tools (positive indicators)
(?i)(semgrep|codeql|bandit|sonar|sonarqube|sonarcloud|eslint.*security|brakeman|gosec|clippy|pylint.*security)

# DAST tools
(?i)(owasp.?zap|burp|nuclei|nikto|arachni|dast)

# IaC scanning tools
(?i)(checkov|tfsec|kics|bridgecrew|terrascan|cfn.?nag|cfn.?lint)

# Secret scanning tools
(?i)(gitleaks|trufflehog|git.?secrets|detect.?secrets|secret.?scan)

# Vulnerability SLA configuration
(?i)(remediation.?sla|fix.?by|patch.?deadline|vuln.?sla|security.?sla)
```

### 4.3 Compliance Reporting

| # | Checkpoint | Severity |
|---|---|---|
| 4.3.1 | Compliance scan results are aggregated into a dashboard or report | **MEDIUM** |
| 4.3.2 | Compliance status is tracked over time (trending, not just point-in-time) | **LOW** |
| 4.3.3 | Compliance reports are generated automatically (not manually assembled for each audit) | **MEDIUM** |
| 4.3.4 | Reports include: scan date, scope, findings summary, exception list, and remediation status | **MEDIUM** |
| 4.3.5 | Reports are accessible to auditors without requiring code repository access | **LOW** |
| 4.3.6 | Policy-as-code is used (OPA/Rego, Sentinel, Kyverno) to define and enforce compliance rules | **LOW** |
| 4.3.7 | Compliance automation covers all applicable frameworks (not just one) | **LOW** |

#### What to Look For

```regex
# Policy-as-code tools
(?i)(opa|open.?policy|rego|sentinel|kyverno|gatekeeper|conftest)

# Compliance dashboard/reporting
(?i)(compliance.?report|compliance.?dashboard|audit.?report|compliance.?status)

# Automated evidence collection
(?i)(evidence.?collect|audit.?evidence|compliance.?evidence|control.?evidence)
```

---

## 5. Evidence Collection

Auditors require specific types of evidence to validate that controls are
operating effectively. This section maps evidence requirements to code-level
artifacts.

### 5.1 Audit Log Requirements

| # | Checkpoint | Severity |
|---|---|---|
| 5.1.1 | Application audit logs capture: who (user/service identity), what (action performed), when (UTC timestamp), where (source IP/service), outcome (success/failure), and target (resource affected) | **HIGH** |
| 5.1.2 | Audit log schema is consistent across all services (shared structured logging format) | **MEDIUM** |
| 5.1.3 | Audit logs are forwarded to a centralized, tamper-evident logging system (SIEM, CloudWatch, Stackdriver, Datadog, Splunk, ELK with immutable storage) | **HIGH** |
| 5.1.4 | Audit logs are protected from modification or deletion by application code or application-level users | **HIGH** |
| 5.1.5 | Audit log retention meets the most stringent applicable requirement: PCI-DSS 12 months, HIPAA 6 years, SOC 2 per policy (typically 1+ year), FedRAMP per policy (typically 1-3 years) | **HIGH** |
| 5.1.6 | Sensitive data (passwords, tokens, PII, PHI, CHD) is excluded from audit logs | **HIGH** |
| 5.1.7 | Audit logs include correlation IDs to trace a request across multiple services | **MEDIUM** |
| 5.1.8 | Clock synchronization (NTP) is configured on all systems generating audit logs | **MEDIUM** |
| 5.1.9 | Audit log entries are generated for all security-relevant events: authentication (success/failure), authorization (success/failure), privilege escalation, data access, configuration changes, user lifecycle changes | **HIGH** |
| 5.1.10 | Audit logging is implemented at the application layer, not solely relying on infrastructure logging | **MEDIUM** |

#### Audit Log Schema Example

```json
{
  "timestamp": "2024-03-15T14:22:33.456Z",
  "level": "audit",
  "event_type": "data.access",
  "actor": {
    "user_id": "usr_abc123",
    "role": "analyst",
    "ip_address": "10.0.1.42",
    "session_id": "sess_xyz789"
  },
  "action": "read",
  "resource": {
    "type": "patient_record",
    "id": "rec_456def",
    "classification": "phi"
  },
  "outcome": "success",
  "correlation_id": "req_abc-def-ghi",
  "service": "clinical-api",
  "environment": "production"
}
```

#### What to Look For

```regex
# Audit logging implementation (positive indicators)
(?i)(audit.?log|audit.?trail|audit.?event|security.?log|access.?log)

# Structured logging (positive indicators)
(?i)(structlog|json.?log|structured.?log|log\.WithField|log\.With\(|slog\.|zerolog\.|zap\.Logger)

# Correlation ID propagation
(?i)(correlation.?id|request.?id|trace.?id|x-request-id|x-correlation-id)

# Sensitive data in logs (negative pattern)
(?i)(log|logger|audit)\.\w+\(.*\b(password|token|secret|ssn|credit.?card|cvv|authorization)\b

# Log forwarding configuration
(?i)(fluentd|fluent-bit|logstash|filebeat|vector|cloudwatch.?logs|stackdriver|datadog.?agent|splunk.?forwarder)
```

### 5.2 Access Records

| # | Checkpoint | Severity |
|---|---|---|
| 5.2.1 | User access provisioning is tracked: who was granted access, by whom, when, and what level | **HIGH** |
| 5.2.2 | User access deprovisioning is tracked: access revoked on role change or departure | **HIGH** |
| 5.2.3 | Periodic access reviews are conducted and documented (quarterly for privileged access, annually for standard access) | **HIGH** |
| 5.2.4 | Service account access is inventoried: each service account has a documented owner, purpose, and permission scope | **MEDIUM** |
| 5.2.5 | API key lifecycle is managed: creation, rotation, and revocation are tracked | **MEDIUM** |
| 5.2.6 | SSH key lifecycle is managed: authorized keys are tracked, rotated, and revoked on personnel changes | **MEDIUM** |
| 5.2.7 | IAM policies are version controlled (Terraform, CloudFormation, Pulumi) for auditability | **HIGH** |
| 5.2.8 | Emergency/break-glass access is logged and reviewed after each use | **HIGH** |
| 5.2.9 | Access records are retained for the duration required by applicable regulations | **MEDIUM** |
| 5.2.10 | Third-party access (contractor, vendor) is separately tracked with start/end dates | **MEDIUM** |

#### What to Look For

```regex
# IAM as code (positive indicators)
(?i)(aws_iam|azurerm_role|google_project_iam|iam.?policy|role.?binding|service.?account)

# Access review references
(?i)(access.?review|entitlement.?review|permission.?audit|rbac.?review)

# Service account management
(?i)(service.?account|service.?principal|workload.?identity|machine.?identity)

# Key rotation
(?i)(key.?rotation|rotate.?key|key.?expir|credential.?rotation)
```

### 5.3 Change Approval Records

| # | Checkpoint | Severity |
|---|---|---|
| 5.3.1 | All production changes are traceable to an approved change request (PR, ticket, or change management record) | **HIGH** |
| 5.3.2 | Change approval records include: what changed, who approved, when it was approved, and the risk assessment | **HIGH** |
| 5.3.3 | Production deployment records include: who initiated the deployment, what version/commit was deployed, when, and whether it succeeded | **HIGH** |
| 5.3.4 | Rollback records are maintained: what was rolled back, when, why, and what version was restored | **MEDIUM** |
| 5.3.5 | Emergency changes follow a documented expedited process with retrospective approval | **MEDIUM** |
| 5.3.6 | Change management records are retained for the duration required by applicable regulations | **MEDIUM** |
| 5.3.7 | Infrastructure changes are managed through the same change control process as application changes (IaC) | **HIGH** |
| 5.3.8 | Database schema changes (migrations) are version controlled and follow the change approval process | **MEDIUM** |
| 5.3.9 | Configuration changes (feature flags, environment variables, settings) are tracked with the same rigor as code changes | **MEDIUM** |
| 5.3.10 | A change advisory board (CAB) or equivalent review process exists for high-risk changes in regulated environments | **MEDIUM** |

#### What to Look For

```regex
# Change management references
(?i)(change.?management|change.?advisory|cab|change.?request|change.?ticket|change.?control)

# Deployment tracking
(?i)(deploy.?log|deployment.?record|release.?log|deploy.?history|deploy.?audit)

# Database migration management
(?i)(migration|migrate|flyway|liquibase|alembic|knex\.migrate|sequelize.*migration|prisma.*migrate|django.*migrate)

# Configuration management
(?i)(feature.?flag|launchdarkly|split\.io|flagsmith|unleash|configmap|parameter.?store|app.?config)
```

---

## 6. Third-Party Risk Management

Third-party dependencies and services extend the attack surface and compliance
obligations of the project. Auditors evaluate whether the organization has
visibility into and control over its third-party risk. This maps to SOC 2
(CC9.2 vendor management), PCI-DSS (Req. 12.8 service provider management),
HIPAA (164.308(b)(1) business associate contracts), FedRAMP (SA-9 external
information system services), and GDPR Art. 28 (processor obligations).

### 6.1 Vendor Assessment

| # | Checkpoint | Severity |
|---|---|---|
| 6.1.1 | A vendor inventory exists listing all third-party services and libraries used by the application | **HIGH** |
| 6.1.2 | Each vendor is classified by risk level based on data access: High (processes PII/PHI/CHD), Medium (has infrastructure access), Low (no data or infrastructure access) | **MEDIUM** |
| 6.1.3 | High-risk vendors have completed a security assessment (SOC 2 report, ISO 27001 certification, penetration test results, or vendor security questionnaire) | **HIGH** |
| 6.1.4 | Vendor security assessments are reviewed and renewed periodically (annually for high-risk, every 2-3 years for medium-risk) | **MEDIUM** |
| 6.1.5 | Vendor contracts include security and privacy requirements (data handling, breach notification, data return/deletion on termination) | **HIGH** |
| 6.1.6 | Vendor service level agreements (SLAs) align with the project's availability requirements | **MEDIUM** |
| 6.1.7 | Vendor business continuity and disaster recovery capabilities are assessed for critical dependencies | **MEDIUM** |
| 6.1.8 | Vendor lock-in risk is assessed: data portability, API compatibility, and migration feasibility | **LOW** |
| 6.1.9 | Open-source dependency maintainer health is assessed for critical libraries: bus factor, maintenance activity, security response history | **MEDIUM** |
| 6.1.10 | Vendor access to the project's systems and data is documented and follows least-privilege | **HIGH** |

### 6.2 Sub-Processor Tracking

**Regulation reference**: GDPR Art. 28(2) (sub-processor authorization), HIPAA 45 CFR 164.502(e)(1)(ii) (subcontractor BAAs)

| # | Checkpoint | Severity |
|---|---|---|
| 6.2.1 | Sub-processors (third parties used by your third-party vendors to process data on your behalf) are identified | **MEDIUM** |
| 6.2.2 | Sub-processor lists are monitored for changes (many vendors publish sub-processor lists and notify of changes) | **MEDIUM** |
| 6.2.3 | Sub-processors that handle regulated data (PII, PHI, CHD) have appropriate agreements in place (DPAs, BAAs) | **HIGH** |
| 6.2.4 | Sub-processor data processing locations are known and comply with data residency requirements | **MEDIUM** |
| 6.2.5 | The application's privacy notice or DPA lists sub-processors or provides a link to the sub-processor list | **MEDIUM** |
| 6.2.6 | A process exists to evaluate and approve new sub-processors before they begin processing data | **MEDIUM** |
| 6.2.7 | Sub-processor changes trigger a re-assessment of the risk profile | **LOW** |

### 6.3 Vendor Inventory Format

A well-maintained vendor inventory should include:

```markdown
# Third-Party Service Inventory

| Vendor | Service | Data Classification | Agreement | Last Assessment | Sub-Processors |
|---|---|---|---|---|---|
| AWS | Infrastructure (EC2, RDS, S3) | PHI, PII, CHD | BAA, DPA | 2024-01 (SOC 2 Type II) | See AWS sub-processor list |
| Stripe | Payment processing | CHD | DPA, PCI attestation | 2024-03 (PCI-DSS AOC) | See Stripe sub-processor list |
| Datadog | Monitoring & logging | Internal (no PII) | DPA | 2024-02 (SOC 2 Type II) | See Datadog sub-processor list |
| SendGrid | Transactional email | PII (email, name) | DPA | 2024-01 (SOC 2 Type II) | See Twilio sub-processor list |
| Auth0 | Authentication | PII (email, auth data) | BAA, DPA | 2024-02 (SOC 2 Type II) | See Okta sub-processor list |
```

### 6.4 Open-Source Dependency Risk

| # | Checkpoint | Severity |
|---|---|---|
| 6.4.1 | Critical dependencies (those in the hot path, handling sensitive data, or with broad permissions) are identified | **MEDIUM** |
| 6.4.2 | Critical dependencies are assessed for maintenance health: commit frequency, issue response time, number of maintainers | **MEDIUM** |
| 6.4.3 | Dependencies with a single maintainer that are critical to the project are flagged as a bus-factor risk | **LOW** |
| 6.4.4 | Deprecated dependencies are identified and scheduled for replacement | **MEDIUM** |
| 6.4.5 | Dependencies that have been archived, abandoned, or have not been updated in over 2 years for actively maintained ecosystems are flagged | **MEDIUM** |
| 6.4.6 | Dependency funding and sustainability are considered for critical libraries (this is informational, not a blocking concern) | **INFO** |
| 6.4.7 | Forked dependencies are documented with the reason for forking and the plan for staying in sync with upstream | **MEDIUM** |

#### What to Look For

```regex
# Vendor inventory references
(?i)(vendor.?inventory|third.?party.?inventory|service.?catalog|dependency.?inventory|vendor.?register)

# Assessment documentation
(?i)(security.?assessment|vendor.?assessment|risk.?assessment|due.?diligence|security.?questionnaire)

# DPA/BAA references
(?i)(data.?processing.?agreement|DPA|business.?associate|BAA|sub.?processor)

# Deprecated dependencies
(?i)(deprecated|end.?of.?life|eol|no.?longer.?maintained|archived|unmaintained)

# Forked dependencies
(?i)(fork|forked)\s+(of|from)\s+
```

---

## Review Procedure Summary

When evaluating audit readiness:

1. **Assess dependency management**: Verify lock files are committed, sources
   are trusted, vulnerability scanning is automated, and SBOMs are generated.
   These are table-stakes for any compliance audit (Section 1).

2. **Evaluate change documentation**: Confirm changelogs are maintained,
   changes are traceable to requirements, and code reviews are enforced. Auditors
   will sample change records to verify the control operates consistently
   (Section 2).

3. **Review security documentation**: Check for SECURITY.md, vulnerability
   disclosure process, security policy, and incident response references.
   Missing security documentation is a common audit finding across all frameworks
   (Section 3).

4. **Verify compliance automation**: Confirm that license checks, vulnerability
   scanning, SAST, and secret scanning are integrated into CI/CD with
   appropriate gates. Automated controls are stronger audit evidence than manual
   controls (Section 4).

5. **Inspect evidence collection**: Verify audit log completeness, access
   record management, and change approval records. Auditors will request
   specific evidence items; the ability to produce them quickly demonstrates
   control maturity (Section 5).

6. **Evaluate third-party risk management**: Confirm vendor inventory exists,
   assessments are current, sub-processors are tracked, and contracts include
   appropriate provisions. Supply chain risk is a growing audit focus area
   (Section 6).

7. **Map gaps to frameworks**: For each finding, identify which compliance
   framework(s) it affects. A single audit readiness gap may impact SOC 2,
   PCI-DSS, HIPAA, and FedRAMP simultaneously.

8. **Classify each finding** using the severity tables in each section.
   Prioritize CRITICAL and HIGH findings that would cause audit failures.
   MEDIUM findings represent control weaknesses that auditors will note.

9. **Provide framework-specific references**: When reporting findings, reference
   the specific control or requirement (e.g., SOC 2 CC8.1, PCI-DSS Req. 6.3,
   NIST 800-53 CM-3, HIPAA 164.312(b)) to help teams understand the regulatory
   context.

10. **Assess operational maturity**: Beyond individual controls, evaluate
    whether compliance is treated as a continuous process (automated, monitored,
    improved) or a periodic event (manual, point-in-time). Continuous compliance
    maturity is a strong positive signal for auditors.
