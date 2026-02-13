# Oversight Agents - Implementation Plan

## Context

This repo provides a reusable set of Claude Code agents for project oversight. The agents are designed to be installed into any project (networking, virtualization, storage, compute, web, IaC, etc.) via `--add-dir` and invoked as slash commands during development. They review code changes for security issues, documentation gaps, quality problems, compliance concerns, and domain-specific best practices.

The user works across diverse infrastructure domains at CCIE-equivalent depth, so domain specialist agents are separated rather than combined into a single shallow "infrastructure" agent.

## Agent Roster (14 agents)

| # | Agent | Scope | Phase |
|---|-------|-------|-------|
| 1 | Security | Secrets detection, vulnerability patterns, enterprise security posture | 1 (Done) |
| 2 | Documentation | Technical writing, reproducibility, customer-facing completeness | 2 |
| 3 | Planning & Progress | Project planning, milestone tracking, progress documentation | 2 |
| 4 | Code Quality & Standards | Code smells, anti-patterns, consistency, maintainability | 1 (Done) |
| 5 | Testing & QA | Coverage, test quality, edge cases, test strategy | 2 |
| 6 | Compliance & Licensing | Dependency licenses, regulatory, audit readiness | 2 |
| 7 | Architecture Review | Design patterns, scalability, coupling, architectural consistency | 1 (Done) |
| 8 | DevOps/Platform (core) | CI/CD, IaC, containerization, config hygiene | 1 (Done) |
| 9 | Networking Specialist | Routing, ACLs, firewalls, BGP/OSPF, DNS, segmentation | 2 |
| 10 | Virtualization Specialist | Hypervisor configs, HA/DRS, resource allocation, templates | 2 |
| 11 | Storage Specialist | RAID, replication, tiering, backup, volume management | 2 |
| 12 | Compute Specialist | Server configs, firmware, capacity planning, bare-metal | 2 |
| 13 | Accessibility | WCAG, ARIA patterns (lowest priority) | 2 |
| 14 | Observability | Logging standards, log correlation, metrics instrumentation, distributed tracing, alerting | 2 |

## Repository Structure

```
oversight-agents/
├── CLAUDE.md                           # Repo-level context for developing ON this repo
├── README.md                           # Installation & usage guide
├── LICENSE
│
├── .claude/
│   ├── commands/                       # Slash commands (user entry points)
│   │   ├── review.md                   # /review - orchestrator, auto-detects relevant agents
│   │   ├── review-security.md          # /review-security
│   │   ├── review-docs.md              # /review-docs
│   │   ├── review-planning.md          # /review-planning
│   │   ├── review-quality.md           # /review-quality
│   │   ├── review-testing.md           # /review-testing
│   │   ├── review-compliance.md        # /review-compliance
│   │   ├── review-architecture.md      # /review-architecture
│   │   ├── review-devops.md            # /review-devops
│   │   ├── review-networking.md        # /review-networking
│   │   ├── review-virtualization.md    # /review-virtualization
│   │   ├── review-storage.md           # /review-storage
│   │   ├── review-compute.md           # /review-compute
│   │   ├── review-accessibility.md     # /review-accessibility
│   │   ├── review-pick.md              # /review-pick security,compliance [full]
│   │   └── review-configure.md         # /review-configure [init|show]
│   │
│   └── settings.json
│
├── criteria/                           # Review checklists (referenced by commands)
│   ├── shared/
│   │   ├── severity-levels.md          # CRITICAL/HIGH/MEDIUM/LOW/INFO definitions
│   │   └── finding-schema.md           # Standard finding format all agents use
│   ├── security/
│   │   ├── secrets-detection.md
│   │   ├── vulnerability-patterns.md
│   │   └── enterprise-posture.md
│   ├── documentation/
│   │   ├── technical-writing.md
│   │   ├── reproducibility.md
│   │   └── customer-facing.md
│   ├── quality/
│   │   ├── code-smells.md
│   │   ├── anti-patterns.md
│   │   └── maintainability.md
│   ├── testing/
│   │   ├── coverage-expectations.md
│   │   ├── test-quality.md
│   │   └── edge-cases.md
│   ├── compliance/
│   │   ├── license-compatibility.md
│   │   ├── regulatory.md
│   │   └── audit-readiness.md
│   ├── architecture/
│   │   ├── design-patterns.md
│   │   ├── scalability.md
│   │   └── coupling.md
│   ├── devops/
│   │   ├── cicd-patterns.md
│   │   ├── iac-hygiene.md
│   │   └── container-best-practices.md
│   ├── networking/
│   │   ├── routing-switching.md
│   │   ├── firewall-acl.md
│   │   ├── dns-lb.md
│   │   └── segmentation.md
│   ├── virtualization/
│   │   ├── hypervisor-config.md
│   │   ├── ha-drs.md
│   │   └── template-management.md
│   ├── storage/
│   │   ├── raid-replication.md
│   │   ├── backup-strategy.md
│   │   └── volume-management.md
│   ├── compute/
│   │   ├── server-config.md
│   │   ├── firmware-lifecycle.md
│   │   └── capacity-planning.md
│   └── accessibility/
│       ├── wcag-standards.md
│       └── aria-patterns.md
│
├── templates/                          # Output format templates
│   ├── report-full.md                  # Full structured report
│   ├── report-summary.md              # Executive summary
│   ├── finding-inline.md              # Inline annotation format
│   └── report-ci.md                   # GitHub PR comment format
│
├── adapters/                           # Non-slash-command invocation methods
│   ├── hooks/
│   │   ├── pre-commit.sh              # Runs security agent on staged files
│   │   ├── pre-push.sh                # Runs security + quality on branch diff
│   │   └── install-hooks.sh           # Installs hooks into target repo
│   ├── ci/
│   │   └── github-actions/
│   │       ├── oversight-review.yml   # Reusable GHA workflow
│   │       └── oversight-pr-review.yml
│   └── cli/
│       └── oversight-review.sh        # Standalone CLI wrapper (uses claude -p)
│
├── install/
│   ├── install.sh                     # Symlink-based install into target project
│   ├── uninstall.sh
│   └── update.sh
│
├── config/
│   ├── defaults.yml                   # Default agent configuration
│   └── schema.yml                     # Schema for .oversight.yml
│
├── examples/
│   ├── oversight.yml.networking       # Example config for networking project
│   ├── oversight.yml.webapp           # Example config for web app
│   ├── oversight.yml.infrastructure   # Example config for IaC project
│   └── oversight.yml.minimal          # Minimal starter config
│
└── docs/
    ├── architecture.md
    ├── agent-authoring.md
    ├── customization.md
    └── ci-integration.md
```

## Architecture Decisions

### Invocation: Slash Commands as Primary Entry Point

Each agent has a corresponding slash command in `.claude/commands/`. The user types `/review-security` and gets a security review. The `/review` orchestrator auto-detects relevant agents and runs them.

**Why slash commands first**: They're explicit, discoverable via tab-completion, support arguments (`$ARGUMENTS`), and work immediately with `--add-dir`. Git hooks and CI adapters come later as thin wrappers around the same agent logic.

### Installation: `--add-dir` (Primary)

```bash
# Clone once
git clone <repo-url> ~/tools/oversight-agents

# Use in any project
cd /path/to/my-project
claude --add-dir ~/tools/oversight-agents
```

No files are copied into the target project. Always uses the latest version. The commands, criteria, and templates are all available in Claude's context via the additional directory.

Alternative methods (symlinks, git submodule, shell alias) are documented for users who prefer them.

### Review Scope: Diff-Based Default

Commands default to reviewing only changed files (`git diff HEAD`). Pass `full` as an argument for a full repo scan, or pass a file path/glob to review specific files.

```
/review-security              # diff only (default)
/review-security full         # full repo scan
/review-security src/auth/    # specific directory
```

### Domain Detection: Auto + Manual Override

The `/review` orchestrator uses file-pattern heuristics to detect which specialist agents are relevant:

| File Patterns | Agent Activated |
|---|---|
| `*.tf`, `*.tfvars`, `terraform/` | devops-reviewer |
| `Dockerfile`, `docker-compose*` | devops-reviewer |
| `.github/workflows/*`, `Jenkinsfile` | devops-reviewer |
| `*.py`, `*.js`, `*.go`, etc. | quality-reviewer, testing-reviewer |
| `*.md`, `docs/` | documentation-reviewer |
| Network device configs, ACLs, firewall rules | networking-reviewer |
| `*.vmx`, `*.ovf`, vSphere/Proxmox configs | virtualization-reviewer |
| RAID/ZFS/LVM/SAN/NAS configs | storage-reviewer |
| BIOS/IPMI/iDRAC/iLO configs | compute-reviewer |
| `package.json`, `go.mod`, `requirements.txt` | compliance-reviewer |
| `*.html`, `*.jsx`, `*.tsx` | accessibility-reviewer |

Manual override via `.oversight.yml` in the target project:

```yaml
project:
  type: networking
agents:
  always: [security, networking]
  disabled: [accessibility]
  config:
    security:
      severity-threshold: medium
      custom-rules:
        - "No SNMP v1/v2c - require v3"
```

### Output Format: Report + Inline

Each finding follows a standard schema (defined in `criteria/shared/finding-schema.md`):

```markdown
### [HIGH] Hardcoded Database Password

- **Agent**: security-reviewer
- **File**: `src/config/database.py` (lines 42-45)
- **Category**: Hardcoded Credentials
- **Finding**: Database password is hardcoded in source code.
- **Evidence**:
  \`\`\`python
  DB_PASSWORD = "production_secret_123"
  \`\`\`
- **Recommendation**: Use environment variables or a secrets manager.
- **Reference**: OWASP A07:2021
```

The full report includes a summary table at the top with finding counts by severity. When running multiple agents, findings are deduplicated by file+line overlap and category similarity.

### Severity Levels

| Level | Meaning | Action |
|---|---|---|
| CRITICAL | Active vulnerability, data loss risk | Must fix before merge |
| HIGH | Significant security/quality gap | Should fix before merge |
| MEDIUM | Best practice violation, tech debt | Fix in current sprint |
| LOW | Minor improvement opportunity | Fix when convenient |
| INFO | Observation, suggestion | No action required |

## Slash Command Structure

Each command file follows this pattern:

```markdown
# [Agent Name] Review

You are a [domain] specialist reviewing code changes.

## Scope
- If $ARGUMENTS is empty: review changed files only (run `git diff --name-only HEAD` and `git diff HEAD`)
- If $ARGUMENTS is "full": review the entire repository
- If $ARGUMENTS is a path/glob: review only matching files

## Review Criteria
[Inline checklist specific to this agent domain]

## Output
Format findings using the standard schema. Include severity, file, line numbers,
evidence, recommendation, and reference for each finding.

Present a summary table at the top:
| Severity | Count |

If no issues found, state "No [domain] issues found" with the scope reviewed.
```

Criteria content is inlined directly in each command file to keep each command self-contained and portable.

## Implementation Phases

### Phase 1: MVP Foundation

**Goal**: Working slash commands for the 4 highest-impact agents + orchestrator.

**Status**: Complete

Files to create:

1. **Repo scaffolding**:
   - `CLAUDE.md` - repo context
   - `README.md` - installation & usage
   - Directory structure

2. **Shared foundations**:
   - `criteria/shared/severity-levels.md`
   - `criteria/shared/finding-schema.md`
   - `templates/report-full.md`
   - `templates/report-summary.md`

3. **4 MVP agents** (command + criteria for each):
   - Security (`review-security.md` + `criteria/security/*`)
   - Code Quality (`review-quality.md` + `criteria/quality/*`)
   - Architecture (`review-architecture.md` + `criteria/architecture/*`)
   - DevOps/Platform (`review-devops.md` + `criteria/devops/*`)

4. **Orchestrator**:
   - `review.md` - auto-detect and run relevant agents
   - `review-pick.md` - run named subset

5. **Configuration**:
   - `review-configure.md`
   - `config/defaults.yml`
   - `config/schema.yml`

**Verification**: Install into an existing project with `claude --add-dir ~/git/oversight-agents`, run `/review-security` on a real diff, confirm output matches the finding schema.

### Phase 2: Complete Agent Roster

**Goal**: All 14 agents operational.

**Status**: Not started (next up)

Files to create:

1. Remaining 10 agents (command + criteria for each):
   - Documentation, Planning, Testing, Compliance
   - Networking, Virtualization, Storage, Compute
   - Accessibility
   - Observability (new - logging, metrics, tracing, alerting)
2. Update orchestrator detection matrix
3. `templates/finding-inline.md`
4. Example configs in `examples/`

### Phase 3: Git Hooks & CI/CD

**Goal**: Automated invocation via hooks and GitHub Actions.

**Status**: Not started

Files to create:

1. `adapters/hooks/pre-commit.sh` - security agent on staged files
2. `adapters/hooks/pre-push.sh` - security + quality on branch diff
3. `adapters/hooks/install-hooks.sh`
4. `adapters/ci/github-actions/oversight-review.yml` - reusable workflow
5. `adapters/ci/github-actions/oversight-pr-review.yml` - PR-triggered
6. `templates/report-ci.md`

### Phase 4: CLI & Documentation

**Goal**: Standalone CLI and comprehensive docs.

**Status**: Not started

Files to create:

1. `adapters/cli/oversight-review.sh`
2. `install/install.sh`, `uninstall.sh`, `update.sh`
3. `docs/architecture.md`, `agent-authoring.md`, `customization.md`, `ci-integration.md`

### Phase 5: Iteration (Ongoing)

- Tune criteria based on false positive/negative rates from real usage
- Add language-specific vulnerability patterns
- Explore MCP server integration
- Add `/review-report` for historical trend reporting
