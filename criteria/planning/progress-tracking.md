# Progress Tracking Reference

This reference provides detailed checkpoints for evaluating progress tracking artifacts: status documents, changelogs, version management, decision records, and velocity patterns. It complements the inline criteria in `review-planning.md` with expanded definitions, severity guidance, and concrete examples.

The `planning-reviewer` agent uses this file as a lookup when evaluating progress documentation. Severity levels follow `criteria/shared/severity-levels.md`.

---

## 1. Status Accuracy

### 1.1 Actual vs. Documented State

**Checkpoint**: The documented project status must reflect the actual state of the codebase. Status documents that claim "done" for unfinished work, or "in progress" for abandoned tasks, are actively misleading. The agent should cross-reference status claims against the codebase when possible.

**Severity guidance**:

| Context | Severity |
|---|---|
| Status shows "complete" for items that are clearly not implemented (verifiable against codebase) | CRITICAL |
| Status shows "in progress" for items with no recent commits or activity | MEDIUM |
| Status is accurate but not recently updated (timestamp > 30 days) | LOW |
| Status is current, accurate, and matches the codebase | INFO (positive) |

**How to verify**:

When reviewing a `PLAN.md`, `STATUS.md`, or similar document that marks tasks as complete, check:

1. Does the referenced feature/file exist in the codebase?
2. Does `git log --oneline --since="30 days ago"` show related activity for "in progress" items?
3. Are tests passing for the "complete" functionality?
4. Does the documented version match the actual version in `package.json`, `pyproject.toml`, or equivalent?

**Example**:

Problem -- status contradicts reality:
```markdown
## Phase 1 Status: Complete ✅

- [x] User authentication (JWT-based)
- [x] Role-based access control
- [x] API rate limiting
- [x] Audit logging
```

But in the codebase:
```python
# src/auth/middleware.py
def authenticate(request):
    # TODO: implement JWT validation
    return True  # Allow everything for now

# src/auth/rbac.py -- file does not exist
# src/middleware/rate_limit.py -- file does not exist
# src/logging/audit.py -- file contains only an empty class
```

**Findings to raise**: CRITICAL. The documented status actively misrepresents the project state. Stakeholders relying on this status will make incorrect decisions.

---

### 1.2 Stale Item Detection

**Checkpoint**: Items marked as "in progress" or "planned" should show evidence of activity. An item that has been "in progress" for months with no commits, no discussion, and no updates is effectively abandoned. Stale items clutter the plan and hide the true scope of remaining work.

**Severity guidance**:

| Context | Severity |
|---|---|
| Item marked "in progress" with no activity for > 90 days | MEDIUM |
| Item marked "in progress" with no activity for 30-90 days | LOW |
| Item marked "planned" with no start date and no recent discussion | LOW |
| Stale items are regularly triaged and either completed, deferred, or removed | INFO (positive) |

**Detection signals**:

- `git log --all --oneline --since="90 days ago" -- <relevant-paths>` returns no commits
- Issue/PR referenced in the plan is closed without merging, or has no activity
- The task description references outdated APIs, removed files, or superseded decisions

**Example**:

Problem:
```markdown
## In Progress
- [ ] Migrate to TypeScript (started Q2 2024)    <-- 6+ months, 3 files converted out of 200
- [ ] Implement caching layer (assigned to Dave)  <-- Dave left the company 4 months ago
- [ ] Add WebSocket support (spike complete)      <-- "spike" was 8 months ago; no follow-up
```

Better -- triaged and honest:
```markdown
## In Progress
- [ ] Migrate to TypeScript (started Q2 2024)
  **Status**: Stalled. 3/200 files converted. Needs decision: continue or abandon.
  **Action**: Discuss in next planning meeting. If no champion, move to Backlog.

## Backlog (not actively worked)
- [ ] Implement caching layer -- reassign; previous owner no longer on team
- [ ] Add WebSocket support -- spike results in docs/spikes/websocket.md; needs prioritization decision

## Dropped
- ~~GraphQL API~~ -- decided against in ADR-012; REST API meets all current needs
```

---

### 1.3 Verification Against Codebase

**Checkpoint**: When a plan or status document references specific deliverables, the agent should verify that those deliverables exist in the codebase. This is especially important for milestone completion claims.

**Severity guidance**:

| Context | Severity |
|---|---|
| Milestone marked complete but key deliverables are missing from the codebase | CRITICAL |
| Task marked complete but implementation is partial (stub, TODO, empty file) | HIGH |
| Task marked complete and implementation exists but is untested | MEDIUM |
| Task marked complete with implementation and tests | INFO (positive) |

**What to look for**:

```
# Signs of incomplete implementation
TODO|FIXME|HACK|XXX|STUB|PLACEHOLDER|NotImplementedError|pass  # in the "completed" code
raise NotImplementedError  # skeleton without implementation
return None  # placeholder return value
# empty function/class bodies
```

---

## 2. Changelog Quality

### 2.1 Changelog Format Compliance

**Checkpoint**: Changelogs should follow a consistent, recognized format. The most widely adopted standard is [Keep a Changelog](https://keepachangelog.com/). The format should be documented and applied consistently across releases.

**Severity guidance**:

| Context | Severity |
|---|---|
| Released project with no changelog at all | HIGH |
| Changelog exists but is an unstructured dump of commit messages | MEDIUM |
| Changelog follows a format but inconsistently | LOW |
| Changelog follows Keep a Changelog or equivalent consistently | INFO (positive) |

**Example**:

Problem -- raw commit log as changelog:
```markdown
# Changes

- fix bug
- update deps
- wip
- Merge branch 'feature/auth'
- fix tests
- actually fix tests
- revert "fix tests"
- properly fix tests this time
- bump version
```

Better -- Keep a Changelog format:
```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added
- OAuth2 authorization code flow with PKCE support (#142)
- Rate limiting on public API endpoints with configurable per-key limits (#156)

### Fixed
- Session tokens not invalidated on password change (#148)

## [2.1.0] - 2025-01-15

### Added
- User profile API endpoints (GET, PATCH) (#130)
- Pagination support on /users listing endpoint (#134)

### Changed
- Default session timeout increased from 30 minutes to 2 hours (#138)

### Deprecated
- `/v1/users/search` endpoint; use `/v2/users?q=` instead. Will be removed in v3.0. (#135)

### Fixed
- Database connection leak under high concurrency (#131)
- Incorrect timezone handling in audit log timestamps (#133)

### Security
- Updated jsonwebtoken to 9.0.2 to address CVE-2024-XXXXX (#132)

## [2.0.0] - 2024-11-01

### Changed
- **BREAKING**: Authentication endpoint moved from /auth/login to /v2/auth/token (#120)
- **BREAKING**: User ID format changed from integer to UUID (#118)

### Removed
- **BREAKING**: Removed deprecated /v1/auth/* endpoints (#121)

### Migration
- See [Migration Guide](docs/migration-v2.md) for upgrade instructions

[Unreleased]: https://github.com/org/repo/compare/v2.1.0...HEAD
[2.1.0]: https://github.com/org/repo/compare/v2.0.0...v2.1.0
[2.0.0]: https://github.com/org/repo/compare/v1.5.0...v2.0.0
```

---

### 2.2 Entry Categorization

**Checkpoint**: Changelog entries should be categorized according to the type of change. Keep a Changelog defines six categories: Added, Changed, Deprecated, Removed, Fixed, Security. Entries in the wrong category mislead readers about the nature of changes.

**Severity guidance**:

| Context | Severity |
|---|---|
| Entries are uncategorized (flat list) | MEDIUM |
| Entries are categorized but some are in the wrong category (e.g., a bug fix under "Added") | LOW |
| Categories are used consistently and correctly | INFO (positive) |

**Category definitions**:

| Category | Use When |
|---|---|
| Added | New features or capabilities |
| Changed | Changes to existing functionality |
| Deprecated | Features that will be removed in a future release |
| Removed | Features removed in this release |
| Fixed | Bug fixes |
| Security | Vulnerability fixes or security improvements |

**Common mistakes**:

- Listing a bug fix under "Changed" (it should be under "Fixed")
- Listing a removed feature under "Changed" (it should be under "Removed")
- Not using the "Security" category for CVE fixes (mixing them into "Fixed")
- Listing internal refactoring under "Changed" when there is no user-visible change (omit it or add as a note)

---

### 2.3 Breaking Change Documentation

**Checkpoint**: Breaking changes must be clearly marked in the changelog. A breaking change is any change that requires consumers to modify their code, configuration, or processes. Missing or unclear breaking change documentation causes upgrade failures and erodes trust.

**Severity guidance**:

| Context | Severity |
|---|---|
| Breaking change shipped with no changelog mention | HIGH |
| Breaking change in changelog but not marked as breaking | MEDIUM |
| Breaking change marked but no migration guidance | MEDIUM |
| Breaking change clearly marked with migration guide | INFO (positive) |

**How to identify breaking changes**:

- Removed or renamed public API endpoints, functions, or classes
- Changed parameter types, names, or order in public APIs
- Changed return type or response schema
- Changed configuration file format or required new configuration
- Changed database schema requiring migration
- Changed minimum supported language/runtime version
- Removed support for a previously supported platform

**Example**:

Problem:
```markdown
## [3.0.0] - 2025-02-01

### Changed
- Updated user API
- Modified configuration format
```

Better:
```markdown
## [3.0.0] - 2025-02-01

### Changed
- **BREAKING**: User API response now returns `created_at` as ISO 8601 string
  instead of Unix timestamp. Update any code that parses this field.
  See [migration guide](docs/migration-v3.md#timestamp-format).
- **BREAKING**: Configuration file format changed from INI to YAML.
  Run `bin/migrate-config` to convert existing configuration files.
  See [migration guide](docs/migration-v3.md#config-format).

### Removed
- **BREAKING**: Removed `/v1/` API endpoints deprecated in v2.0.
  All clients must use `/v2/` endpoints.

### Migration
For a complete list of breaking changes and upgrade instructions,
see [Migration Guide v2 to v3](docs/migration-v3.md).
```

---

### 2.4 Issue and PR Linking

**Checkpoint**: Changelog entries should link to the issue, pull request, or commit that implements the change. Links provide traceability and allow readers to find implementation details, discussion context, and related changes.

**Severity**: LOW when links are absent. This is a quality improvement, not a structural deficiency.

**Example**:

Problem:
```markdown
### Fixed
- Fixed race condition in session management
```

Better:
```markdown
### Fixed
- Fixed race condition in session management that could cause duplicate sessions
  under high concurrency ([#245](https://github.com/org/repo/pull/245))
```

---

### 2.5 Conventional Commits Compliance

**Checkpoint**: If the project uses [Conventional Commits](https://www.conventionalcommits.org/), commit messages should follow the specification. Conventional Commits enable automated changelog generation, semantic version bumping, and consistent commit history.

**Severity guidance**:

| Context | Severity |
|---|---|
| Project declares use of Conventional Commits but most commits do not follow it | MEDIUM |
| Most commits follow convention; a few deviate | LOW |
| Consistent Conventional Commits usage | INFO (positive) |
| Project does not use Conventional Commits (no finding) | -- |

**Conventional Commits format**:

```
<type>[optional scope]: <description>

[optional body]

[optional footer(s)]
```

**Valid types**: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`

**Breaking change indicators**:
- `feat!: ` or `fix!: ` (exclamation mark before colon)
- `BREAKING CHANGE: ` in the footer

**Example**:

Problem:
```
fix stuff
update code
wip
Merge branch 'main' into feature
```

Better:
```
feat(auth): add TOTP-based two-factor authentication

Implements RFC 6238 TOTP with configurable time step and code length.
Supports both authenticator apps and SMS fallback.

Closes #142

---

fix(api): prevent session fixation on privilege escalation

Regenerate session token when user role changes to prevent
session fixation attacks.

BREAKING CHANGE: Session tokens are now invalidated on role change.
Clients must re-authenticate after admin promotes a user.

Closes #148
```

---

## 3. Version Management

### 3.1 Semantic Versioning Compliance

**Checkpoint**: If the project uses semantic versioning (most projects should), version numbers must follow the MAJOR.MINOR.PATCH scheme as defined by [semver.org](https://semver.org/). The version increment must match the nature of the changes.

**Severity guidance**:

| Context | Severity |
|---|---|
| Breaking change with only a patch or minor version bump | HIGH |
| New feature with only a patch version bump | MEDIUM |
| Patch version bump for a bug fix (correct) | -- |
| Version number does not follow semver format | MEDIUM |
| Pre-1.0 project with unstable API (semver rules are relaxed) | LOW |

**Semver rules**:

| Change Type | Version Increment | Example |
|---|---|---|
| Breaking change (removes/changes public API) | MAJOR | 1.2.3 -> 2.0.0 |
| New feature (backward-compatible addition) | MINOR | 1.2.3 -> 1.3.0 |
| Bug fix (backward-compatible) | PATCH | 1.2.3 -> 1.2.4 |
| Pre-release | Append identifier | 2.0.0-beta.1 |
| Build metadata | Append after `+` | 2.0.0+build.42 |

---

### 3.2 Version Consistency Across Files

**Checkpoint**: All files that declare the project version must agree. Common locations include `package.json`, `pyproject.toml`, `setup.py`, `setup.cfg`, `version.txt`, `VERSION`, `*.gemspec`, `Cargo.toml`, `build.gradle`, `pom.xml`, `Chart.yaml`, and in-code constants like `__version__`. Inconsistent versions cause confusion about what is actually deployed and what changelog applies.

**Severity guidance**:

| Context | Severity |
|---|---|
| Version mismatch between package manifest and published release | HIGH |
| Version mismatch between multiple version-declaring files | MEDIUM |
| Version in code constant does not match package manifest | MEDIUM |
| Single source of truth for version, consistently applied | INFO (positive) |

**How to check**:

```bash
# Common version locations to cross-reference
grep -r '"version"' package.json
grep 'version' pyproject.toml setup.cfg setup.py
cat VERSION version.txt
grep '__version__' src/**/*.py
grep 'version' Cargo.toml Chart.yaml
```

**Example**:

Problem:
```
package.json:     "version": "2.1.0"
pyproject.toml:   version = "2.0.3"
src/__init__.py:  __version__ = "2.0.0"
CHANGELOG.md:     ## [2.1.1] - 2025-01-15  (latest entry)
```

Four files, four different versions. Which is correct?

Better -- single source of truth:
```toml
# pyproject.toml (authoritative source)
[project]
version = "2.1.1"
dynamic = []

# All other references derive from this:
# - CI reads version from pyproject.toml
# - __version__ is set via importlib.metadata
# - Release tags are created from the same value
```

```python
# src/__init__.py
from importlib.metadata import version
__version__ = version("mypackage")  # reads from installed package metadata
```

---

### 3.3 Pre-Release Version Handling

**Checkpoint**: Pre-release versions (alpha, beta, release candidate) should follow semver pre-release conventions. Pre-release identifiers must not be used for production releases. The progression from pre-release to release should be clear.

**Severity guidance**:

| Context | Severity |
|---|---|
| Production deployment running a pre-release version | HIGH |
| Pre-release identifiers used inconsistently (mixing `alpha`, `dev`, `rc`, `beta` without convention) | LOW |
| Clear pre-release progression documented | INFO (positive) |

**Correct pre-release progression**:

```
2.0.0-alpha.1    First alpha (internal testing)
2.0.0-alpha.2    Second alpha
2.0.0-beta.1     First beta (external testing)
2.0.0-beta.2     Second beta
2.0.0-rc.1       Release candidate 1
2.0.0-rc.2       Release candidate 2 (if rc.1 had issues)
2.0.0            Stable release
```

---

### 3.4 Version-Changelog Alignment

**Checkpoint**: Every version bump should have a corresponding changelog entry, and every changelog entry should correspond to a version bump (or the `[Unreleased]` section). Misalignment indicates either undocumented releases or phantom changelog entries.

**Severity guidance**:

| Context | Severity |
|---|---|
| Released version with no changelog entry | HIGH |
| Changelog entry for a version that does not exist (no tag, no release) | MEDIUM |
| Version bumped in manifest but changelog still shows previous version as latest | MEDIUM |
| Changelog and version are aligned | INFO (positive) |

---

## 4. Decision Records

### 4.1 ADR Format Compliance

**Checkpoint**: Architectural Decision Records (ADRs) should follow a consistent format. The most widely adopted format includes: title, status, context, decision, and consequences. ADRs without these sections are incomplete and hard to use.

**Severity guidance**:

| Context | Severity |
|---|---|
| Significant decisions made with no documentation at all | HIGH |
| Decision records exist but lack key sections (context, consequences) | MEDIUM |
| ADRs follow a consistent format with all required sections | INFO (positive) |

**Standard ADR template**:

```markdown
# ADR-NNN: Title of Decision

## Status
[Proposed | Accepted | Deprecated | Superseded by ADR-NNN]

## Date
YYYY-MM-DD

## Context
What is the issue that we're seeing that is motivating this decision or change?
What are the forces at play (technical, business, organizational)?

## Decision
What is the change that we're proposing or have agreed to implement?

## Alternatives Considered

### Alternative 1: [Name]
- **Pros**: ...
- **Cons**: ...
- **Why rejected**: ...

### Alternative 2: [Name]
- **Pros**: ...
- **Cons**: ...
- **Why rejected**: ...

## Consequences

### Positive
- What becomes easier or possible as a result of this change?

### Negative
- What becomes harder or is lost as a result of this change?

### Risks
- What new risks does this decision introduce?

## References
- Links to relevant documents, issues, or discussions
```

**Example**:

Problem -- undocumented decision with lost context:
```python
# Why are we using Redis instead of Memcached? Nobody remembers.
# Why is this a REST API and not GraphQL? The person who decided left.
# Why do we deploy to us-east-1 and not us-west-2? ¯\_(ツ)_/¯
```

Better -- decision documented as ADR:
```markdown
# ADR-007: Use Redis for Session Storage

## Status
Accepted

## Date
2024-09-15

## Context
We need a session storage backend for our web application. Sessions must survive
application restarts, support TTL-based expiration, and handle ~5,000 concurrent
sessions. Our infrastructure already runs Redis for the job queue.

## Decision
Use Redis (our existing instance) for session storage with a dedicated database
number (db=1) to isolate session data from job queue data.

## Alternatives Considered

### Memcached
- **Pros**: Simpler, purpose-built for caching
- **Cons**: No persistence; sessions lost on restart; requires new infrastructure
- **Why rejected**: We need session persistence across deploys; adding Memcached
  increases operational burden for no clear benefit over Redis

### PostgreSQL
- **Pros**: Already in our stack; ACID guarantees
- **Cons**: Higher latency for session lookups; connection pool pressure
- **Why rejected**: Session lookups are on every request; adding load to the
  primary database is not acceptable at our projected scale

## Consequences

### Positive
- No new infrastructure to manage
- Sub-millisecond session lookups
- Built-in TTL support for session expiration

### Negative
- Redis becomes a single point of failure for both jobs and sessions
- Must monitor Redis memory usage more carefully

### Risks
- Redis memory exhaustion could affect both sessions and job queue
- Mitigation: Set maxmemory policy; alert at 70% usage; plan Redis cluster at 80%
```

---

### 4.2 Decision Discoverability

**Checkpoint**: Decision records must be discoverable. ADRs buried in a directory nobody knows about are as useless as no ADRs. There should be an index, a link from the main README or PLAN.md, or a consistent well-known location.

**Severity guidance**:

| Context | Severity |
|---|---|
| ADRs exist but are not linked from any navigational document | MEDIUM |
| ADRs are in a well-known location (e.g., `docs/adr/`) but not indexed | LOW |
| ADR index exists with status and links | INFO (positive) |

**Example of an ADR index**:

```markdown
# Architecture Decision Records

| # | Title | Status | Date |
|---|-------|--------|------|
| [ADR-001](001-use-postgresql.md) | Use PostgreSQL for primary database | Accepted | 2024-06-01 |
| [ADR-002](002-rest-api-design.md) | REST API design conventions | Accepted | 2024-06-15 |
| [ADR-003](003-monorepo-structure.md) | Monorepo with workspace packages | Accepted | 2024-07-01 |
| [ADR-004](004-graphql-for-frontend.md) | Use GraphQL for frontend API | Superseded by ADR-008 | 2024-07-15 |
| [ADR-005](005-jwt-authentication.md) | JWT-based authentication | Accepted | 2024-08-01 |
| [ADR-006](006-kubernetes-deployment.md) | Deploy on Kubernetes | Accepted | 2024-08-15 |
| [ADR-007](007-redis-session-storage.md) | Use Redis for session storage | Accepted | 2024-09-15 |
| [ADR-008](008-rest-only-api.md) | REST-only API (replaces GraphQL) | Accepted | 2024-10-01 |
```

---

### 4.3 Supersession Tracking

**Checkpoint**: When a decision is superseded by a new decision, both the old and new ADR should reference each other. The old ADR's status should change to "Superseded by ADR-NNN" and the new ADR should state "Supersedes ADR-NNN" in its context.

**Severity guidance**:

| Context | Severity |
|---|---|
| Contradictory ADRs with no supersession tracking (both appear active) | MEDIUM |
| Superseded ADR not updated with new status | LOW |
| Clear supersession chain with bidirectional references | INFO (positive) |

**Example**:

Problem:
```markdown
# ADR-004: Use GraphQL for frontend API
## Status: Accepted        <-- still says "Accepted" but team switched to REST

# ADR-008: Use REST for all APIs
## Status: Accepted        <-- no reference to ADR-004
```

Better:
```markdown
# ADR-004: Use GraphQL for Frontend API
## Status: Superseded by [ADR-008](008-rest-only-api.md)
...

# ADR-008: REST-Only API
## Status: Accepted
## Context
Supersedes [ADR-004](004-graphql-for-frontend.md).
After 3 months with GraphQL, we found that the schema maintenance overhead
exceeded the benefits for our relatively simple data model...
```

---

## 5. Velocity Tracking

### 5.1 Burn-Down / Burn-Up Patterns

**Checkpoint**: For time-bound projects, some form of progress tracking against plan should exist. This can be a formal burn-down chart, a simple checklist with completion dates, or a periodic status update that tracks done vs. remaining. The goal is to answer: "Are we on track?"

**Severity guidance**:

| Context | Severity |
|---|---|
| Deadline-driven project with no progress tracking mechanism | HIGH |
| Progress tracking exists but is not updated regularly | MEDIUM |
| Informal tracking (e.g., checked-off items with dates) | LOW |
| Regular progress updates with trend analysis | INFO (positive) |

**Example**:

Problem -- no way to tell if the project is on track:
```markdown
## Phase 2 (due April 30)
- [ ] Task A
- [ ] Task B
- [x] Task C
- [ ] Task D
- [x] Task E
- [ ] Task F
```
(Is this on track? When were C and E completed? What is the completion rate?)

Better -- progress with timestamps:
```markdown
## Phase 2 Progress (due April 30)

**Started**: March 1 | **Target**: April 30 | **Elapsed**: 4 of 9 weeks

| Task | Status | Completed | Notes |
|------|--------|-----------|-------|
| Task A: Database migration | Done | Mar 8 | Took 1 week (estimated 1 week) |
| Task B: API endpoints | Done | Mar 20 | Took 1.5 weeks (estimated 1 week) |
| Task C: Authentication | In Progress | -- | Started Mar 21; on track |
| Task D: Frontend integration | Not Started | -- | Blocked by Task C |
| Task E: Load testing | Not Started | -- | Depends on Task B + D |
| Task F: Documentation | Not Started | -- | Can parallelize with E |

**Velocity**: 2 tasks in 3 weeks (~0.67 tasks/week)
**Remaining**: 4 tasks in 5 weeks (~0.80 tasks/week needed)
**Assessment**: Slightly behind pace. Task B overran estimate by 50%.
If Task C also overruns, consider descoping Task F to post-launch.
```

---

### 5.2 Estimation Accuracy

**Checkpoint**: If the project uses estimates, comparing estimates to actuals provides valuable data for future planning. Consistently overestimating or underestimating indicates a calibration problem. The agent should flag systematic estimation issues when the data is available.

**Severity guidance**:

| Context | Severity |
|---|---|
| Estimates exist but actuals are never recorded, preventing learning | MEDIUM |
| Systematic underestimation (most tasks take 2-3x longer) with no adjustment | MEDIUM |
| Estimates and actuals tracked with occasional retrospective notes | INFO (positive) |

**Example**:

Problem -- estimation data lost:
```markdown
## Tasks
- [x] Task A (estimated: 1 week)    <-- how long did it actually take?
- [x] Task B (estimated: 3 days)    <-- no actual recorded
- [ ] Task C (estimated: 2 weeks)   <-- is this estimate reliable given the above?
```

Better:
```markdown
## Tasks (with estimation tracking)

| Task | Estimate | Actual | Variance | Notes |
|------|----------|--------|----------|-------|
| Task A | 1 week | 1.5 weeks | +50% | Unexpected schema migration complexity |
| Task B | 3 days | 2 days | -33% | Reused pattern from Task A |
| Task C | 2 weeks | -- | -- | Adjusted to 3 weeks based on Task A variance |

**Estimation accuracy**: Averaging +10% over (small sample). Will recalibrate
after Phase 1 completes.
```

---

### 5.3 Scope Creep Detection

**Checkpoint**: The agent should detect signs of scope creep: tasks being added to a phase after it was planned, the total task count growing without corresponding timeline adjustment, or new requirements appearing mid-phase without explicit acknowledgment.

**Severity guidance**:

| Context | Severity |
|---|---|
| Scope expanded significantly with no timeline or resource adjustment | HIGH |
| New tasks added with acknowledgment but no impact analysis | MEDIUM |
| Scope changes tracked with explicit trade-off decisions | INFO (positive) |

**Detection signals**:

- Git history of planning documents shows task additions after initial plan commit
- Total task count in a phase is higher than the original plan
- New categories of work appear that were not in the original scope
- Timeline unchanged despite scope growth

**Example**:

Problem -- scope creep in git history:

```diff
# git diff of PLAN.md between initial commit and current state

 ## Phase 1 Tasks (due March 31)
 - [ ] Implement user API
 - [ ] Implement auth
 - [ ] Set up CI/CD
+- [ ] Add admin dashboard          <-- added Feb 15
+- [ ] Build reporting module       <-- added Feb 20
+- [ ] Implement email notifications  <-- added Feb 28
+- [ ] Add CSV export               <-- added Mar 5
+- [ ] Multi-language support       <-- added Mar 10
```

Five tasks added to a phase originally scoped at three tasks, with no change to the March 31 deadline.

Better -- scope change acknowledged:
```markdown
## Phase 1 Scope Change Log

| Date | Change | Impact | Decision |
|------|--------|--------|----------|
| Feb 15 | Added admin dashboard | +2 weeks effort | Accepted; shifted CSV export to Phase 2 |
| Feb 28 | Added email notifications | +1 week effort | Accepted; extended Phase 1 deadline to April 7 |
| Mar 10 | Request for multi-language support | +3 weeks effort | Deferred to Phase 2; not critical for launch |
```

---

## Progress Tracking Templates

### Status Update Template

```markdown
## Status Update - [Date]

### Overall Status: [On Track | At Risk | Behind | Blocked]

### Completed Since Last Update
- Item 1 (completed [date])
- Item 2 (completed [date])

### In Progress
- Item 3: [brief status, expected completion]
- Item 4: [brief status, expected completion]

### Blockers
- Blocker 1: [description, who can unblock, escalation path]

### Risks (new or changed)
- Risk 1: [description, likelihood change, mitigation update]

### Scope Changes
- [Any additions, removals, or deferrals since last update]

### Next Week Focus
- Priority 1
- Priority 2
```

### Changelog Entry Template

```markdown
## [X.Y.Z] - YYYY-MM-DD

### Added
- Description of new feature ([#PR](link))

### Changed
- **BREAKING**: Description of breaking change ([#PR](link))
  Migration: [brief instruction or link to migration guide]
- Description of non-breaking change ([#PR](link))

### Deprecated
- Description of deprecated feature; use [replacement] instead.
  Removal planned for vX.0. ([#PR](link))

### Removed
- **BREAKING**: Description of removed feature ([#PR](link))

### Fixed
- Description of bug fix ([#PR](link))

### Security
- Description of security fix, CVE reference if applicable ([#PR](link))
```

---

## Quick-Reference Summary

| Area | Checkpoint | Severity |
|---|---|---|
| Status Accuracy | Status says "complete" for unimplemented items | CRITICAL |
| Status Accuracy | "In progress" items with no activity > 90 days | MEDIUM |
| Status Accuracy | Status not updated in > 30 days | LOW |
| Status Accuracy | Milestone complete but deliverables missing from codebase | CRITICAL |
| Status Accuracy | Task complete but implementation is a stub/TODO | HIGH |
| Changelog | Released project with no changelog | HIGH |
| Changelog | Changelog is unstructured commit dump | MEDIUM |
| Changelog | Entries uncategorized (flat list) | MEDIUM |
| Changelog | Entries in wrong category | LOW |
| Changelog | Breaking change shipped with no changelog mention | HIGH |
| Changelog | Breaking change in changelog but not marked as breaking | MEDIUM |
| Changelog | Breaking change marked but no migration guidance | MEDIUM |
| Changelog | No links to issues/PRs | LOW |
| Changelog | Conventional Commits declared but not followed | MEDIUM |
| Version Mgmt | Breaking change with patch/minor version bump | HIGH |
| Version Mgmt | New feature with patch-only bump | MEDIUM |
| Version Mgmt | Version mismatch between files | MEDIUM |
| Version Mgmt | Production running pre-release version | HIGH |
| Version Mgmt | Released version with no changelog entry | HIGH |
| Version Mgmt | Changelog entry for nonexistent version | MEDIUM |
| Decisions | Significant decisions with no documentation | HIGH |
| Decisions | ADRs missing key sections (context, consequences) | MEDIUM |
| Decisions | ADRs exist but not discoverable (no index, no links) | MEDIUM |
| Decisions | Contradictory active ADRs with no supersession tracking | MEDIUM |
| Velocity | Deadline project with no progress tracking | HIGH |
| Velocity | Progress tracking not updated regularly | MEDIUM |
| Velocity | Estimates exist but actuals never recorded | MEDIUM |
| Velocity | Systematic underestimation with no adjustment | MEDIUM |
| Velocity | Scope expanded with no timeline/resource adjustment | HIGH |
| Velocity | Scope changes tracked with trade-off decisions | INFO (positive) |
