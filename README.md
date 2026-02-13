# Oversight Agents

Reusable Claude Code agents for automated code review across security, quality, architecture, DevOps, and infrastructure domains.

## Quick Start

```bash
# Clone the repo
git clone <repo-url> ~/tools/oversight-agents

# Use in any project
cd /path/to/your-project
claude --add-dir ~/tools/oversight-agents
```

Then invoke any review command:

```
/review                    # Auto-detect relevant agents and review changes
/review-security           # Security-focused review of changed files
/review-security full      # Security review of entire repo
/review-quality            # Code quality review
/review-architecture       # Architecture review
/review-devops             # DevOps/platform review
/review-pick security,quality   # Run specific agents
```

## Available Agents

### Phase 1 (Available Now)

| Command | Scope |
|---------|-------|
| `/review` | Orchestrator - auto-detects and runs relevant agents |
| `/review-security` | Secrets, vulnerabilities, enterprise security posture |
| `/review-quality` | Code smells, anti-patterns, maintainability |
| `/review-architecture` | Design patterns, scalability, coupling |
| `/review-devops` | CI/CD, IaC, containers, config hygiene |
| `/review-pick` | Run a named subset of agents |
| `/review-configure` | View/initialize project configuration |

### Phase 2 (Planned)

| Command | Scope |
|---------|-------|
| `/review-docs` | Technical writing, reproducibility, completeness |
| `/review-planning` | Project planning, milestone tracking |
| `/review-testing` | Coverage, test quality, edge cases |
| `/review-compliance` | Licenses, regulatory, audit readiness |
| `/review-networking` | Routing, ACLs, firewalls, BGP/OSPF, DNS |
| `/review-virtualization` | Hypervisor configs, HA/DRS, templates |
| `/review-storage` | RAID, replication, backup, volumes |
| `/review-compute` | Server configs, firmware, capacity |
| `/review-accessibility` | WCAG, ARIA patterns |

## Review Scope

All agents default to reviewing only changed files (`git diff HEAD`). Override with arguments:

```
/review-security              # Changed files only (default)
/review-security full         # Entire repository
/review-security src/auth/    # Specific path or glob
```

## Output Format

Findings follow a standard schema with severity levels:

| Level | Meaning | Action |
|-------|---------|--------|
| CRITICAL | Active vulnerability, data loss risk | Must fix before merge |
| HIGH | Significant security/quality gap | Should fix before merge |
| MEDIUM | Best practice violation, tech debt | Fix in current sprint |
| LOW | Minor improvement opportunity | Fix when convenient |
| INFO | Observation, suggestion | No action required |

Each finding includes: severity, file location, category, evidence, recommendation, and reference.

## Project Configuration

Create an `.oversight.yml` in your project root to customize agent behavior:

```yaml
project:
  type: networking          # Helps auto-detection

agents:
  always: [security]        # Always run these agents
  disabled: [accessibility] # Never run these agents
  config:
    security:
      severity-threshold: medium
      custom-rules:
        - "No SNMP v1/v2c - require v3"
```

Run `/review-configure init` to generate a starter config, or `/review-configure show` to display current settings.

## Installation Methods

### Primary: `--add-dir` (Recommended)

```bash
claude --add-dir ~/tools/oversight-agents
```

No files copied into your project. Always uses latest version.

### Alternative: Shell Alias

```bash
# Add to ~/.bashrc or ~/.zshrc
alias claude-review='claude --add-dir ~/tools/oversight-agents'

# Then use:
cd /path/to/project
claude-review
```

## Architecture

- **Slash commands** (`.claude/commands/`) are the primary entry points
- Each command is self-contained with inlined review criteria
- **Criteria reference docs** (`criteria/`) maintain the canonical checklists
- **Templates** (`templates/`) define output formatting
- **Config** (`config/`) provides defaults and schema for `.oversight.yml`

See `PLAN.md` for the full implementation roadmap.
