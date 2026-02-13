# Project Planning Reference

This reference provides detailed checkpoints for evaluating project planning artifacts. It complements the inline criteria in `review-planning.md` with expanded definitions, severity guidance, concrete examples, and templates for each checkpoint.

The `planning-reviewer` agent uses this file as a lookup when evaluating planning documents. Severity levels follow `criteria/shared/severity-levels.md`.

---

## 1. Goal Clarity

### 1.1 Measurable Objectives

**Checkpoint**: Project goals must be specific and measurable. Vague aspirational statements ("improve performance," "make it better") are not actionable goals. Every objective should answer: what will be true when this is done, and how will we verify it?

**Severity guidance**:

| Context | Severity |
|---|---|
| Project has no stated goals at all | HIGH |
| Goals exist but are entirely unmeasurable ("make it faster") | MEDIUM |
| Goals are mostly clear but one or two lack success criteria | LOW |
| All goals have measurable success criteria | INFO (positive) |

**Example**:

Problem -- vague goals:
```markdown
## Goals
- Improve the API
- Make the system more scalable
- Better error handling
- Clean up the code
```

Better -- measurable objectives:
```markdown
## Goals
- Reduce API p99 latency from 800ms to under 200ms for the /search endpoint
- Support 10,000 concurrent users (up from 1,000) without degrading response times beyond SLA thresholds
- Achieve zero unhandled exceptions in production (currently ~15/day in Sentry)
- Reduce cyclomatic complexity of modules in src/core/ to under 15 (currently 6 modules exceed 30)
```

---

### 1.2 Success Criteria

**Checkpoint**: Each goal must have explicit criteria that define when it is complete. Success criteria prevent scope creep and enable objective verification. Without them, "done" becomes a matter of opinion.

**Severity guidance**:

| Context | Severity |
|---|---|
| No success criteria for any goal | HIGH |
| Success criteria exist but are subjective ("feels faster") | MEDIUM |
| Success criteria are defined but not verifiable (no benchmark, no test) | LOW |
| Success criteria are defined, measurable, and linked to verification methods | INFO (positive) |

**Example**:

Problem:
```markdown
## Success Criteria
- The migration is done
- Performance is acceptable
- Users are happy
```

Better:
```markdown
## Success Criteria
- [ ] All 47 database tables migrated to PostgreSQL with row count verification
- [ ] Automated migration script runs end-to-end in under 30 minutes on staging
- [ ] All 312 existing integration tests pass against the new database
- [ ] p95 query latency remains under 100ms (measured via Datadog APM over 24h post-migration)
- [ ] Zero data loss verified by checksum comparison on critical tables (users, orders, transactions)
```

---

### 1.3 Scope Definition

**Checkpoint**: The plan must define what is included and what is excluded. A scope without boundaries expands indefinitely. Explicit scope prevents misaligned expectations between contributors, stakeholders, and reviewers.

**Severity**: MEDIUM when scope is partially defined (inclusions but no exclusions). HIGH when there is no scope definition and the project has multiple contributors or stakeholder visibility.

**Example**:

Problem:
```markdown
## Plan
We're going to rewrite the authentication system.
```

Better:
```markdown
## Scope

### In Scope
- Replace password hashing from MD5 to bcrypt
- Add TOTP-based two-factor authentication
- Implement session token rotation on privilege escalation
- Update login/logout API endpoints to match OpenAPI spec v3

### Out of Scope (deferred to Phase 2)
- OAuth2/OIDC federation with external identity providers
- Passwordless authentication (WebAuthn/FIDO2)
- Account recovery flow redesign
- Admin user management UI
```

---

### 1.4 Non-Goals

**Checkpoint**: Explicitly stating what the project will *not* do is as important as stating what it will do. Non-goals prevent contributors from building features that are out of scope and prevent stakeholders from expecting deliverables that were never planned.

**Severity**: MEDIUM when non-goals are absent for a project with broad scope. LOW for narrowly scoped projects where scope boundaries are obvious.

**Example**:

Problem -- no non-goals documented, leading to scope creep:
```markdown
## Project: API Rate Limiting
Add rate limiting to the public API.
```

Better:
```markdown
## Project: API Rate Limiting

### Goals
- Implement per-API-key rate limiting on all public endpoints
- Return 429 responses with Retry-After headers when limits are exceeded
- Provide rate limit status via response headers (X-RateLimit-Remaining, X-RateLimit-Reset)

### Non-Goals
- Per-endpoint rate limit customization (all endpoints share a single limit per key in v1)
- Rate limiting for internal service-to-service calls
- Usage-based billing integration (tracked separately in the billing roadmap)
- IP-based rate limiting (API-key-based only for this phase)
- Rate limit management UI for customers (customers contact support to adjust limits)
```

---

## 2. Task Breakdown

### 2.1 Task Sizing

**Checkpoint**: Tasks should be small enough to complete in a single work session (ideally 1-3 days for implementation tasks, up to 1 week for research/spike tasks). Tasks that are too large are hard to estimate, hard to track, and hard to review. Tasks that are too granular create overhead without value.

**Severity guidance**:

| Context | Severity |
|---|---|
| Plan has only high-level goals with no task breakdown | HIGH |
| Tasks exist but are multi-week monoliths ("Implement the backend") | MEDIUM |
| Most tasks are well-sized; a few are too large or too small | LOW |
| Tasks are consistently well-sized with clear boundaries | INFO (positive) |

**Example**:

Problem -- monolithic tasks:
```markdown
## Tasks
- [ ] Build the frontend
- [ ] Build the backend
- [ ] Set up infrastructure
- [ ] Testing
```

Better -- right-sized tasks:
```markdown
## Phase 1 Tasks

### Backend API
- [ ] Define OpenAPI spec for /users endpoints (POST, GET, PATCH, DELETE)
- [ ] Implement user creation endpoint with input validation
- [ ] Implement user lookup by ID and by email
- [ ] Implement user update with partial patch support
- [ ] Implement soft-delete with cascading deactivation of sessions
- [ ] Add pagination to user listing endpoint (cursor-based)

### Data Layer
- [ ] Create users table migration with indexes on email and created_at
- [ ] Implement UserRepository with CRUD operations
- [ ] Add database connection pooling configuration
- [ ] Write seed data script for development environment

### Testing
- [ ] Unit tests for UserRepository (CRUD + edge cases)
- [ ] Integration tests for /users endpoints (happy path + error cases)
- [ ] Load test for user listing endpoint (target: 500 rps at p99 < 200ms)
```

---

### 2.2 Dependencies Between Tasks

**Checkpoint**: Tasks that depend on other tasks must have those dependencies documented. Undocumented dependencies cause blocked work, wasted effort on tasks that cannot proceed, and confusion about sequencing.

**Severity guidance**:

| Context | Severity |
|---|---|
| Tasks have obvious dependencies that are not documented, causing blocked work | HIGH |
| Dependencies exist but are informal ("we should do X first") | MEDIUM |
| Dependencies are documented but incomplete | LOW |
| Dependencies are clearly mapped with a logical execution order | INFO (positive) |

**Example**:

Problem -- no dependency tracking:
```markdown
## Tasks
- [ ] Write API documentation
- [ ] Deploy to production
- [ ] Implement authentication
- [ ] Set up CI/CD pipeline
- [ ] Write integration tests
```

Better -- dependencies explicit:
```markdown
## Tasks (with dependencies)

1. [ ] Set up CI/CD pipeline
2. [ ] Implement authentication
3. [ ] Write integration tests (depends on: #2)
4. [ ] Write API documentation (depends on: #2, #3 -- needs stable API)
5. [ ] Deploy to production (depends on: #1, #3, #4)

### Dependency Graph
```
CI/CD (#1) ──────────────────────────┐
Authentication (#2) ─┬─> Tests (#3) ─┼─> Deploy (#5)
                     └─> Docs (#4) ──┘
```
```

---

### 2.3 Ownership and Assignment

**Checkpoint**: Every task should have a clear ownership model. This does not require named individual assignment for small teams, but it does require knowing who is responsible for each work area and how unassigned tasks get picked up.

**Severity guidance**:

| Context | Severity |
|---|---|
| Multi-team project with no ownership model at all | HIGH |
| Ownership is assumed but not documented | MEDIUM |
| Ownership model exists but some tasks are orphaned | LOW |
| Clear ownership with defined escalation paths | INFO (positive) |

**Example**:

Problem -- unclear ownership on multi-team project:
```markdown
## Tasks
- [ ] Migrate database schema
- [ ] Update API clients
- [ ] Notify downstream consumers
- [ ] Update runbooks
```

Better:
```markdown
## Tasks

### Database Team (owner: @db-team)
- [ ] Migrate database schema
- [ ] Verify replication lag during migration
- [ ] Update connection strings in Vault

### API Team (owner: @api-team)
- [ ] Update API clients to use new schema
- [ ] Run backward compatibility tests

### Platform Team (owner: @platform-team)
- [ ] Notify downstream consumers 2 weeks before migration
- [ ] Update runbooks and monitoring dashboards

### Unassigned (to be picked up in sprint planning)
- [ ] Post-migration performance benchmarking
```

---

### 2.4 Estimation

**Checkpoint**: Tasks should have effort estimates. Estimates do not need to be precise, but they should exist to enable capacity planning, timeline projection, and progress measurement. Common formats include T-shirt sizes (S/M/L/XL), story points, or time ranges (1-2 days).

**Severity guidance**:

| Context | Severity |
|---|---|
| No estimation on any task for a time-bound project | HIGH |
| Estimates exist but are clearly unrealistic (100 tasks marked "1 day" each) | MEDIUM |
| Some tasks estimated, others not | LOW |
| Consistent estimation with documented assumptions | INFO (positive) |

**Example**:

Problem -- no estimates for a deadline-driven project:
```markdown
## Q1 Deliverables (due March 31)
- [ ] SSO integration
- [ ] Audit logging
- [ ] Data export API
- [ ] Admin dashboard
- [ ] Performance optimization
```

Better:
```markdown
## Q1 Deliverables (due March 31)

| Task | Estimate | Confidence | Notes |
|------|----------|------------|-------|
| SSO integration (SAML + OIDC) | 3 weeks | Medium | Depends on IdP sandbox access |
| Audit logging | 1 week | High | Pattern established in billing module |
| Data export API | 2 weeks | Medium | CSV + JSON; async for large datasets |
| Admin dashboard | 3 weeks | Low | Design not finalized; estimate may increase |
| Performance optimization | 2 weeks | Low | Scope depends on profiling results |

**Total estimated effort**: 11 weeks
**Available capacity**: 12 engineer-weeks (2 engineers x 6 weeks remaining)
**Buffer**: ~1 week (tight; consider descoping admin dashboard to Phase 2)
```

---

## 3. Prioritization

### 3.1 Priority Scheme

**Checkpoint**: Tasks should be prioritized using a consistent, explicit scheme. Common schemes include MoSCoW (Must/Should/Could/Won't), P0/P1/P2/P3, numbered priority, or a simple ordered list. The scheme should be documented so contributors understand why tasks are ordered the way they are.

**Severity guidance**:

| Context | Severity |
|---|---|
| No priority on any task for a project with limited resources and a deadline | HIGH |
| Priority exists but is inconsistent (everything is "high priority") | MEDIUM |
| Priority scheme is applied to most but not all tasks | LOW |
| Consistent priority scheme with documented rationale | INFO (positive) |

**Example**:

Problem -- everything is priority 1:
```markdown
## Tasks (all high priority)
- [P1] Add rate limiting
- [P1] Refactor database layer
- [P1] Fix CSS alignment on mobile
- [P1] Implement search
- [P1] Update dependencies
- [P1] Add dark mode
```

Better:
```markdown
## Prioritized Tasks

### P0 - Must ship (blocks launch)
- [ ] Add rate limiting (required for production readiness)
- [ ] Implement search (core user-facing feature)

### P1 - Should ship (significant value)
- [ ] Refactor database layer (tech debt blocking P0 performance targets)
- [ ] Update dependencies (3 HIGH CVEs in current versions)

### P2 - Nice to have (if time permits)
- [ ] Fix CSS alignment on mobile (cosmetic; workaround exists)
- [ ] Add dark mode (user request; not blocking adoption)
```

---

### 3.2 Critical Path Identification

**Checkpoint**: The critical path -- the longest sequence of dependent tasks that determines the minimum project duration -- should be identified. Tasks on the critical path cannot slip without delaying the project. Tasks off the critical path have float (slack time).

**Severity guidance**:

| Context | Severity |
|---|---|
| Deadline-driven project with no critical path analysis | HIGH |
| Critical path is implied but not documented | MEDIUM |
| Critical path identified but not updated as tasks complete | LOW |
| Critical path documented, maintained, and used for resource allocation | INFO (positive) |

**Example**:

Problem -- critical path not identified for a deadline project:
```markdown
## Launch Plan (target: April 15)
- Database migration
- API development
- Frontend integration
- Load testing
- Security audit
- Documentation
```

Better:
```markdown
## Launch Plan (target: April 15)

### Critical Path (determines launch date)
1. Database migration (2 weeks) - must complete before API work
2. API development (3 weeks) - depends on new schema
3. Load testing (1 week) - depends on API completion
4. Security audit (1 week) - depends on feature-complete API
**Critical path duration: 7 weeks (latest start: Feb 25)**

### Parallel Work (has float)
- Frontend integration (2 weeks) - can start after API spec is defined (week 3)
- Documentation (1 week) - can be written during load testing/audit
- Monitoring setup (3 days) - independent, can happen anytime
```

---

### 3.3 Quick Wins

**Checkpoint**: Tasks that deliver high value with low effort should be identified and prioritized early. Quick wins build momentum, demonstrate progress, and often unblock other work. Plans that defer all easy wins in favor of large, risky tasks miss the opportunity to show early value.

**Severity**: LOW. This is an optimization, not a structural deficiency.

**Example**:

Better:
```markdown
## Quick Wins (< 1 day effort, immediate impact)
- [ ] Enable gzip compression on API responses (5-line config change; ~60% bandwidth reduction)
- [ ] Add health check endpoint (required for load balancer; 30m implementation)
- [ ] Fix N+1 query on /orders listing (known issue; 2h fix; 10x performance improvement)
- [ ] Add request ID header for log correlation (simplifies debugging immediately)
```

---

## 4. Phase and Milestone Structure

### 4.1 Milestone Deliverables

**Checkpoint**: Each milestone or phase must define what it delivers. A milestone without deliverables is just a date on a calendar. Deliverables should be concrete, verifiable, and represent meaningful increments of value.

**Severity guidance**:

| Context | Severity |
|---|---|
| Project has no milestones or phases at all | HIGH |
| Milestones exist but deliverables are vague ("Phase 1 complete") | MEDIUM |
| Deliverables are defined for most milestones but some are unclear | LOW |
| Each milestone has concrete, verifiable deliverables | INFO (positive) |

**Example**:

Problem:
```markdown
## Milestones
- Phase 1: Foundation (March)
- Phase 2: Core Features (May)
- Phase 3: Polish (June)
- Phase 4: Launch (July)
```

Better:
```markdown
## Milestones

### Phase 1: Foundation (due March 31)
**Delivers**: Working development environment and data layer
- [ ] Database schema finalized and migrated
- [ ] Repository pattern implemented for all entities
- [ ] CI pipeline running lint + test on every PR
- [ ] Development environment reproducible via docker-compose up
**Definition of done**: New developer can clone, run, and execute all tests within 15 minutes

### Phase 2: Core API (due May 15)
**Delivers**: Feature-complete API ready for frontend integration
- [ ] All CRUD endpoints implemented per OpenAPI spec
- [ ] Authentication and authorization enforced on all endpoints
- [ ] Rate limiting active on public endpoints
- [ ] API documentation published and accurate
**Definition of done**: Frontend team can build against the API using only the published docs
```

---

### 4.2 Definition of Done

**Checkpoint**: Each phase or milestone needs an explicit definition of done (DoD). Without a DoD, "done" becomes subjective, leading to incomplete work being marked complete and accumulating hidden debt.

**Severity guidance**:

| Context | Severity |
|---|---|
| Milestones with no definition of done, marked complete despite incomplete work | HIGH |
| Definition of done exists but is not consistently applied | MEDIUM |
| Definition of done is informal or incomplete | LOW |
| Clear, measurable DoD for every milestone | INFO (positive) |

**Example**:

Problem -- "done" means different things to different people:
```markdown
## Phase 1: Complete ✅
(But tests are failing, no documentation, no code review on last 3 PRs)
```

Better:
```markdown
## Definition of Done (applies to all phases)

A phase is complete when ALL of the following are true:
- [ ] All tasks in the phase are merged to main
- [ ] All CI checks pass on main (lint, test, build, security scan)
- [ ] Test coverage for new code meets project threshold (>= 80%)
- [ ] API documentation is updated for any new or changed endpoints
- [ ] CHANGELOG.md is updated with entries for all user-facing changes
- [ ] No CRITICAL or HIGH findings from /review-security on the phase diff
- [ ] At least one team member (not the author) has reviewed all PRs
```

---

### 4.3 Logical Progression

**Checkpoint**: Phases should build on each other in a logical order. Later phases should depend on earlier phases, not the reverse. A plan that requires Phase 3 deliverables to complete Phase 2 work indicates poor structuring.

**Severity**: MEDIUM when phases have backward dependencies. LOW when ordering is suboptimal but not circular.

**Example**:

Problem -- circular dependency between phases:
```markdown
Phase 1: Build the API
Phase 2: Build the database layer   <-- API depends on this; should be Phase 1
Phase 3: Write tests for Phase 1    <-- tests should be concurrent, not deferred
```

Better:
```markdown
Phase 1: Data layer + CI foundation  (no dependencies)
Phase 2: API implementation          (depends on Phase 1 data layer)
Phase 3: Frontend integration        (depends on Phase 2 API)
Phase 4: Performance + hardening     (depends on Phase 2 + 3 for realistic load testing)
```

---

## 5. Risk Identification

### 5.1 Documented Risks

**Checkpoint**: Projects should identify known risks proactively. A risk register does not need to be formal or exhaustive, but it must exist. Common risk categories include technical risks (unproven technology, performance unknowns), dependency risks (external teams, third-party services), resource risks (key person dependency, capacity constraints), and schedule risks (aggressive timelines, external deadlines).

**Severity guidance**:

| Context | Severity |
|---|---|
| Deadline-driven project with zero documented risks | HIGH |
| Risks are acknowledged informally but not tracked | MEDIUM |
| Risk register exists but is stale or incomplete | LOW |
| Active risk register with regular updates | INFO (positive) |

**Example**:

Problem -- no risks documented for a complex migration:
```markdown
## Plan: Database Migration to PostgreSQL
(No risk section)
```

Better:
```markdown
## Risks

| # | Risk | Likelihood | Impact | Mitigation |
|---|------|-----------|--------|------------|
| R1 | Data loss during migration | Low | Critical | Dry-run migration on staging first; checksum verification; point-in-time backup before cutover |
| R2 | Application queries incompatible with PostgreSQL | Medium | High | Run full test suite against PostgreSQL in CI before migration; track pg-incompatible queries |
| R3 | Migration window exceeds maintenance window (4h) | Medium | Medium | Benchmark migration time on staging with production-scale data; prepare rollback script |
| R4 | Third-party ORM generates suboptimal PostgreSQL queries | Low | Medium | Profile top-20 queries post-migration; have query optimization sprint budgeted |
| R5 | Team lacks PostgreSQL operational experience | Medium | Medium | Schedule PostgreSQL administration training; engage DBA consultant for first month |
```

---

### 5.2 Mitigation Plans

**Checkpoint**: Each identified risk should have a mitigation strategy. Mitigation can be avoidance (eliminate the risk), reduction (lower likelihood or impact), transfer (insurance, SLA from vendor), or acceptance (acknowledge and monitor). Risks without mitigation are just worries.

**Severity**: MEDIUM when risks are identified but have no mitigation. LOW when mitigation exists but is vague ("we'll deal with it").

**Example**:

Problem:
```markdown
## Risks
- The vendor API might be slow
- We might not have enough time
- The new framework might have bugs
```

Better:
```markdown
## Risks and Mitigations

### R1: Vendor API latency exceeds SLA
**Likelihood**: Medium | **Impact**: High
**Mitigation**:
- Implement circuit breaker with 3s timeout and fallback to cached data
- Pre-negotiate dedicated API tier with vendor (contract amendment in progress)
- Build async processing queue so user requests are not blocked by vendor calls
**Owner**: @backend-team | **Status**: Circuit breaker implemented; vendor negotiation pending
```

---

### 5.3 Contingency Plans

**Checkpoint**: For high-impact risks, a contingency plan (what to do if the risk materializes) should exist alongside the mitigation plan (how to prevent it). Contingency answers "what is plan B?"

**Severity**: HIGH when a critical-path risk has no contingency. MEDIUM when contingencies are vague.

**Example**:

Problem:
```markdown
Risk: Cloud provider outage
Mitigation: Hope it doesn't happen
```

Better:
```markdown
### R3: Primary cloud region outage during launch week
**Likelihood**: Low | **Impact**: Critical

**Mitigation** (prevent/reduce):
- Multi-AZ deployment within primary region
- Database replication to secondary region (async, RPO < 5 minutes)
- Health check and auto-failover on load balancer

**Contingency** (if it happens):
1. Activate secondary region via DNS failover (automated, < 5 min)
2. Verify data consistency on secondary (runbook: docs/runbooks/region-failover.md)
3. Communicate status to customers via status page and email
4. Post-incident: evaluate whether to promote secondary to primary or fail back
**Last tested**: 2024-11-15 (quarterly failover drill)
```

---

## 6. Resource Planning

### 6.1 Skill Requirements

**Checkpoint**: The plan should identify what skills or expertise are needed and whether the team has them. Skill gaps discovered mid-project cause delays and quality issues.

**Severity guidance**:

| Context | Severity |
|---|---|
| Project requires expertise the team does not have, with no plan to acquire it | HIGH |
| Skill requirements not documented; gaps discovered during execution | MEDIUM |
| Skill requirements listed but gaps are unaddressed | LOW |
| Skills mapped, gaps identified, and acquisition plan in place | INFO (positive) |

---

### 6.2 Capacity Constraints

**Checkpoint**: The plan should account for available capacity (people, time, infrastructure). Plans that assume unlimited capacity are unrealistic. Overcommitted teams deliver nothing well.

**Severity guidance**:

| Context | Severity |
|---|---|
| Plan requires more capacity than available with no acknowledgment | HIGH |
| Capacity constraints acknowledged but not quantified | MEDIUM |
| Capacity is roughly estimated | LOW |
| Capacity planned with buffer for unknowns | INFO (positive) |

**Example**:

Problem:
```markdown
## Q1 Plan
- Rewrite authentication (4 weeks)
- Build analytics pipeline (6 weeks)
- Migrate to Kubernetes (8 weeks)
- Implement multi-tenancy (6 weeks)
Total: 24 engineer-weeks of work. Team: 2 engineers. Quarter: 12 weeks.
(No capacity analysis; 24 weeks of work for 24 engineer-weeks of capacity
 assumes zero meetings, bugs, on-call, or interruptions)
```

Better:
```markdown
## Q1 Capacity Analysis

**Team**: 2 engineers
**Calendar weeks**: 13 (Jan 6 - Mar 28)
**Gross capacity**: 26 engineer-weeks
**Deductions**:
- PTO/holidays: -3 engineer-weeks
- On-call rotation: -2 engineer-weeks
- Meetings/overhead: -3 engineer-weeks (estimated 15%)
- Bug fixes/support: -2 engineer-weeks (based on Q4 actuals)
**Net capacity**: 16 engineer-weeks

**Planned work**: 14 engineer-weeks (88% utilization)
**Buffer**: 2 engineer-weeks (12%) for unknowns

| Project | Estimate | Priority | Fits in Q1? |
|---------|----------|----------|-------------|
| Rewrite authentication | 4 weeks | P0 | Yes |
| Build analytics pipeline | 6 weeks | P1 | Yes |
| Migrate to Kubernetes | 4 weeks | P1 | Yes (descoped to Phase 1 only) |
| Implement multi-tenancy | 6 weeks | P2 | No - deferred to Q2 |
```

---

### 6.3 External Dependencies

**Checkpoint**: Dependencies on external teams, vendors, open source projects, or infrastructure provisioning should be explicitly documented with expected timelines and fallback plans. External dependencies are the most common source of unexpected delays.

**Severity guidance**:

| Context | Severity |
|---|---|
| Critical-path dependency on external team with no communication or timeline | HIGH |
| External dependencies listed but no expected dates or contacts | MEDIUM |
| Dependencies documented with contacts and dates but no fallback | LOW |
| Full external dependency tracking with owners, dates, fallbacks, and status | INFO (positive) |

**Example**:

Problem:
```markdown
## Plan
- Integrate with payments API (need access from vendor)
- Deploy to production (need VPN config from network team)
```

Better:
```markdown
## External Dependencies

| Dependency | Owner | Needed By | Status | Fallback |
|-----------|-------|-----------|--------|----------|
| Payment API sandbox credentials | Stripe (contact: acct-mgr@stripe.com) | Feb 15 | Requested Jan 20; follow-up scheduled Feb 5 | Mock API for development; delay integration testing |
| VPN tunnel to production | Network team (@net-ops, ticket NET-4521) | Mar 1 | Approved; provisioning scheduled Feb 20 | Deploy to staging for demo; production deploy shifts 1 week |
| Security audit approval | InfoSec team (@security, ticket SEC-892) | Mar 15 | Not yet requested | Submit by Feb 1 to allow 6-week review cycle |
| Design mockups for admin UI | Design team (@design) | Feb 28 | In progress; first draft expected Feb 10 | Build with wireframes; iterate on design post-launch |
```

---

## Planning Artifact Templates

### Well-Structured PLAN.md Template

```markdown
# Project Name

## Overview
One-paragraph description of what this project accomplishes and why it matters.

## Goals
- [ ] Goal 1 (measurable: specific metric or verifiable outcome)
- [ ] Goal 2
- [ ] Goal 3

## Non-Goals
- Item explicitly out of scope and why
- Item deferred to a future phase

## Success Criteria
- [ ] Criterion 1 (how to verify)
- [ ] Criterion 2

## Phases

### Phase 1: [Name] (target: [date])
**Delivers**: [what stakeholders get]

#### Tasks
- [ ] Task 1.1 [estimate] [owner]
- [ ] Task 1.2 [estimate] [owner] (depends on: 1.1)
- [ ] Task 1.3 [estimate] [owner]

**Definition of done**: [specific, verifiable criteria]

### Phase 2: [Name] (target: [date])
...

## Risks

| Risk | Likelihood | Impact | Mitigation | Owner |
|------|-----------|--------|------------|-------|
| ... | ... | ... | ... | ... |

## External Dependencies

| Dependency | Owner | Needed By | Status | Fallback |
|-----------|-------|-----------|--------|----------|
| ... | ... | ... | ... | ... |

## Capacity

**Available**: X engineer-weeks
**Planned**: Y engineer-weeks
**Buffer**: Z engineer-weeks (N%)

## Decision Log
See `docs/decisions/` for ADRs.
```

### Anti-Pattern: Vague TODO List

The following is an anti-pattern that this agent should flag:

```markdown
# TODO

- fix the thing
- auth stuff
- make it work on mobile
- talk to Dave about the API
- tests
- deployment ???
- clean up
- docs maybe
- ask about timeline
- the security thing from standup
```

**Why this is a problem**: No prioritization. No sizing. No ownership. No dependencies. No scope. No success criteria. No dates. This list cannot be tracked, reported on, or used for planning. It is a brain dump, not a plan.

**Findings to raise**: MEDIUM for the structural deficiency. Specific items like "fix the thing" should be flagged as LOW for vagueness. If this is the only planning artifact for a project with a deadline, escalate to HIGH.

---

## Quick-Reference Summary

| Area | Checkpoint | Severity |
|---|---|---|
| Goal Clarity | No stated goals at all | HIGH |
| Goal Clarity | Goals exist but unmeasurable | MEDIUM |
| Goal Clarity | Goals mostly clear, few lack criteria | LOW |
| Goal Clarity | No scope definition | MEDIUM-HIGH |
| Goal Clarity | No non-goals for broad-scope project | MEDIUM |
| Task Breakdown | No task breakdown, only high-level goals | HIGH |
| Task Breakdown | Tasks are multi-week monoliths | MEDIUM |
| Task Breakdown | Dependencies not documented, causing blocked work | HIGH |
| Task Breakdown | Dependencies informal or incomplete | MEDIUM-LOW |
| Task Breakdown | Multi-team project with no ownership model | HIGH |
| Task Breakdown | No estimation for deadline-driven project | HIGH |
| Task Breakdown | Estimates clearly unrealistic | MEDIUM |
| Prioritization | No priority for resource-constrained project | HIGH |
| Prioritization | Everything marked same priority | MEDIUM |
| Prioritization | Critical path not identified for deadline project | HIGH |
| Prioritization | Quick wins not identified | LOW |
| Milestones | No milestones or phases | HIGH |
| Milestones | Milestones without deliverables | MEDIUM |
| Milestones | No definition of done | MEDIUM-HIGH |
| Milestones | Backward dependencies between phases | MEDIUM |
| Risks | Zero documented risks for complex project | HIGH |
| Risks | Risks identified but no mitigation | MEDIUM |
| Risks | Critical-path risk with no contingency | HIGH |
| Resources | Project needs skills team lacks, no acquisition plan | HIGH |
| Resources | Plan exceeds available capacity, no acknowledgment | HIGH |
| Resources | Critical external dependency untracked | HIGH |
| Resources | External dependencies listed without dates or fallback | MEDIUM |
