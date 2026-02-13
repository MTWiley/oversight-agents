# Stakeholder Communication Reference

This reference provides detailed checkpoints for evaluating how well a project communicates status, progress, releases, and plans to its stakeholders. It complements the inline criteria in `review-planning.md` with expanded definitions, severity guidance, stakeholder analysis, and templates.

The `planning-reviewer` agent uses this file as a lookup when evaluating communication artifacts. Severity levels follow `criteria/shared/severity-levels.md`.

---

## 1. Progress Visibility

### 1.1 Status at a Glance

**Checkpoint**: A stakeholder should be able to determine the current project status within two minutes. This requires a summary view -- not a deep-dive document, but a dashboard or top-of-document summary that answers: "Is this project on track, at risk, or behind?"

**Severity guidance**:

| Context | Severity |
|---|---|
| No project status summary exists anywhere | HIGH |
| Status exists but requires reading 10+ pages to understand current state | MEDIUM |
| Status summary exists but is stale (> 30 days without update) | MEDIUM |
| Clear, current status summary at the top of the primary planning document | INFO (positive) |

**Example**:

Problem -- status buried and inaccessible:
```markdown
# Project Plan

## Architecture Decisions
(500 lines of architecture discussion)

## Original Requirements
(200 lines of requirements)

## Task List
(150 tasks with mixed completion states, no summary)

## Meeting Notes
(300 lines of meeting notes from 6 months of meetings)

## Current Status
Somewhere around here, 1200 lines in, there might be a status paragraph.
```

Better -- status at a glance:
```markdown
# Project Plan

## Status Summary

| Metric | Value |
|--------|-------|
| **Overall Status** | At Risk |
| **Phase** | Phase 2 of 4 (Core API) |
| **Target Date** | April 30, 2025 |
| **Progress** | 12 of 20 tasks complete (60%) |
| **On Schedule?** | Behind by ~1 week |
| **Active Blockers** | 1 (external API credentials pending) |
| **Open Risks** | 2 (see Risk Register) |
| **Last Updated** | 2025-02-10 |

### Key Updates (this week)
- Completed authentication endpoints (ahead of schedule)
- Blocked on Stripe sandbox access; escalated to account manager
- Added rate limiting to scope (previously Phase 3); timeline impact: +3 days
```

---

### 1.2 Dashboard Views

**Checkpoint**: For multi-phase or multi-workstream projects, a visual progress indicator helps stakeholders understand where the project stands without reading every task. This can be a progress bar, a phase diagram, a checklist with completion percentages, or a table of workstream statuses.

**Severity guidance**:

| Context | Severity |
|---|---|
| Multi-phase project with no overview of phase completion | MEDIUM |
| Overview exists but is not maintained | LOW |
| Clear dashboard with phase and workstream status | INFO (positive) |

**Example**:

```markdown
## Project Dashboard

### Phase Progress
| Phase | Status | Progress | Target |
|-------|--------|----------|--------|
| Phase 1: Foundation | Complete | 8/8 tasks | Jan 31 (met) |
| Phase 2: Core API | In Progress | 12/20 tasks | Apr 30 |
| Phase 3: Frontend | Not Started | 0/15 tasks | Jun 30 |
| Phase 4: Launch | Not Started | 0/10 tasks | Jul 31 |

### Workstream Status
| Workstream | Owner | Status | Notes |
|-----------|-------|--------|-------|
| Backend API | @backend-team | On Track | 3 endpoints remaining |
| Authentication | @security-team | Complete | Passed security review |
| Infrastructure | @platform-team | At Risk | K8s cluster provisioning delayed |
| Documentation | @docs-team | Behind | Blocked on API stability |
```

---

### 1.3 Blocker and Risk Visibility

**Checkpoint**: Blockers (issues preventing progress) and risks (issues that may prevent future progress) must be visible to stakeholders. They should not be buried in task descriptions or meeting notes. A dedicated section with owner, escalation path, and expected resolution date allows stakeholders to help unblock the project.

**Severity guidance**:

| Context | Severity |
|---|---|
| Active blocker preventing progress with no documentation or visibility | HIGH |
| Blockers documented but not in an easily findable location | MEDIUM |
| Blockers documented with owners but no escalation path or target resolution | LOW |
| Blocker tracker with owners, escalation paths, and resolution targets | INFO (positive) |

**Example**:

Problem -- blockers invisible:
```markdown
## Tasks
- [ ] Integrate payment API
  <!-- need API keys from vendor, asked 3 weeks ago, no response -->
```

Better:
```markdown
## Active Blockers

| # | Blocker | Blocking | Owner | Escalation | Since | Target Resolution |
|---|---------|----------|-------|------------|-------|-------------------|
| B1 | Stripe sandbox API credentials not received | Payment integration (Phase 2, Task 7) | @backend-lead | Account manager (Jane Doe, jane@stripe.com); escalate to VP Sales if no response by Feb 15 | Jan 25 | Feb 12 |
| B2 | Security team capacity for audit | Production deployment (Phase 3) | @security-lead | CTO (copied on request Feb 5) | Feb 1 | Feb 28 |

## Current Risks

| # | Risk | Likelihood | Impact | Status | Mitigation Update |
|---|------|-----------|--------|--------|-------------------|
| R1 | Load testing reveals performance regression | Medium | High | Monitoring | Baseline benchmarks established; automated perf tests in CI |
| R2 | New EU data residency regulation | Low | High | New (Feb 10) | Legal team reviewing; may require infrastructure changes by Q3 |
```

---

### 1.4 External Dependency Tracking

**Checkpoint**: Dependencies on teams, vendors, or systems outside the project's direct control must be tracked with enough detail that stakeholders understand the dependency chain and can intervene if needed.

**Severity guidance**:

| Context | Severity |
|---|---|
| Critical-path external dependency with no tracking or communication | HIGH |
| External dependencies listed but without status updates | MEDIUM |
| External dependencies tracked with status but no fallback plans | LOW |
| Comprehensive external dependency tracking with status, contacts, and fallbacks | INFO (positive) |

**Example**:

Problem:
```markdown
## Dependencies
- Need something from the network team
- Waiting on a vendor
```

Better:
```markdown
## External Dependencies

| Dependency | External Team/Vendor | Contact | Status | Needed By | Fallback |
|-----------|---------------------|---------|--------|-----------|----------|
| VPN tunnel config | Network Ops | @net-ops (ticket NET-4521) | Approved; provisioning Feb 20 | Mar 1 | Use staging VPN; delay production deploy 1 week |
| SSO IdP metadata | Customer (Acme Corp) | security@acme.com | Requested Feb 1; reminder sent Feb 8 | Feb 28 | Provide test IdP for development; SSO testing delays to UAT |
| Compliance sign-off | Legal/Compliance | @compliance-team | Review in progress | Mar 15 | Submitted Jan 30; 6-week SLA; on track |
| Design assets | Design team | @design-lead | First draft received; feedback sent | Feb 20 | Build with wireframes; iterate post-launch |

**Last updated**: Feb 10, 2025
**Next review**: Feb 17, 2025 (weekly sync)
```

---

## 2. Release Documentation

### 2.1 Audience-Appropriate Release Notes

**Checkpoint**: Release notes should be written for the people who will read them: end users, operators, or integrating developers. Internal implementation details (refactored a class, renamed a variable) should not appear in user-facing release notes. Conversely, developer-facing release notes for a library should include API changes and migration details.

**Severity guidance**:

| Context | Severity |
|---|---|
| Released software with no release notes at all | HIGH |
| Release notes are just a list of commit messages (developer-internal) for a user-facing product | MEDIUM |
| Release notes exist but do not match the target audience | MEDIUM |
| Release notes are appropriate for the audience and explain impact | INFO (positive) |

**Example**:

Problem -- internal commit messages as user-facing release notes:
```markdown
## Release 3.2.0

- Refactored UserService to use dependency injection
- Updated webpack config for tree-shaking
- Fixed lint warnings in auth module
- Renamed `fetchData` to `loadData` internally
- Bumped lodash from 4.17.20 to 4.17.21
- Merged PR #456: Add search
```

Better -- user-facing release notes:
```markdown
## What's New in 3.2.0 (February 2025)

### New Features
- **Search**: You can now search across all your documents from the navigation bar.
  Type any keyword and results appear instantly. Supports exact phrase matching
  with quotes. [Learn more](docs/features/search.md)

### Improvements
- Page load time reduced by 40% on content-heavy pages
- Search results now highlight matching terms

### Bug Fixes
- Fixed an issue where uploaded images would not display on Safari 17+
- Fixed incorrect date formatting for users in UTC+12 and UTC+13 timezones

### Security
- Updated a dependency to address a moderate-severity vulnerability (CVE-2025-XXXXX).
  No user action required; the fix is applied automatically.

### Coming Soon
- Document sharing and collaboration (in beta; [sign up for early access](link))
```

---

### 2.2 Impact Explanation

**Checkpoint**: Each change in the release notes should explain what it means for the reader. "Changed the session timeout" is less useful than "Sessions now expire after 8 hours of inactivity (previously 30 minutes). You will need to log in less frequently."

**Severity guidance**:

| Context | Severity |
|---|---|
| Breaking change with no explanation of user impact | HIGH |
| Changes listed without context on what they mean for users | MEDIUM |
| Changes explain both what changed and why it matters | INFO (positive) |

**Example**:

Problem:
```markdown
### Changed
- Default timeout is now 60 seconds
```

Better:
```markdown
### Changed
- **API request timeout increased from 10 seconds to 60 seconds**:
  This affects all API calls. If your application relies on the previous 10-second
  timeout for fail-fast behavior, update your client configuration to set an explicit
  timeout. The default was increased to accommodate large data export requests that
  previously timed out for accounts with more than 50,000 records.
```

---

### 2.3 Upgrade Instructions

**Checkpoint**: When a release contains breaking changes, the release notes must include upgrade instructions or link to a migration guide. Users should know exactly what to change in their code, configuration, or processes to adopt the new version.

**Severity guidance**:

| Context | Severity |
|---|---|
| Breaking change released with no upgrade instructions | HIGH |
| Upgrade instructions exist but are incomplete (cover some changes but not all) | MEDIUM |
| Upgrade instructions linked but not inline (link is to migration guide) | LOW |
| Clear, step-by-step upgrade instructions covering all breaking changes | INFO (positive) |

**Example**:

Problem:
```markdown
## 4.0.0

### Changed
- New configuration format
- API authentication changed
- Database schema updated
```

Better:
```markdown
## 4.0.0 - Upgrade Guide

### Step 1: Update Configuration File

The configuration format changed from INI to YAML.

**Before** (config.ini):
```ini
[database]
host = localhost
port = 5432
name = myapp
```

**After** (config.yaml):
```yaml
database:
  host: localhost
  port: 5432
  name: myapp
```

An automated migration tool is available:
```bash
bin/migrate-config config.ini > config.yaml
```

### Step 2: Update API Authentication

API key authentication now uses Bearer tokens instead of query parameters.

**Before**:
```bash
curl "https://api.example.com/users?api_key=YOUR_KEY"
```

**After**:
```bash
curl -H "Authorization: Bearer YOUR_KEY" "https://api.example.com/users"
```

### Step 3: Run Database Migration

```bash
bin/db-migrate --version 4.0.0
```

This migration adds two columns and one index. Expected duration on a database
with 1M records: approximately 30 seconds. The migration is reversible:
```bash
bin/db-migrate --rollback --version 4.0.0
```

### Step 4: Verify

After upgrading, run the health check to verify:
```bash
curl https://your-instance.example.com/health
# Expected: {"status": "ok", "version": "4.0.0"}
```
```

---

### 2.4 Known Issues

**Checkpoint**: Release notes should document known issues in the release: bugs that were discovered but not fixed before release, limitations, compatibility issues, or workarounds. This prevents users from reporting known issues and reduces support burden.

**Severity guidance**:

| Context | Severity |
|---|---|
| Known critical bug shipped with no mention in release notes | HIGH |
| Known non-critical issues not documented | LOW |
| Known issues documented with workarounds and fix timeline | INFO (positive) |

**Example**:

```markdown
### Known Issues

| Issue | Severity | Workaround | Fix Target |
|-------|----------|------------|------------|
| CSV export fails for files > 100MB | Medium | Export in smaller date ranges; or use the API endpoint `/v2/export` which handles large files | v3.2.1 (next patch) |
| Dark mode: charts use light-mode colors | Low | No workaround; cosmetic only | v3.3.0 |
| Safari 16: file upload progress bar does not animate | Low | Upload completes successfully; only the visual indicator is affected | Under investigation |
```

---

## 3. Roadmap Communication

### 3.1 Forward-Looking Plans

**Checkpoint**: Stakeholders need to know what is coming next, not just what was delivered. A roadmap or forward-looking plan -- even a rough one -- enables stakeholders to plan their own work, provide early feedback, and set expectations with their own stakeholders.

**Severity guidance**:

| Context | Severity |
|---|---|
| Actively developed project with no roadmap or forward-looking plan | HIGH |
| Roadmap exists but has not been updated in > 3 months | MEDIUM |
| Roadmap exists and is current | INFO (positive) |

**Example**:

Problem:
```markdown
# Project Documentation

(Everything is about the current release. No indication of what comes next.
 Users have no way to plan for future changes.)
```

Better:
```markdown
## Roadmap

### Q1 2025 (current)
- User authentication and authorization (in progress)
- Public API v2 with rate limiting (in progress)
- Admin dashboard MVP (planned, starting March)

### Q2 2025
- Multi-tenancy support
- Audit logging and compliance reports
- Self-service API key management

### Q3 2025 (tentative)
- OAuth2/OIDC federation
- Webhook support for event notifications
- Geographic data residency options

### Future (no timeline)
- GraphQL API
- Mobile SDK
- On-premises deployment option

**Note**: This roadmap reflects current priorities and is subject to change.
Items in Q3 and Future are directional, not commitments. We welcome feedback
on priorities via [GitHub Discussions](link) or [feedback@example.com](mailto:feedback@example.com).
```

---

### 3.2 Timeline Realism

**Checkpoint**: Roadmap timelines should be realistic based on past velocity and available capacity. Overcommitted roadmaps that repeatedly slip erode stakeholder trust. It is better to underpromise than to miss every deadline.

**Severity guidance**:

| Context | Severity |
|---|---|
| Roadmap promises more work than capacity allows (verifiable from past velocity) | HIGH |
| Roadmap has no dates (perpetually "upcoming") | MEDIUM |
| Timelines marked as tentative with appropriate caveats | LOW |
| Realistic timelines with track record of delivery | INFO (positive) |

**Detection signals**:

- Previous roadmap items were consistently delivered late
- Current quarter's scope exceeds what was delivered in similar past periods
- No buffer time between phases
- Dependencies on unconfirmed external resources
- Team size assumed but not confirmed

**Example**:

Problem:
```markdown
## Q1 Roadmap (2 engineers, 13 weeks)
- Complete authentication rewrite (8 weeks estimated)
- Build analytics pipeline (6 weeks estimated)
- Implement multi-tenancy (6 weeks estimated)
- Redesign admin UI (4 weeks estimated)
Total: 24 weeks of work. Available: 26 engineer-weeks gross, ~16 net.
```

Better:
```markdown
## Q1 Roadmap (2 engineers, 13 weeks)

**Net capacity**: 16 engineer-weeks (after PTO, on-call, overhead)
**Planned**: 14 engineer-weeks (88% utilization; 12% buffer)

| Item | Estimate | Confidence | Status |
|------|----------|------------|--------|
| Authentication rewrite | 8 weeks | High | In scope |
| Analytics pipeline | 6 weeks | Medium | In scope |
| **Total committed** | **14 weeks** | | |
| Multi-tenancy | 6 weeks | Low | Deferred to Q2 |
| Admin UI redesign | 4 weeks | Low | Deferred to Q2 |

**Note**: Multi-tenancy and admin UI are deferred based on capacity analysis.
If the authentication rewrite completes early, we will pull multi-tenancy
forward.
```

---

### 3.3 Breaking Change Advance Notice

**Checkpoint**: Breaking changes should be communicated well in advance of the release that contains them. Stakeholders need time to prepare. The advance notice period depends on the audience: internal consumers may need 2-4 weeks; external API consumers may need 3-6 months. Surprise breaking changes are the fastest way to lose stakeholder trust.

**Severity guidance**:

| Context | Severity |
|---|---|
| Breaking change shipped with no advance notice to consumers | HIGH |
| Advance notice given but insufficient lead time | MEDIUM |
| Breaking changes communicated in roadmap with timeline | INFO (positive) |

**Example**:

Problem -- surprise breaking change:
```markdown
## v3.0.0 Release Notes (today)
- Authentication endpoint changed from /auth/login to /v3/auth/token
  (Oh, and all your existing integrations are now broken. Surprise!)
```

Better -- advance notice:
```markdown
## Upcoming Breaking Changes (v3.0.0, planned for Q2 2025)

The following breaking changes are planned for v3.0.0. We are publishing
this notice now (Q1 2025) to give all consumers at least 3 months to prepare.

### 1. Authentication Endpoint Change
**Current**: POST /auth/login
**New**: POST /v3/auth/token
**Reason**: Aligning with OAuth2 token endpoint conventions
**Migration**: Update your authentication client to use the new endpoint.
The old endpoint will return 301 redirects for 90 days after v3.0.0 release.
**Action required by**: June 30, 2025

### 2. User ID Format Change
**Current**: Integer IDs (e.g., 12345)
**New**: UUID format (e.g., 550e8400-e29b-41d4-a716-446655440000)
**Reason**: Support for distributed systems and cross-region replication
**Migration**: See [migration guide](docs/migration-v3.md#user-ids)
**Action required by**: June 30, 2025

**Questions?** Contact api-support@example.com or open a
[GitHub Discussion](link).
```

---

### 3.4 Deprecation Policy

**Checkpoint**: Projects should have a clear deprecation policy that defines how features, APIs, or behaviors are deprecated and eventually removed. Without a policy, deprecation is ad-hoc, inconsistent, and surprising.

**Severity guidance**:

| Context | Severity |
|---|---|
| Features removed without any deprecation period | HIGH |
| Features deprecated but no removal timeline communicated | MEDIUM |
| Deprecation policy documented and consistently followed | INFO (positive) |

**Example deprecation policy**:

```markdown
## Deprecation Policy

### Process
1. **Announce**: Feature is marked as deprecated in release notes, API docs,
   and runtime warnings. The replacement (if any) is documented.
2. **Warn**: Deprecated features emit runtime warnings (log level WARN) for
   at least 2 minor releases or 6 months, whichever is longer.
3. **Remove**: Feature is removed in the next major version after the warning
   period. Removal is documented in the changelog with the "Removed" category.

### Timeline
- **Minor release APIs**: Minimum 2 minor versions between deprecation and removal
- **Major features**: Minimum 6 months between deprecation announcement and removal
- **Configuration options**: Minimum 1 major version with backward compatibility

### Communication
- Deprecation announcements are included in release notes
- Upcoming removals are listed in the roadmap
- Breaking change advance notice is published at least 3 months before removal
- Affected users are notified directly when possible (email, in-app notification)

### Exceptions
- Security vulnerabilities may bypass the deprecation timeline
- Emergency removals are communicated immediately with explanation
```

---

## 4. Reporting Templates

### 4.1 Status Report Structure

**Checkpoint**: Status reports should follow a consistent structure that stakeholders can rely on. Ad-hoc status reports in different formats force readers to hunt for information.

**Severity**: LOW for format inconsistency. MEDIUM if the lack of structure causes miscommunication or missed blockers.

**Recommended status report template**:

```markdown
# Status Report: [Project Name]
**Period**: [Start Date] to [End Date]
**Author**: [Name]
**Distribution**: [Stakeholder list]

## Executive Summary
[2-3 sentence summary: overall status, key achievement, biggest risk]

## Overall Status: [On Track | At Risk | Behind | Blocked]

## Key Metrics
| Metric | Current | Target | Trend |
|--------|---------|--------|-------|
| Tasks complete | 12/20 (60%) | 20/20 by Apr 30 | On track |
| Test coverage | 78% | 80% | Improving |
| Open blockers | 1 | 0 | Stable |
| Open risks | 2 | -- | +1 new this period |

## Accomplishments (this period)
- [Accomplishment 1]: [brief description, impact]
- [Accomplishment 2]: [brief description, impact]

## In Progress
| Item | Owner | Expected Completion | Status |
|------|-------|-------------------|--------|
| [Item 1] | [Owner] | [Date] | On track |
| [Item 2] | [Owner] | [Date] | At risk (reason) |

## Blockers
| Blocker | Impact | Owner | Escalation | Target Resolution |
|---------|--------|-------|------------|-------------------|
| [Blocker] | [What it blocks] | [Owner] | [Escalation path] | [Date] |

## Risks
| Risk | Likelihood | Impact | Mitigation | Change |
|------|-----------|--------|------------|--------|
| [Risk 1] | [L/M/H] | [L/M/H] | [Plan] | [New/Unchanged/Increased] |

## Decisions Needed
- [Decision 1]: [Context and options. Needed by: date]

## Next Period Focus
- [Priority 1]
- [Priority 2]
- [Priority 3]
```

---

### 4.2 Executive Summary Format

**Checkpoint**: For executive stakeholders, the full status report is too detailed. An executive summary should be a single paragraph (3-5 sentences) or a single-page view that covers: overall status, key achievements, top risk or blocker, and any decision needed from leadership.

**Severity**: LOW if absent. MEDIUM if executives are making decisions without adequate project visibility.

**Example**:

Problem -- sending the full 5-page status report to the CTO:
```
(5 pages of task-level detail that no executive will read)
```

Better -- executive summary:
```markdown
## Executive Summary

**Project Alpha is At Risk.** We completed the authentication module on schedule
and API development is 60% complete. However, we are blocked on Stripe API
credentials (requested 3 weeks ago; escalated to account manager). If not
resolved by Feb 15, the payment integration will slip by 2 weeks, pushing the
launch from April 30 to mid-May. **Decision needed**: Should we engage a
secondary payment provider as a contingency? Estimated effort for parallel
integration: 1 engineer-week.
```

---

### 4.3 Technical Detail Level

**Checkpoint**: Status reports and release notes should provide the right level of technical detail for their audience. A report to the engineering team can include API signatures and code snippets. A report to product managers should focus on features and business impact. A report to executives should focus on timelines, risks, and decisions.

**Severity**: LOW for minor audience mismatch. MEDIUM if the wrong detail level causes misunderstanding or missed information.

**Detail level guide**:

| Audience | Appropriate Detail | Avoid |
|---|---|---|
| Engineering team | API changes, code examples, technical trade-offs, dependency updates | Business strategy, revenue projections |
| Product managers | Feature descriptions, user impact, timeline changes, scope adjustments | Implementation details, code snippets, infrastructure config |
| Executives | Project status, risks, decisions needed, timeline, budget impact | Task-level details, technical implementation, individual contributor names |
| Operations | Deployment changes, configuration changes, monitoring updates, runbook links | Feature descriptions, business justification |
| Customers | User-facing changes, upgrade instructions, known issues, deprecation timeline | Internal team names, infrastructure details, security vulnerability specifics |

---

## 5. Stakeholder Types and Information Needs

### 5.1 Stakeholder Analysis

**Checkpoint**: Different stakeholders need different information at different frequencies. Projects that communicate the same way to all stakeholders either overwhelm some with irrelevant detail or leave others without critical information.

**Severity guidance**:

| Context | Severity |
|---|---|
| No stakeholder identification or communication plan for a multi-team project | HIGH |
| Stakeholders identified but communication is ad-hoc and inconsistent | MEDIUM |
| Regular communication to identified stakeholders at appropriate detail levels | INFO (positive) |

**Stakeholder communication matrix**:

| Stakeholder | Needs to Know | Frequency | Format | Channel |
|---|---|---|---|---|
| **Executives** | Overall status, timeline, risks, decisions needed, budget impact | Biweekly or monthly | Executive summary (1 paragraph) | Email, Slack, or meeting |
| **Product Managers** | Feature progress, scope changes, user-facing impacts, timeline changes | Weekly | Feature-focused status report | Standup, Jira/Linear, Slack |
| **Engineering Team** | Technical details, architecture decisions, blockers, code review context | Daily/continuous | Standup, ADRs, PR descriptions | Slack, standups, code review |
| **Operations/SRE** | Deployment schedule, infrastructure changes, monitoring impacts, runbook updates | Per deployment + weekly | Deployment checklist, change log | Deployment channel, runbook updates |
| **Customers (external)** | Release notes, upgrade instructions, deprecation notices, known issues | Per release + quarterly roadmap | Release notes, blog posts, email | Product blog, email, changelog page |
| **QA/Testing** | Feature specifications, test scope, known issues, timeline for test readiness | Per feature + weekly | Test plan, feature spec | Jira/Linear, Slack, test management tool |
| **Security/Compliance** | Security-relevant changes, compliance impacts, audit artifacts | Per release + quarterly | Security changelog, compliance checklist | Security review channel, audit log |
| **Support** | User-facing changes, known issues, FAQ updates, escalation paths | Pre-release (advance notice) | Internal release notes, FAQ | Support wiki, training session |

---

### 5.2 Information Needs by Stakeholder Type

#### Executives

**What they need**:
- Is the project on track for the committed date?
- What are the top risks to delivery?
- Are there decisions that need executive-level input?
- Is the project within budget?
- What is the business impact of any delays?

**What they do NOT need**:
- Individual task status
- Technical implementation details
- Code-level findings
- Day-to-day team dynamics

**Communication anti-pattern**:
```
Subject: Weekly Status Update

We refactored the UserRepository to use the Unit of Work pattern and migrated
3 of 47 database tables from MySQL to PostgreSQL. The CI pipeline now runs
in 12 minutes instead of 15. We fixed a race condition in the session manager
by implementing optimistic locking with retry. Sprint velocity was 34 points.
```

**Better for executives**:
```
Subject: Project Alpha - Status: On Track

The database migration is 15% complete and on schedule. All migrated tables
are passing validation. No blockers. One risk: the vendor has not confirmed
the API changes needed for Phase 3 (due in 6 weeks). We're following up
this week. No decisions needed at this time.
```

---

#### Product Managers

**What they need**:
- Which features are done, in progress, or blocked?
- What changed from the original plan (scope changes, timeline shifts)?
- What is the user impact of each change?
- When will specific features be ready for user testing or release?
- What trade-offs need product decisions?

**What they do NOT need**:
- Database schema details
- Infrastructure configuration
- Code review findings
- Build pipeline optimization

---

#### Developers

**What they need**:
- Architectural decisions and their rationale
- API changes, code conventions, breaking changes
- Technical debt decisions and refactoring plans
- CI/CD changes that affect their workflow
- Code review context for PRs
- Dependency updates and their implications

**What they do NOT need**:
- Business strategy or revenue projections
- Executive-level summaries
- Marketing language about features

---

#### Operations / SRE

**What they need**:
- What is changing in the next deployment?
- Are there infrastructure changes (new services, configuration changes, resource requirements)?
- Are runbooks updated for new functionality?
- Are monitoring and alerting updated?
- What is the rollback plan?
- What are the expected performance impacts?

**What they do NOT need**:
- Feature descriptions or business justification
- Code-level implementation details
- Sprint planning artifacts

---

#### Customers (External)

**What they need**:
- What new features are available?
- What changed that affects their usage?
- How to upgrade (if applicable)?
- Known issues and workarounds
- What is coming next (high level)?
- Advance notice of breaking changes

**What they must NOT receive**:
- Internal team names or organizational details
- Specific vulnerability details before patches are available
- Infrastructure architecture details
- Internal decision-making context
- Unfiltered technical jargon without explanation

---

### 5.3 Communication Frequency

**Checkpoint**: Communication frequency should match stakeholder needs and project tempo. Over-communication causes alert fatigue; under-communication causes surprises.

**Severity guidance**:

| Context | Severity |
|---|---|
| Key stakeholders not informed of significant project changes (e.g., missed deadline, scope change) | HIGH |
| Stakeholders informed but infrequently (quarterly for a weekly-changing project) | MEDIUM |
| Communication frequency appropriate for the project tempo and stakeholder role | INFO (positive) |

**Frequency guidelines**:

| Project Phase | Internal Stakeholders | External Stakeholders |
|---|---|---|
| Active development | Weekly status updates | Per release |
| Pre-launch (2-4 weeks before) | Daily standups + weekly report | Advance notice of launch date and changes |
| Stable / maintenance | Biweekly or monthly | Per release + quarterly roadmap |
| Incident / crisis | Real-time updates until resolved | Status page updates; post-incident report |

---

## Communication Anti-Patterns

### Anti-Pattern 1: The Information Black Hole

**Description**: No status updates, no changelog, no roadmap. Stakeholders have no idea what the project is doing.

**Signals**:
- No `STATUS.md`, `CHANGELOG.md`, or `ROADMAP.md` in the repository
- No regular status communication to any stakeholder
- Questions about project status met with "check the code"
- Last update was months ago

**Severity**: HIGH for actively developed projects with stakeholders. MEDIUM for internal tools with a single developer.

---

### Anti-Pattern 2: The Firehose

**Description**: Every commit, every standup note, every code review comment is forwarded to all stakeholders. Volume overwhelms signal.

**Signals**:
- Executives receiving 50+ notification emails per day from the project
- All stakeholders on the same Slack channel with no differentiation
- Status reports that are 20+ pages with task-level detail

**Severity**: LOW. This is a quality issue, not a structural deficiency. Recommend tiered communication.

---

### Anti-Pattern 3: The Optimism Report

**Description**: Status is always "on track" and risks are always "low" until the project misses its deadline. Bad news is never communicated upward.

**Signals**:
- Status has been "90% complete" for weeks
- Risk register has not changed since project kickoff despite emerging issues
- Blockers are not reported until they have already caused a delay
- Post-mortems reveal that risks were known weeks before they materialized

**Severity**: HIGH. This is actively harmful -- it prevents stakeholders from taking corrective action.

---

### Anti-Pattern 4: The Surprise Breaking Change

**Description**: Breaking changes are communicated in the release notes on the day of release, giving consumers zero time to prepare.

**Signals**:
- Changelog contains `**BREAKING**` tags with no advance notice
- No deprecation period before removal
- Customer support tickets spike after releases

**Severity**: HIGH for external-facing APIs. MEDIUM for internal APIs with known consumers.

---

## Quick-Reference Summary

| Area | Checkpoint | Severity |
|---|---|---|
| Progress Visibility | No project status summary exists | HIGH |
| Progress Visibility | Status requires deep reading to understand | MEDIUM |
| Progress Visibility | Status summary stale (> 30 days) | MEDIUM |
| Progress Visibility | Multi-phase project with no overview | MEDIUM |
| Progress Visibility | Active blocker not documented or visible | HIGH |
| Progress Visibility | Critical external dependency untracked | HIGH |
| Release Docs | Released software with no release notes | HIGH |
| Release Docs | Release notes are raw commit messages for user-facing product | MEDIUM |
| Release Docs | Release notes do not match target audience | MEDIUM |
| Release Docs | Breaking change with no impact explanation | HIGH |
| Release Docs | Breaking change with no upgrade instructions | HIGH |
| Release Docs | Known critical bug shipped with no mention | HIGH |
| Roadmap | Active project with no forward-looking plan | HIGH |
| Roadmap | Roadmap stale (> 3 months without update) | MEDIUM |
| Roadmap | Roadmap promises more than capacity allows | HIGH |
| Roadmap | Roadmap has no dates (perpetually "upcoming") | MEDIUM |
| Roadmap | Breaking change shipped with no advance notice | HIGH |
| Roadmap | Features removed without deprecation period | HIGH |
| Reporting | No stakeholder identification for multi-team project | HIGH |
| Reporting | Communication ad-hoc and inconsistent | MEDIUM |
| Reporting | Status always "on track" despite emerging issues | HIGH |
| Reporting | Key stakeholders not informed of significant changes | HIGH |
| Reporting | All stakeholders get same detail level (firehose or starvation) | LOW-MEDIUM |
| Reporting | Appropriate tiered communication to identified stakeholders | INFO (positive) |
