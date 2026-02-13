# Review Summary

> **{PASS | WARN | FAIL}** -- {one-line verdict description}

<!-- Verdict examples:
     PASS -- "No critical or high-severity issues found. 2 medium findings noted."
     WARN -- "1 high-severity finding requires attention before merge."
     FAIL -- "2 critical findings must be resolved. Do not merge."
-->

---

## Severity Overview

| Severity | Count |
|----------|-------|
| CRITICAL | {N}   |
| HIGH     | {N}   |
| MEDIUM   | {N}   |
| LOW      | {N}   |
| INFO     | {N}   |

**Total findings**: {N} across {FILE_COUNT} files reviewed

---

## Top Findings

<!--
  List up to 3 findings, prioritized by severity (CRITICAL first, then HIGH).
  If there are no CRITICAL or HIGH findings, show the top MEDIUM findings instead.
  If there are no findings at all, replace this section with:
  "No issues found. All reviewed files passed {agent-name} checks."

  Use the abbreviated format below -- no evidence blocks, no references.
  The full report contains complete details.
-->

1. **[{SEVERITY}] {Finding Title}**
   `{path/to/file}` (lines {X}-{Y})
   {One-to-two sentence description of the finding and its impact.}

2. **[{SEVERITY}] {Finding Title}**
   `{path/to/file}` (lines {X}-{Y})
   {One-to-two sentence description.}

3. **[{SEVERITY}] {Finding Title}**
   `{path/to/file}` (lines {X}-{Y})
   {One-to-two sentence description.}

---

## Agents

<!-- For single-agent runs, show one row. For orchestrated runs, list all. -->

| Agent | Findings | Verdict |
|-------|----------|---------|
| {agent-id} | {N} | {PASS \| WARN \| FAIL} |
| {agent-id} | {N} | {PASS \| WARN \| FAIL} |

---

## Scope

- **Mode**: {diff | full | targeted}
- **Base**: `{base-ref}` --> `{head-ref}`
- **Files reviewed**: {N}
- **Timestamp**: {YYYY-MM-DD HH:MM UTC}

---

{N} additional findings at MEDIUM or below -- see full report for details.

<!--
  TEMPLATE USAGE NOTES (remove this section from actual reports):

  1. This template is designed for PR comments, Slack notifications, and quick
     status checks. Keep it concise. The full report provides all details.

  2. Verdict logic matches the full report:
     - FAIL  = any CRITICAL, or HIGH count exceeds threshold
     - WARN  = any HIGH, or MEDIUM count exceeds threshold
     - PASS  = no CRITICAL or HIGH, MEDIUM within threshold

  3. The "Top Findings" section caps at 3 items. For FAIL verdicts, always
     lead with CRITICAL findings. For WARN verdicts, lead with HIGH findings.

  4. The trailing line about additional findings should be omitted if all
     findings are shown in the Top Findings section, or if the total count
     is 3 or fewer.

  5. For PR comment usage, this entire summary can be posted as a single
     comment. GitHub and GitLab both render the Markdown correctly.

  6. When used in CI, the verdict (PASS/WARN/FAIL) maps to exit codes:
     - PASS = exit 0
     - WARN = exit 0 (with annotation)
     - FAIL = exit 1 (blocks pipeline)
-->
