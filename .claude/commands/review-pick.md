# Oversight Review - Pick Agents

You run a specific subset of review agents chosen by the user.

## Arguments

`$ARGUMENTS` should be parsed as: `agent1,agent2[,agentN] [scope]`

**Examples:**
- `security,quality` — Run security and quality agents on changed files
- `security,devops full` — Run security and devops agents on entire repo
- `architecture src/` — Run architecture agent on src/ directory
- `security,quality,architecture,devops full` — Run core agents on entire repo
- `networking,security` — Run networking and security agents on changed files
- `testing,quality src/` — Run testing and quality agents on src/

**Parse rules:**
1. Split `$ARGUMENTS` by whitespace into tokens
2. First token is a comma-separated list of agent names
3. Remaining tokens (if any) are the scope (same rules as individual agents: empty/path/"full")

**Valid agent names:** `security`, `quality`, `architecture`, `devops`, `docs`, `planning`, `testing`, `compliance`, `networking`, `virtualization`, `storage`, `compute`, `accessibility`, `observability`

If an invalid agent name is provided, warn the user and skip it. If no valid agents remain, inform the user and list available agents.

## Scope

Determine what to review based on the scope portion of arguments:

- **If scope is empty**: Review changed files only. Run `git diff --name-only HEAD` and `git diff HEAD`.
- **If scope is "full"**: Review the entire repository.
- **If scope is a path/glob**: Review only matching files.

## Execution

For each requested agent, perform its full review using the same criteria as the individual agent commands:

- **security**: Secrets detection, vulnerability patterns, enterprise security posture
- **quality**: Code smells, anti-patterns, maintainability
- **architecture**: Design patterns, scalability, coupling/cohesion
- **devops**: CI/CD patterns, IaC hygiene, container best practices
- **docs**: Technical writing, reproducibility, customer-facing quality
- **planning**: Project planning, progress tracking, stakeholder communication
- **testing**: Test coverage, test quality, edge cases, test strategy
- **compliance**: License compatibility, regulatory compliance, audit readiness
- **networking**: Routing/switching, firewall/ACL, DNS/load balancing, segmentation
- **virtualization**: Hypervisor config, HA/DRS, resource allocation, templates
- **storage**: RAID/replication, backup strategy, volume management
- **compute**: Server config, firmware lifecycle, capacity planning
- **accessibility**: WCAG 2.1 standards, ARIA patterns
- **observability**: Logging, log correlation, metrics, tracing, alerting/SLOs

## Output

Use the same unified report format as the `/review` orchestrator:

```
# Oversight Review Report (Selected Agents)

**Scope**: [diff / full / specific path]
**Files reviewed**: [count]
**Agents selected**: [list]

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | X     |
| HIGH     | X     |
| MEDIUM   | X     |
| LOW      | X     |
| INFO     | X     |

**Verdict**: [PASS / WARN / FAIL]

## Findings

[Findings sorted by severity, then by agent, using standard schema]

### [SEVERITY] Finding Title

- **Agent**: agent-name
- **File**: `path/to/file` (lines X-Y)
- **Category**: Category Name
- **Finding**: Description
- **Evidence**:
  ```
  code
  ```
- **Recommendation**: What to do
- **Reference**: Standard reference
```

### Deduplication

If multiple agents flag the same file+line range, consolidate into a single finding with the highest severity.

### No Issues

If no issues found: state "No issues found" with the scope and agents that were run.
