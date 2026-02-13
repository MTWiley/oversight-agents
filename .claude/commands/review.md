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
| `*.html`, `*.jsx`, `*.tsx`, `*.vue`, `*.svelte` | accessibility |
| `*.md`, `docs/`, `README*`, `CHANGELOG*` | documentation |
| `PLAN.md`, `ROADMAP.md`, `CHANGELOG*`, `adr/`, `decisions/`, `VERSION` | planning |
| `package.json`, `go.mod`, `requirements.txt`, `Gemfile`, `Cargo.toml`, `pom.xml`, `*.csproj`, `composer.json`, `LICENSE*` | compliance |
| `*test*`, `*spec*`, `__tests__/`, `test/`, `tests/`, `jest.config.*`, `pytest.ini`, `vitest.config.*` | testing |
| Network device configs (files containing `router bgp`, `router ospf`, `access-list`, `ip route`, `interface` keywords; Ansible network playbooks with `ios_*`/`nxos_*`/`junos_*` modules; NAPALM/Nornir inventory) | networking |
| `*.vmx`, `*.ovf`, `*.ova`, files containing `vcenter`/`esxi`/`vsphere`/`proxmox`/`libvirt`/`hypervisor` keywords; Terraform `vsphere_*`/`proxmox_*` resources; Packer templates | virtualization |
| ZFS/LVM/RAID configs (files containing `zpool`/`zfs`/`mdadm`/`lvcreate`/`pvcreate`), Kubernetes `PersistentVolume`/`StorageClass`, cloud storage configs | storage |
| BMC/IPMI/iDRAC/iLO configs (files containing `ipmi`/`idrac`/`ilo`/`redfish`/`bios`/`firmware` keywords), server hardware manifests, provisioning templates (kickstart, preseed, autoinstall) | compute |
| Files containing logging/metrics/tracing libraries (`structlog`/`zerolog`/`zap`/`pino`/`winston`/`prometheus`/`opentelemetry`), monitoring configs (`prometheus.yml`, `alertmanager.yml`, `grafana/`), health check endpoints | observability |

### Available Agents

All agents are implemented and available:

**Core agents (always considered):**
1. **security** — Always runs
2. **quality** — Runs when source code files are present
3. **architecture** — Runs when source code files are present
4. **devops** — Runs when CI/CD, IaC, or container files are present

**Domain agents (activated by detection):**
5. **documentation** — Runs when documentation files are present
6. **planning** — Runs when planning/project management files are present
7. **testing** — Runs when test files or test config are present
8. **compliance** — Runs when package manifests or license files are present
9. **networking** — Runs when network configurations are detected
10. **virtualization** — Runs when hypervisor configs are detected
11. **storage** — Runs when storage configurations are detected
12. **compute** — Runs when server/hardware configs are detected
13. **accessibility** — Runs when UI/frontend files are present
14. **observability** — Runs when logging/metrics/tracing code is detected

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

### Documentation Review Pass
Follow the complete documentation review criteria: technical writing quality, reproducibility, and customer-facing quality. Evaluate documentation files in scope.

### Planning Review Pass
Follow the complete planning review criteria: project planning, progress tracking, and stakeholder communication. Evaluate planning artifacts in scope.

### Testing Review Pass
Follow the complete testing review criteria: test coverage, test quality, edge cases, and test strategy. Evaluate test files and source files in scope.

### Compliance Review Pass
Follow the complete compliance review criteria: license compatibility, regulatory compliance, and audit readiness. Evaluate package manifests, license files, and related artifacts in scope.

### Networking Review Pass
Follow the complete networking review criteria: routing/switching, firewall/ACL, DNS/load balancing, and segmentation. Evaluate network configurations and automation in scope.

### Virtualization Review Pass
Follow the complete virtualization review criteria: hypervisor configuration, HA/DRS, resource allocation, and template management. Evaluate virtualization configs in scope.

### Storage Review Pass
Follow the complete storage review criteria: RAID/replication, backup strategy, and volume management. Evaluate storage configurations in scope.

### Compute Review Pass
Follow the complete compute review criteria: server configuration, firmware lifecycle, and capacity planning. Evaluate server/hardware configs in scope.

### Accessibility Review Pass
Follow the complete accessibility review criteria: WCAG 2.1 standards and ARIA patterns. Evaluate UI/frontend files in scope.

### Observability Review Pass
Follow the complete observability review criteria: logging standards, log correlation, metrics instrumentation, distributed tracing, and alerting/SLOs. Evaluate source code and monitoring configs in scope.

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
