# Oversight Agents

This repository contains reusable Claude Code review agents installed via `--add-dir`.

## Developing on This Repo

- Slash commands live in `.claude/commands/` and are the primary user interface
- Each command file is self-contained with inlined review criteria
- Criteria reference docs live in `criteria/` organized by domain
- The `$ARGUMENTS` variable controls review scope: empty = diff, `full` = repo-wide, path = specific files
- All findings must follow the schema in `criteria/shared/finding-schema.md`
- Severity levels: CRITICAL, HIGH, MEDIUM, LOW, INFO (see `criteria/shared/severity-levels.md`)

## Testing Changes

To test a command change, install this repo into a target project and invoke the slash command:

```bash
cd /path/to/test-project
claude --add-dir ~/git/oversight-agents
# Then use /review-security, /review, etc.
```

## Conventions

- Command files use markdown with structured sections (Scope, Review Criteria, Output)
- Keep criteria actionable and specific - avoid vague guidance
- Enterprise context: assume production environments with compliance requirements
- Domain specialists (networking, virtualization, storage, compute) should operate at expert depth
