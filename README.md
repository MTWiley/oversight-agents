# Oversight Agents

Reusable Claude Code agents for automated code review across security, quality, architecture, DevOps, infrastructure, and operational domains. 14 specialized agents covering enterprise environments from application code to bare-metal compute.

## Installation

Clone this repo once to a permanent location on your machine:

```bash
git clone https://github.com/YOUR-ORG/oversight-agents.git ~/tools/oversight-agents
```

That's it. No dependencies, no build step, no package manager. The agents are plain Markdown files that Claude Code reads directly.

## Usage

### Using in any project

From inside any project, launch Claude Code with `--add-dir` pointing to your local clone:

```bash
cd ~/projects/my-webapp
claude --add-dir ~/tools/oversight-agents
```

This makes all `/review-*` slash commands available in that session without copying any files into your project. Your project stays clean — no dotfiles, no config, no dependencies added.

```
/review                          # Auto-detect relevant agents and review changes
/review full                     # Auto-detect and review entire repo
/review-security                 # Security-focused review of changed files
/review-security full            # Security review of entire repo
/review-pick security,quality    # Run specific agents
/review-pick networking,devops full  # Specific agents on full repo
```

### Using across multiple projects

The same clone works for every project. Just point `--add-dir` at it:

```bash
# Web app
cd ~/projects/my-webapp
claude --add-dir ~/tools/oversight-agents

# Network automation
cd ~/projects/network-configs
claude --add-dir ~/tools/oversight-agents

# Terraform infrastructure
cd ~/projects/infra-as-code
claude --add-dir ~/tools/oversight-agents
```

Each project gets the same agents. The orchestrator auto-detects which agents are relevant based on each project's file types — a web app activates security + quality + architecture + testing, while a network automation repo activates security + networking + devops.

### Recommended: shell alias

To avoid typing the `--add-dir` path every time, add an alias to your shell profile:

```bash
# Add to ~/.bashrc, ~/.zshrc, or ~/.config/fish/config.fish
alias claude-review='claude --add-dir ~/tools/oversight-agents'
```

Then use it anywhere:

```bash
cd ~/projects/anything
claude-review
# All /review-* commands are available
```

### Updating

To get the latest agents, pull the repo:

```bash
cd ~/tools/oversight-agents && git pull
```

The next time you run `claude --add-dir`, it picks up the updated agents automatically.

## Available Agents

### Orchestration

| Command | Purpose |
|---------|---------|
| `/review` | Auto-detects and runs all relevant agents, unified report |
| `/review-pick` | Run a named subset of agents |
| `/review-configure` | View/initialize project `.oversight.yml` |

### Core Agents

| Command | Scope |
|---------|-------|
| `/review-security` | Secrets, vulnerabilities, enterprise security posture |
| `/review-quality` | Code smells, anti-patterns, maintainability |
| `/review-architecture` | Design patterns, scalability, coupling |
| `/review-devops` | CI/CD, IaC, containers, config hygiene |

### Domain Agents

| Command | Scope |
|---------|-------|
| `/review-docs` | Technical writing, reproducibility, completeness |
| `/review-planning` | Project planning, milestone tracking, progress |
| `/review-testing` | Coverage, test quality, edge cases, test strategy |
| `/review-compliance` | Licenses, regulatory, audit readiness |
| `/review-observability` | Logging, metrics, tracing, alerting, SLOs |

### Infrastructure Specialists

| Command | Scope |
|---------|-------|
| `/review-networking` | Routing, ACLs, firewalls, BGP/OSPF, DNS, segmentation |
| `/review-virtualization` | Hypervisor configs, HA/DRS, resource allocation, templates |
| `/review-storage` | RAID, replication, backup strategy, volumes |
| `/review-compute` | Server configs, firmware lifecycle, capacity planning |
| `/review-accessibility` | WCAG 2.1, ARIA patterns, keyboard navigation |

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

## How It Works

- **No files are copied** into your project. `--add-dir` tells Claude Code to read the agents directory as additional context.
- **Agents auto-detect** which checks are relevant based on your project's file types. A Python web app gets security + quality + architecture + testing. A Cisco config repo gets security + networking.
- **Per-project config** is optional. Drop an `.oversight.yml` in any project to override agent selection, severity thresholds, or add custom rules. Run `/review-configure init` to generate one.
- **Findings follow a standard schema** across all agents, so output is consistent whether you run one agent or all fourteen.

## Architecture

- **Slash commands** (`.claude/commands/`) are the primary entry points
- Each command is self-contained with inlined review criteria
- **Criteria reference docs** (`criteria/`) maintain the canonical checklists
- **Templates** (`templates/`) define output formatting
- **Config** (`config/`) provides defaults and schema for `.oversight.yml`

See `PLAN.md` for the full implementation roadmap.
