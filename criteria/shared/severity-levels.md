# Severity Levels

All oversight agents use the same five severity levels. Consistent severity classification ensures that findings from different agents can be triaged, prioritized, and compared in a unified report.

## Level Definitions

### CRITICAL

**Action required**: Must fix before merge. Block the PR/deployment.

A CRITICAL finding indicates an active, exploitable vulnerability, imminent data loss risk, or a condition that would cause immediate harm in production. These are not theoretical concerns -- they represent issues where the damage path is short and the blast radius is large.

**Qualifies as CRITICAL when**:

- Credentials, API keys, tokens, or private keys are exposed in source code, configuration, or commit history
- A known CVE with a public exploit is present in a direct dependency at the version in use
- SQL injection, command injection, or other injection flaws exist with unsanitized user input reaching an execution context
- Authentication or authorization is completely absent on a sensitive endpoint or operation
- Data is being written to a publicly accessible location without access controls (e.g., world-readable S3 bucket, open NFS export)
- Encryption is disabled or broken for data that is regulated or contains PII (e.g., TLS disabled, plaintext password storage with no hashing)
- A configuration would cause data loss on deployment (e.g., `DROP TABLE` in a migration without a backup step, destructive Terraform plan on production state)
- Firewall rules permit unrestricted inbound access (0.0.0.0/0) to management interfaces (SSH, RDP, iDRAC, iLO, vCenter)
- SNMP v1/v2c with default or well-known community strings on production network equipment
- BGP sessions configured without authentication, enabling route hijacking

**Examples**:

- `AWS_SECRET_ACCESS_KEY=AKIA...` hardcoded in `config.py`
- `password = "admin123"` in a database connection string committed to the repo
- `iptables -A INPUT -p tcp --dport 22 -j ACCEPT` with no source restriction on a public-facing host
- A Terraform `destroy` operation targeting a production database with no lifecycle protection
- An API endpoint that returns user records with no authentication check

---

### HIGH

**Action required**: Should fix before merge. Strong recommendation to block.

A HIGH finding represents a significant security, quality, or reliability gap where the risk is concrete but requires some additional conditions to be exploitable, or where missing safeguards would make incident response substantially harder.

**Qualifies as HIGH when**:

- Authentication exists but is weak or bypassable (e.g., JWT without signature verification, session tokens in URL parameters)
- Authorization checks are present but incomplete (e.g., horizontal privilege escalation possible, missing role checks on some operations)
- Sensitive data is logged or exposed in error messages (stack traces, connection strings, internal IPs in user-facing responses)
- Dependencies have known HIGH-severity CVEs without public exploits, or CRITICAL CVEs in transitive dependencies
- Error handling is absent in critical paths, risking uncontrolled failures (e.g., no try/catch around database transactions, no error handling on file operations that affect state)
- Infrastructure lacks redundancy where failure would cause an outage (e.g., single-point-of-failure in a production path with no failover)
- Backup or recovery mechanisms are missing for stateful systems
- TLS is configured but with weak cipher suites, expired certificates, or missing certificate validation
- Network segmentation violations where traffic flows cross trust boundaries without inspection
- VLAN hopping possible due to trunk port misconfiguration on access ports
- RAID arrays in degraded state or configured without a hot spare on production systems

**Examples**:

- `except Exception: pass` swallowing errors in a payment processing function
- A REST API that checks authentication but never validates that user A can access user B's resources
- `verify=False` on an HTTPS request to an internal service carrying credentials
- A Kubernetes deployment with no resource limits, risking node-level resource exhaustion
- A VMware HA cluster with admission control disabled

---

### MEDIUM

**Action required**: Fix in current sprint. Do not let this accumulate.

A MEDIUM finding represents a best-practice violation or technical debt that does not pose an immediate threat but will compound over time, increase the attack surface, degrade maintainability, or make future incidents more likely or harder to diagnose.

**Qualifies as MEDIUM when**:

- Input validation is present but incomplete (e.g., length limits but no character filtering, client-side only validation)
- Code duplication spans multiple files, creating divergence risk for business logic
- Functions or methods exceed reasonable complexity thresholds (cyclomatic complexity > 15, functions > 100 lines)
- Test coverage is significantly below project norms for changed code (e.g., new module with 0% coverage when project averages 70%)
- Configuration values are hardcoded that should be externalized (non-secret values like timeouts, retry counts, feature flags)
- Logging is insufficient for operational debugging (e.g., no request IDs, no correlation between related operations)
- Documentation is missing for public APIs, exported functions, or complex business logic
- Deprecated APIs or libraries are in use with known replacements available
- Container images use a mutable tag (`:latest`) instead of a pinned digest or version
- DNS TTLs are set inappropriately (too high for dynamic environments, too low causing excessive queries)
- Network ACLs are overly permissive (broader than required) but not completely open
- Storage volumes lack monitoring or alerting on capacity thresholds

**Examples**:

- A 200-line function that handles validation, transformation, persistence, and notification in a single method
- Three copies of the same retry logic scattered across different service clients
- A new REST controller with no unit tests while the rest of the project has 80% coverage
- `FROM node:latest` in a production Dockerfile
- An OSPF configuration advertising more specific routes than necessary, causing routing table bloat

---

### LOW

**Action required**: Fix when convenient. Track as backlog items.

A LOW finding represents a minor improvement opportunity or a style issue that has some measurable impact on readability, maintainability, or operational efficiency, but does not meaningfully affect security, reliability, or correctness.

**Qualifies as LOW when**:

- Naming conventions are inconsistent within a module (e.g., mixing camelCase and snake_case in the same file)
- Code comments are stale, misleading, or refer to removed functionality
- Magic numbers or string literals are used where named constants would improve clarity
- Import ordering, file organization, or module structure deviates from project conventions
- Log levels are inappropriate (e.g., using `INFO` for debug-level detail, `ERROR` for non-error conditions)
- Configuration file formatting is inconsistent (e.g., mixed indentation, inconsistent quoting)
- Minor inefficiencies exist that do not affect user-visible performance (e.g., unnecessary object copies in a non-hot path)
- Documentation exists but has minor gaps, typos, or formatting issues
- Network interface descriptions or port labels are missing or inconsistent
- VM names or resource tags do not follow organizational naming conventions
- Storage LUN naming does not follow a consistent scheme

**Examples**:

- `x = 86400` instead of `SECONDS_PER_DAY = 86400`
- A function named `processData()` that only validates input
- `console.log("here")` left in non-debug code
- A Terraform resource missing the standard set of tags required by organizational policy
- Interface descriptions on a switch that say "uplink" without specifying the destination

---

### INFO

**Action required**: None. For awareness only.

An INFO finding is an observation, suggestion, or positive callout. It provides context that reviewers or maintainers may find useful but does not require any change. INFO findings are also used to highlight particularly good practices worth replicating elsewhere.

**Qualifies as INFO when**:

- A pattern is well-implemented and worth noting as a positive example for the team
- An alternative approach exists that might be worth considering in the future but is not clearly better
- A dependency or technology choice is noted for awareness (e.g., "this library is maintained by a single developer")
- The codebase would benefit from a future refactoring that is out of scope for the current change
- A configuration is correct but uses a non-default value that operators should be aware of
- Metrics, monitoring, or observability hooks could be added to improve operational visibility
- A design decision is noted that affects future extensibility but is fine for current requirements

**Examples**:

- "Good use of the circuit breaker pattern here -- consider applying the same approach to the payment service client."
- "This function is correct but could be simplified with the `itertools.groupby` standard library function."
- "Note: this project uses `pnpm` workspaces; CI must use `pnpm install` rather than `npm install`."
- "The BGP configuration correctly uses route maps and prefix lists. Consider documenting the maximum-prefix limits in the network runbook."
- "Terraform state is stored in S3 with DynamoDB locking -- this is the recommended pattern."

---

## Severity in Context

### Severity Can Depend on Environment

The same finding may warrant different severities depending on context:

| Finding | Internal tool | Customer-facing production |
|---------|--------------|---------------------------|
| Missing rate limiting | LOW | HIGH |
| No input validation on admin form | MEDIUM | HIGH |
| Plaintext HTTP for health checks | LOW | MEDIUM |
| Default SNMP community string | MEDIUM (lab) | CRITICAL (production) |

Agents should note when severity is context-dependent and explain their reasoning.

### Severity Threshold Configuration

Projects can configure a severity threshold in `.oversight.yml` to filter the report output:

```yaml
agents:
  config:
    security:
      severity-threshold: medium  # Only report MEDIUM and above
```

When a threshold is set, findings below that level are still counted in the summary table but their details are collapsed or omitted from the findings section.

## Summary

| Level | Meaning | Action | Blocks Merge |
|-------|---------|--------|--------------|
| CRITICAL | Active exploit / data loss risk | Must fix immediately | Yes |
| HIGH | Significant gap, concrete risk | Should fix before merge | Recommended |
| MEDIUM | Best practice violation, compounding debt | Fix in current sprint | No |
| LOW | Minor improvement, style with impact | Fix when convenient | No |
| INFO | Observation or positive callout | No action required | No |
