# Finding Schema

Every finding produced by any oversight agent must conform to this schema. Consistent structure enables automated parsing, cross-agent deduplication, severity-based filtering, and unified reporting through the orchestrator.

## Individual Finding Format

Each finding is a third-level Markdown heading with the severity in brackets, followed by structured metadata fields.

```markdown
### [SEVERITY] Short Title

- **Agent**: agent-name
- **File**: `path/to/file` (lines X-Y)
- **Category**: Category Name
- **Finding**: Description of what was found.
- **Evidence**:
  ```language
  code snippet or configuration excerpt
  ```
- **Recommendation**: What to do about it.
- **Reference**: Standard/guideline reference (e.g., OWASP A07:2021)
```

### Field Definitions

#### Severity (required)

One of: `CRITICAL`, `HIGH`, `MEDIUM`, `LOW`, `INFO`. Must be uppercase and enclosed in square brackets in the heading. See `criteria/shared/severity-levels.md` for full definitions.

#### Short Title (required)

A concise, descriptive title for the finding. Should be specific enough to distinguish it from similar findings.

- Good: "Hardcoded Database Password in Connection String"
- Bad: "Security Issue"
- Good: "Missing Input Validation on User Registration Endpoint"
- Bad: "Input Problem"

#### Agent (required)

The agent that produced the finding. Use the agent's canonical identifier:

| Agent | Identifier |
|-------|-----------|
| Security | `security-reviewer` |
| Documentation | `documentation-reviewer` |
| Planning & Progress | `planning-reviewer` |
| Code Quality | `quality-reviewer` |
| Testing & QA | `testing-reviewer` |
| Compliance & Licensing | `compliance-reviewer` |
| Architecture | `architecture-reviewer` |
| DevOps/Platform | `devops-reviewer` |
| Networking | `networking-reviewer` |
| Virtualization | `virtualization-reviewer` |
| Storage | `storage-reviewer` |
| Compute | `compute-reviewer` |
| Accessibility | `accessibility-reviewer` |

#### File (required for file-specific findings)

The file path relative to the repository root, enclosed in backticks. Include the line range in parentheses when applicable. Omit this field only for project-level or cross-cutting findings that do not map to a specific file.

Formats:
- Single file with lines: `` `src/auth/login.py` (lines 42-58) ``
- Single file without lines: `` `Dockerfile` ``
- Directory-level: `` `src/api/` ``
- Omitted: for project-wide observations (e.g., "No CI pipeline configured")

#### Category (required)

A standardized category name that groups related findings. Each agent defines its own categories in its criteria files. Examples:

- Security: Hardcoded Credentials, Injection, Authentication, Authorization, Cryptography
- Quality: Code Complexity, Duplication, Error Handling, Naming Conventions
- Architecture: Coupling, Layering Violation, Missing Abstraction
- DevOps: Container Security, IaC Hygiene, CI/CD Configuration
- Networking: Routing Security, Firewall Rules, Protocol Configuration

Categories must be consistent within an agent's findings across runs. Do not invent ad-hoc categories per finding.

#### Finding (required)

A clear description of what was found. This should be factual and specific:
- State what the issue is, not just that an issue exists
- Include enough context for someone unfamiliar with the code to understand the problem
- For INFO findings, describe the observation or positive pattern

Good: "The `authenticate()` function compares passwords using `==` instead of a constant-time comparison function, making it vulnerable to timing attacks."

Bad: "There is a security issue with password comparison."

#### Evidence (conditional)

A code snippet, configuration excerpt, command output, or other concrete evidence supporting the finding. Use fenced code blocks with the appropriate language identifier for syntax highlighting.

**When to include evidence**:
- CRITICAL and HIGH findings: Always include evidence. These findings must be verifiable.
- MEDIUM findings: Include evidence when it aids understanding. May be omitted for well-understood patterns (e.g., "function exceeds 200 lines").
- LOW findings: Include evidence when the issue is not obvious from the description alone.
- INFO findings: Include evidence when highlighting a specific positive pattern. Omit for general observations.

**Evidence formatting**:
- Use the correct language identifier (```python, ```yaml, ```hcl, ```bash, etc.)
- Keep snippets focused -- show only the relevant lines, not entire files
- If the snippet requires context, add a brief comment or ellipsis to indicate surrounding code
- For configuration findings, show both the current (problematic) state and the expected state when helpful

```markdown
- **Evidence**:
  ```python
  # Current (vulnerable)
  if user_password == stored_hash:
      return True
  ```
  Expected:
  ```python
  import hmac
  if hmac.compare_digest(user_password, stored_hash):
      return True
  ```
```

#### Recommendation (required)

A specific, actionable recommendation for addressing the finding. Should describe what to do, not just what is wrong. Include enough detail that a developer can act on it without extensive research.

- Good: "Replace the hardcoded password with an environment variable. Use `os.environ['DB_PASSWORD']` and ensure the variable is set via your secrets management system (Vault, AWS Secrets Manager, etc.)."
- Bad: "Fix the password."
- Good: "No action required. This is a well-implemented pattern."
- For INFO findings: "No action required." followed by any relevant suggestion or context.

#### Reference (optional but encouraged)

A reference to an external standard, guideline, or documented best practice. This adds authority to the finding and helps teams understand the broader context.

Common reference frameworks:
- **OWASP**: `OWASP A01:2021 - Broken Access Control`
- **CIS Benchmarks**: `CIS Docker Benchmark 4.1`
- **NIST**: `NIST SP 800-53 AC-6`
- **CVE**: `CVE-2024-1234`
- **RFC**: `RFC 7525 Section 3.1`
- **Vendor docs**: `Cisco IOS Security Guide - Control Plane Protection`
- **Language/framework**: `PEP 484`, `ESLint rule no-eval`, `Go Code Review Comments`

Omit this field rather than inventing a vague reference. Not every finding maps to a published standard.

---

## Summary Table Format

Every report begins with a severity count summary table. This provides an at-a-glance assessment of the review results.

```markdown
| Severity | Count |
|----------|-------|
| CRITICAL | 0     |
| HIGH     | 1     |
| MEDIUM   | 3     |
| LOW      | 2     |
| INFO     | 1     |
```

Rules for the summary table:
- Always include all five severity levels, even if the count is zero
- Order from CRITICAL (top) to INFO (bottom)
- Counts reflect the final deduplicated findings (after orchestrator merging, if applicable)
- When a severity threshold is configured, include a note below the table: "*Findings below MEDIUM threshold are counted but details are omitted.*"

---

## Multi-File Findings

Some findings span multiple files. Use the following extended format when a single finding involves more than one location.

```markdown
### [MEDIUM] Duplicated Retry Logic Across Service Clients

- **Agent**: quality-reviewer
- **Files**:
  - `src/clients/payment.py` (lines 15-32)
  - `src/clients/shipping.py` (lines 22-39)
  - `src/clients/inventory.py` (lines 18-35)
- **Category**: Duplication
- **Finding**: The same retry-with-backoff logic is implemented independently in three service clients, with minor variations in backoff multiplier and max retries.
- **Evidence**:
  `src/clients/payment.py`:
  ```python
  for attempt in range(3):
      try:
          return self._post(url, data)
      except ConnectionError:
          time.sleep(2 ** attempt)
  ```
  `src/clients/shipping.py`:
  ```python
  for attempt in range(5):  # Different max retries
      try:
          return self._post(url, data)
      except ConnectionError:
          time.sleep(1.5 ** attempt)  # Different backoff
  ```
- **Recommendation**: Extract a shared retry decorator or utility function with configurable max retries and backoff parameters. Apply it consistently across all service clients.
- **Reference**: DRY Principle
```

Key differences from single-file format:
- Use **Files** (plural) instead of **File**
- List each file on its own indented line
- Label each evidence block with its source file
- The finding description should explain the cross-file relationship

---

## Deduplication Rules

When the orchestrator combines findings from multiple agents, duplicate or overlapping findings must be merged. The following rules govern deduplication.

### Rule 1: Exact File + Line Overlap

If two findings reference the same file and overlapping line ranges, they are candidates for deduplication.

- **Same category**: Merge into a single finding. Keep the higher severity. Combine recommendations. List both agents in the Agent field (e.g., `security-reviewer, quality-reviewer`).
- **Different category**: Keep both findings. They represent different concerns about the same code (e.g., security vs. quality).

### Rule 2: Same Category + Same File (Non-Overlapping Lines)

If two findings are in the same file and same category but different lines, keep both as separate findings. They may represent distinct instances of the same problem pattern.

### Rule 3: Same Category + Different Files

Keep both findings. Even if the category is the same, findings in different files are distinct issues that need separate remediation.

### Rule 4: Superseding Findings

If a higher-severity finding encompasses a lower-severity one (e.g., CRITICAL "Exposed AWS Credentials" supersedes LOW "Hardcoded Configuration Value" for the same line), keep only the higher-severity finding.

### Rule 5: INFO Findings Never Deduplicate

INFO findings from different agents are always kept, even if they reference the same code. Different agents may provide different positive observations or suggestions.

### Deduplication Metadata

When findings are merged during deduplication, add a note:

```markdown
- **Merged From**: security-reviewer (CRITICAL), quality-reviewer (HIGH)
```

This preserves traceability to the originating agents.

---

## Ordering

Within a report, findings are ordered by:

1. **Severity** (descending): CRITICAL first, then HIGH, MEDIUM, LOW, INFO
2. **File path** (ascending): Alphabetical by file path within the same severity
3. **Line number** (ascending): By starting line number within the same file

---

## Minimal Valid Finding

The smallest valid finding includes severity, title, agent, category, finding description, and recommendation:

```markdown
### [INFO] Well-Structured Error Handling

- **Agent**: quality-reviewer
- **Category**: Error Handling
- **Finding**: The error handling in the authentication module follows a consistent pattern with specific exception types, structured logging, and appropriate fallback behavior.
- **Recommendation**: No action required. Consider documenting this pattern in the team's style guide for reuse.
```

File and evidence are omitted because this is a general observation rather than a file-specific issue.

---

## Machine-Parseable Conventions

To enable automated processing of findings (e.g., by CI pipelines or reporting tools), agents must follow these conventions strictly:

1. The heading format `### [SEVERITY] Title` must use exactly three `#` characters, one space, an opening bracket, the severity in uppercase, a closing bracket, one space, then the title.
2. Metadata fields use exactly the format `- **Field Name**:` with the field name in bold.
3. The severity in the heading must match one of the five defined levels exactly.
4. No additional fields beyond those defined in this schema. Agents may add context within the Finding or Recommendation text, but must not introduce new top-level metadata fields.
