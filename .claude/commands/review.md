# Oversight Review - Orchestrator

You are the oversight review orchestrator. Your job is to auto-detect which review agents are relevant for the current project and changes, then run each relevant review sequentially, and produce a unified report.

## Scope

Determine what to review based on `$ARGUMENTS`:

- **If `$ARGUMENTS` is empty**: Review changed files only. Run `git diff --name-only HEAD` to get the list of changed files. If there are no changes, check `git diff --name-only HEAD~1` for the most recent commit. If there are still no changes, inform the user and stop.
- **If `$ARGUMENTS` is "full"**: Review the entire repository. Use `find` or `git ls-files` to enumerate all tracked files.
- **If `$ARGUMENTS` is a path or glob**: Review only matching files.

Store the list of files and the full diff content for use by each agent pass.

## Agent Detection

Based on the files in scope, determine which agents to activate. Check for a `.oversight.yml` config file first for overrides.

### Config Override (if `.oversight.yml` exists)

```yaml
agents:
  always: [security, quality]     # Always run these
  disabled: [accessibility]       # Never run these
```

Agents listed in `always` run regardless of file detection. Agents in `disabled` are skipped even if detected.

### File-Pattern Detection Matrix

Scan the file list and activate agents based on these patterns:

| Pattern | Agent |
|---------|-------|
| **Always active** | security |
| `*.py`, `*.js`, `*.ts`, `*.go`, `*.java`, `*.rb`, `*.rs`, `*.c`, `*.cpp`, `*.cs`, `*.php`, `*.swift`, `*.kt` | quality |
| `*.tf`, `*.tfvars`, `terraform/`, `*.yaml`/`*.yml` (with k8s/helm content), `Dockerfile*`, `docker-compose*`, `.github/workflows/*`, `Jenkinsfile`, `*.groovy` (pipeline), `.gitlab-ci.yml`, `ansible/`, `*.playbook.yml` | devops |
| Any source code files (same as quality list) | architecture |
| `*.html`, `*.jsx`, `*.tsx`, `*.vue`, `*.svelte` | accessibility (Phase 2) |
| `*.md`, `docs/`, `README*`, `CHANGELOG*` | documentation (Phase 2) |
| `package.json`, `go.mod`, `requirements.txt`, `Gemfile`, `Cargo.toml`, `pom.xml`, `*.csproj`, `composer.json` | compliance (Phase 2) |
| `*test*`, `*spec*`, `__tests__/`, `test/`, `tests/` | testing (Phase 2) |
| Network device configs (files containing `router bgp`, `router ospf`, `access-list`, `ip route`, `interface` keywords; Ansible network playbooks with `ios_*`/`nxos_*`/`junos_*` modules; NAPALM/Nornir inventory) | networking (Phase 2) |
| `*.vmx`, `*.ovf`, `*.ova`, vSphere/Proxmox/Hyper-V configs | virtualization (Phase 2) |
| ZFS/LVM/RAID configs, `*.zpool`, storage-related YAML | storage (Phase 2) |
| BIOS/IPMI/iDRAC/iLO configs, server hardware manifests | compute (Phase 2) |

### Currently Available Agents (Phase 1)

Only invoke these agents — others are not yet implemented:

1. **security** — Always runs
2. **quality** — Runs when source code files are present
3. **architecture** — Runs when source code files are present
4. **devops** — Runs when CI/CD, IaC, or container files are present

## Execution

For each activated agent, perform its full review. Use the exact same criteria and approach as the individual agent commands:

### Security Review Pass
Follow the complete security review criteria: secrets detection, vulnerability patterns, and enterprise security posture. Evaluate every file in scope.

### Quality Review Pass
Follow the complete quality review criteria: code smells, anti-patterns, and maintainability. Evaluate source code files in scope.

### Architecture Review Pass
Follow the complete architecture review criteria: design patterns, scalability, and coupling/cohesion. Evaluate source code and configuration files in scope.

### DevOps Review Pass
Follow the complete DevOps review criteria: CI/CD patterns, IaC hygiene, and container best practices. Evaluate infrastructure and pipeline files in scope.

## Output

### Unified Report Format

```
# Oversight Review Report

**Scope**: [diff / full / specific path]
**Files reviewed**: [count]
**Agents activated**: [list]

## Summary

| Severity | Count |
|----------|-------|
| CRITICAL | X     |
| HIGH     | X     |
| MEDIUM   | X     |
| LOW      | X     |
| INFO     | X     |

**Verdict**: [PASS — no HIGH or CRITICAL findings / WARN — HIGH findings present / FAIL — CRITICAL findings present]

## Findings

[All findings sorted by severity (CRITICAL first), then by agent]

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

[Repeat for each finding]

## Agents Not Activated

[List agents that were not activated and why, so the user knows what wasn't checked]
```

### Deduplication

If multiple agents flag the same file+line range for overlapping concerns, consolidate into a single finding with the highest severity and note which agents flagged it.

### No Issues

If no issues are found across all agents:

```
# Oversight Review Report

**Scope**: [scope]
**Files reviewed**: [count]
**Agents activated**: [list]

No issues found. All activated agents reviewed the code and found no findings to report.
```
