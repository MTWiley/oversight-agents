# {AGENT_NAME} Review Report

> **Agent**: {AGENT_ID}
> **Scope**: {SCOPE_DESCRIPTION}
> **Reviewed**: {FILE_COUNT} files
> **Timestamp**: {YYYY-MM-DD HH:MM UTC}
> **Mode**: {diff | full | targeted}

---

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | {N}   |
| HIGH     | {N}   |
| MEDIUM   | {N}   |
| LOW      | {N}   |
| INFO     | {N}   |

**Verdict**: {PASS | WARN | FAIL}

<!-- Verdict logic:
     FAIL  = any CRITICAL findings, or HIGH count exceeds threshold
     WARN  = any HIGH findings, or MEDIUM count exceeds threshold
     PASS  = no CRITICAL or HIGH findings, MEDIUM count within threshold

     Default thresholds: FAIL if HIGH > 0, WARN if MEDIUM > 3
     Thresholds are configurable via .oversight.yml -->

---

## Findings

<!--
  Findings are ordered by severity (CRITICAL first), then by file path
  (alphabetical), then by line number (ascending).

  Each finding follows the schema defined in criteria/shared/finding-schema.md.

  Group findings by severity level using second-level headings for clarity
  in longer reports. If a severity level has zero findings, omit that section.
-->

### CRITICAL

<!-- Include this section only if CRITICAL findings exist. -->

#### [CRITICAL] {Finding Title}

- **Agent**: {agent-id}
- **File**: `{path/to/file}` (lines {X}-{Y})
- **Category**: {Category Name}
- **Finding**: {Detailed description of what was found, with enough context for someone unfamiliar with the code to understand the issue and its impact.}
- **Evidence**:
  ```{language}
  {Relevant code snippet or configuration excerpt}
  ```
- **Recommendation**: {Specific, actionable steps to remediate the issue.}
- **Reference**: {Standard or guideline reference, e.g., OWASP A07:2021}

---

### HIGH

<!-- Include this section only if HIGH findings exist. -->

#### [HIGH] {Finding Title}

- **Agent**: {agent-id}
- **File**: `{path/to/file}` (lines {X}-{Y})
- **Category**: {Category Name}
- **Finding**: {Description.}
- **Evidence**:
  ```{language}
  {Code snippet}
  ```
- **Recommendation**: {Remediation steps.}
- **Reference**: {Reference}

---

### MEDIUM

<!-- Include this section only if MEDIUM findings exist. -->

#### [MEDIUM] {Finding Title}

- **Agent**: {agent-id}
- **File**: `{path/to/file}` (lines {X}-{Y})
- **Category**: {Category Name}
- **Finding**: {Description.}
- **Evidence**:
  ```{language}
  {Code snippet}
  ```
- **Recommendation**: {Remediation steps.}
- **Reference**: {Reference}

---

### LOW

<!-- Include this section only if LOW findings exist. -->

#### [LOW] {Finding Title}

- **Agent**: {agent-id}
- **File**: `{path/to/file}` (lines {X}-{Y})
- **Category**: {Category Name}
- **Finding**: {Description.}
- **Recommendation**: {Remediation steps.}

---

### INFO

<!-- Include this section only if INFO findings exist. -->

#### [INFO] {Finding Title}

- **Agent**: {agent-id}
- **Category**: {Category Name}
- **Finding**: {Observation or positive callout.}
- **Recommendation**: No action required. {Optional suggestion.}

---

## Files Reviewed

<!-- List all files included in this review. This provides traceability and
     helps identify gaps if expected files are missing. -->

| # | File | Status |
|---|------|--------|
| 1 | `{path/to/file1}` | {findings_count} findings |
| 2 | `{path/to/file2}` | Clean |
| 3 | `{path/to/file3}` | {findings_count} findings |
| ... | ... | ... |

---

## Metadata

| Field | Value |
|-------|-------|
| Agent Version | {VERSION} |
| Review Mode | {diff \| full \| targeted} |
| Scope | {SCOPE_DESCRIPTION} |
| Base Ref | {git ref, e.g., main, HEAD~1, abc1234} |
| Head Ref | {git ref, e.g., HEAD, feature-branch, def5678} |
| Files in Scope | {N} |
| Files with Findings | {N} |
| Total Findings | {N} |
| Severity Threshold | {configured threshold or "none"} |
| Config Source | {.oversight.yml \| defaults} |
| Duration | {execution time} |

---

<!--
  TEMPLATE USAGE NOTES (remove this section from actual reports):

  1. Replace all {PLACEHOLDER} values with actual data.
  2. Remove severity sections that have zero findings.
  3. The verdict logic should be computed based on findings and thresholds.
  4. Evidence is required for CRITICAL and HIGH, recommended for MEDIUM,
     optional for LOW, and situational for INFO.
  5. The Files Reviewed section helps verify completeness -- every file in
     scope should appear even if it has no findings.
  6. For orchestrated reports combining multiple agents, add an "Agents Run"
     row to the Metadata table listing all agents that contributed.
  7. When findings are deduplicated by the orchestrator, include a
     "Merged From" field per the finding schema deduplication rules.
-->
