# Customer-Facing Documentation Reference Checklist

Canonical reference for evaluating documentation intended for external users, customers, and integrators. Customer-facing documentation has higher standards than internal documentation because it represents the product, shapes user success, and directly affects support burden and adoption. Complements the inlined criteria in `review-documentation.md`.

The `documentation-reviewer` agent uses this file as a lookup when evaluating customer-facing documentation. Severity levels follow `criteria/shared/severity-levels.md`.

---

## 1. Audience Analysis

**Category**: Audience

Customer-facing documentation serves multiple audiences with different goals, skill levels, and contexts. Documentation that conflates these audiences fails all of them. Audience analysis determines what content to write, at what depth, and in what structure.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Target audiences are explicitly identified (e.g., developers, operators, architects, end users) | MEDIUM | Without defined audiences, documentation oscillates in depth and tone |
| Each document or section is written for a specific audience, not "everyone" | MEDIUM | All-audience documents are too basic for experts and too advanced for beginners |
| Audience-specific entry points exist (developer docs, admin guide, user guide) | HIGH | Forcing all audiences through the same path wastes time for experienced users and overwhelms beginners |
| Personas or user stories are considered when structuring documentation | LOW | Personas help organize content around user goals rather than product features |
| Documentation accounts for users who are evaluating the product (not yet committed) | MEDIUM | Evaluators need quick answers: what does it do, how hard is it, what does it cost in complexity |
| Documentation accounts for existing users performing day-to-day operations | MEDIUM | Operations docs are different from setup docs; existing users need different content than new users |
| Documentation accounts for users migrating from a competing product | LOW | Migration guides reduce adoption friction for users switching from alternatives |

### Detection Patterns

**Audience conflation**: Look for documents that swing between explaining basic concepts and assuming advanced knowledge within the same section. This indicates no defined audience.

**Missing audience segments**: Review the documentation structure and check for gaps:

```
Common audience segments for infrastructure/platform products:
- Evaluator:  Needs a clear product overview, feature comparison, quickstart
- Developer:  Needs API reference, SDK guides, code examples, integration patterns
- Operator:   Needs deployment guide, configuration reference, monitoring setup, backup procedures
- Architect:  Needs architecture overview, scalability model, security model, integration points
- End user:   Needs UI documentation, workflow guides, FAQs
- Migrator:   Needs migration guide from specific alternatives
```

If any relevant segment has no dedicated content, this is a gap.

### Common Mistakes

**Mistake**: Writing all documentation for one persona.

Bad: A project that has only developer-focused documentation with API references and code examples, but no operational documentation for the platform team that runs it in production.

Good: Separate documentation tracks:
```
docs/
  getting-started/          <- Evaluators and new users
    overview.md
    quickstart.md
    concepts.md
  developer-guide/          <- Developers integrating with the product
    api-reference.md
    sdk-guide.md
    examples/
  operations-guide/         <- Platform/ops teams
    deployment.md
    configuration.md
    monitoring.md
    backup-restore.md
    upgrade.md
  architecture/             <- Architects evaluating fit
    overview.md
    security-model.md
    scalability.md
```

**Mistake**: Not considering the evaluator audience.

Evaluators are the highest-leverage audience. If they cannot quickly determine whether the product fits their needs, they move on. Evaluators need:
- A one-paragraph description of what the product does
- A list of key features and capabilities
- A quickstart that works in under 10 minutes
- Clear documentation of limitations and requirements

### Severity Guidance

| Context | Severity |
|---------|----------|
| No audience-specific entry points; all content in one undifferentiated mass | HIGH |
| Important audience segment completely unserved (e.g., no operations docs for a production system) | HIGH |
| Audiences identified but content oscillates between levels | MEDIUM |
| Evaluator audience not considered (no quickstart or overview) | MEDIUM |
| Migration guides missing for users of known competing products | LOW |
| Well-segmented documentation with clear audience labels on each section | INFO (positive) |

---

## 2. Skill-Level Appropriate Content

**Category**: Audience

Different audiences have different skill levels. A getting-started guide for Python beginners and an advanced integration guide for senior engineers require fundamentally different approaches. Within each audience, content should match the expected skill level without condescension or assumption.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Getting-started guides assume minimal prior knowledge of the product (though they may assume domain knowledge) | HIGH | First-time users who fail at getting started do not become users at all |
| Advanced guides are appropriately dense and do not re-explain basics covered in getting-started | LOW | Experienced users are frustrated by repeated basic explanations in advanced content |
| Concepts are introduced before they are used in procedures | MEDIUM | Using a term before defining it loses readers who are learning |
| Progressive complexity: simple examples first, then advanced variations | MEDIUM | Building from simple to complex matches how people learn |
| Escape hatches exist for readers who need more background ("for background on X, see...") | LOW | Links to background content let advanced readers skip and beginners dive deeper |
| Content does not assume familiarity with the product's internal architecture | MEDIUM | Customers do not know (and should not need to know) internal implementation details |
| Jargon is product-external only; internal code names and team-specific terms are not used | MEDIUM | "Run the bifrost pipeline" means nothing to a customer |

### Common Mistakes

**Mistake**: Getting-started guide that assumes product knowledge.

Bad:
```markdown
# Getting Started

1. Initialize the workspace resolver:
   ```bash
   myproduct init --resolver=standard
   ```

2. Configure the sync target:
   ```bash
   myproduct config set sync.target s3://your-bucket/path
   ```

3. Run the first reconciliation:
   ```bash
   myproduct reconcile --mode=full
   ```
```

This assumes the reader knows what a "workspace resolver", "sync target", and "reconciliation" are in this product's context.

Good:
```markdown
# Getting Started

This guide walks you through setting up myproduct to synchronize your
local project files with cloud storage. By the end, your project
directory will be automatically backed up to S3.

## Step 1: Initialize Your Project

Create a myproduct configuration in your project directory:
```bash
cd /path/to/your/project
myproduct init
```

This creates a `.myproduct/` directory containing configuration files.

## Step 2: Connect to Cloud Storage

Tell myproduct where to store your synchronized files:
```bash
myproduct config set sync.target s3://your-bucket/myproject
```

Replace `your-bucket` with your S3 bucket name. You must have write
access to this bucket (see [AWS Permissions](#aws-permissions)).

## Step 3: Run Your First Sync

Synchronize your project files to cloud storage:
```bash
myproduct sync
```

Expected output:
```
Scanning 142 files...
Uploading 142 files to s3://your-bucket/myproject
Sync complete: 142 files uploaded (23.4 MB)
```
```

**Mistake**: Using internal terminology in customer-facing docs.

Bad:
```markdown
If the hydra-cache is stale, trigger a phoenix rebuild by calling the
/admin/rebuild endpoint. This invokes the thunderbolt pipeline which
reprocesses all entities through the kraken service.
```

Good:
```markdown
If the cache is stale, rebuild it by calling the admin API:
```bash
curl -X POST http://localhost:8080/admin/rebuild
```

This triggers a full cache rebuild, which reprocesses all entities. The
rebuild typically takes 2-5 minutes depending on data volume.
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Getting-started guide fails without undocumented product knowledge | HIGH |
| Internal code names or team jargon used in customer-facing docs | MEDIUM |
| Concepts used before they are introduced or explained | MEDIUM |
| Advanced content re-explains basics, adding unnecessary length | LOW |
| Progressive complexity with clear links to background material | INFO (positive) |

---

## 3. Quickstart, Reference, and Advanced Separation

**Category**: Document Structure

Three types of documentation serve fundamentally different purposes: quickstarts (learning), references (looking up), and advanced guides (deepening). Mixing them produces documents that are too long to learn from, too incomplete to reference, and too shallow for advanced use.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| A quickstart guide exists that gets users to a working state in under 15 minutes | HIGH | Quickstarts are the #1 driver of adoption; their absence correlates with abandonment |
| A reference section exists with complete, exhaustive coverage of all features | MEDIUM | Quickstarts cover the common path; reference covers everything |
| Advanced/how-to guides exist for non-trivial use cases | MEDIUM | Real-world usage goes beyond the quickstart; users need guidance for complex scenarios |
| Quickstart is minimal (does not try to cover all features) | MEDIUM | Quickstarts that cover everything become tutorials, not quickstarts |
| Reference documentation is exhaustive (every parameter, every option, every endpoint) | HIGH | Incomplete reference means some features are undocumented |
| Content type is labeled or structurally obvious (users can tell what type of document they are reading) | LOW | Users looking for a reference who land on a tutorial waste time |
| Cross-references link between types ("for the full API reference, see..." in the quickstart) | LOW | Cross-references help users navigate between content types |

### Content Type Definitions

**Quickstart** (also: Getting Started, Tutorial, Hello World):
- Goal: Get the user to a working result as fast as possible
- Length: 5-15 minutes of reading and doing
- Scope: One happy path, the most common use case
- Style: Step-by-step, prescriptive, "do this, then this"
- Does not explain: alternative approaches, edge cases, or all options
- Ends with: a working result and links to learn more

**Reference** (also: API Reference, Configuration Reference, CLI Reference):
- Goal: Provide complete, look-uppable documentation of all features
- Length: As long as needed for completeness
- Scope: Everything -- every parameter, endpoint, option, and error code
- Style: Factual, structured, consistently formatted entries
- Does not include: step-by-step tutorials or opinion on best approaches
- Organized by: logical grouping (by resource, by module, alphabetically)

**Advanced Guide** (also: How-To, Cookbook, Recipe):
- Goal: Solve a specific complex problem
- Length: Varies (typically 5-20 minutes)
- Scope: One specific use case or integration pattern
- Style: Problem-solution format; assumes familiarity with basics
- Does not include: basic setup (links to quickstart instead)
- Examples: "How to integrate with Kafka", "Setting up HA", "Custom authentication"

### Common Mistakes

**Mistake**: Quickstart that tries to be a reference.

Bad:
```markdown
# Quickstart

## Step 1: Initialize

```bash
myapp init [--workspace NAME] [--template TEMPLATE] [--no-git] [--verbose]
```

Options:
- `--workspace`: The workspace name (default: directory name, max 64 chars,
  alphanumeric and hyphens only)
- `--template`: Template to use. Options: basic, advanced, enterprise,
  microservice, monolith (default: basic)
- `--no-git`: Skip git initialization
- `--verbose`: Show detailed output
...
```

This is reference content in a quickstart. The user does not need all options at this point; they need the one command that works.

Good:
```markdown
# Quickstart

## Step 1: Initialize

```bash
myapp init
```

This creates a new workspace in the current directory with default settings.
For custom options, see the [`init` command reference](./reference/cli.md#init).
```

**Mistake**: No quickstart at all; documentation starts with concepts or architecture.

Bad (documentation structure):
```
docs/
  architecture.md
  data-model.md
  api-reference.md
  configuration.md
```

A new user has no entry point. They must understand the architecture and data model before they can use the product.

Good:
```
docs/
  quickstart.md              <- First entry point for all new users
  concepts.md                <- Background for users who want to understand before doing
  developer-guide/
    api-reference.md
    sdk-guide.md
  operations-guide/
    configuration.md
    deployment.md
  architecture/
    overview.md
    data-model.md
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No quickstart guide exists | HIGH |
| Reference documentation incomplete (features exist but are undocumented) | HIGH |
| Quickstart tries to cover all features, becoming a 30+ minute tutorial | MEDIUM |
| No advanced guides for non-trivial use cases (HA, custom auth, integrations) | MEDIUM |
| Content types not labeled or structurally distinguished | LOW |
| Clean separation between quickstart, reference, and advanced guides with cross-references | INFO (positive) |

---

## 4. Troubleshooting Sections

**Category**: Support

Troubleshooting documentation directly reduces support burden. Every documented error-to-resolution mapping is a support ticket that does not get filed. For customer-facing products, troubleshooting quality has measurable business impact.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| A troubleshooting section or page exists | HIGH | Absence of troubleshooting docs sends every error to support |
| Troubleshooting covers the 10 most common support issues | HIGH | The Pareto principle applies: documenting top-10 issues deflects 80% of support tickets |
| Each troubleshooting entry has: symptom, cause, and fix | MEDIUM | Symptom-only or cause-only entries are not actionable |
| Error messages are searchable (quoted exactly as they appear) | MEDIUM | Users search by copying error messages; docs must match exactly |
| Diagnostic commands are provided ("how to check if X is the problem") | MEDIUM | Users need to confirm the diagnosis before applying the fix |
| Troubleshooting entries are organized by symptom, not cause | MEDIUM | Users know their symptom but not the cause; organize for their starting point |
| Escalation path is documented when self-service troubleshooting fails | LOW | Users who exhaust the troubleshooting guide need to know where to go next |
| Troubleshooting is kept up-to-date as new issues are discovered | MEDIUM | Stale troubleshooting docs that do not cover recent issues lose trust |

### Detection Patterns

**Missing troubleshooting**: Check for these indicators of missing troubleshooting content:
- No `troubleshooting` section, page, or document in the docs
- No FAQ or "Common Issues" section
- Error messages in the code that have no corresponding documentation
- Support channels (GitHub issues, forums) with repeated questions that could be in docs

**Symptom vs. cause organization**: Troubleshooting organized by cause requires the user to already know the cause, which defeats the purpose.

Bad (organized by cause):
```markdown
## DNS Resolution

If DNS resolution is failing, you may see connection timeout errors...

## Certificate Expiration

If your TLS certificate has expired, you may see handshake errors...
```

Good (organized by symptom):
```markdown
## "Connection timed out" error

**Possible causes**:
1. DNS resolution failure
2. Firewall blocking outbound connections
3. Target service is down

**Diagnosis**:
...
```

### Common Mistakes

**Mistake**: Troubleshooting entries without diagnostic steps.

Bad:
```markdown
## Service won't start

If the service fails to start, it may be a port conflict. Try changing
the port in the configuration.
```

This skips diagnosis. Maybe the port is not the problem at all.

Good:
```markdown
## Service fails to start

### Symptom
The service exits immediately after `myservice start` with no output,
or with the message "Failed to bind to address."

### Diagnosis

1. Check if another process is using the configured port:
   ```bash
   lsof -i :8080
   ```
   If a process is listed, the port is in use.

2. Check the service logs for the specific error:
   ```bash
   journalctl -u myservice --since "5 minutes ago"
   ```

### Fix

**If port conflict** (Step 1 showed a process):
- Stop the conflicting process, or
- Change the port in `config.yaml`:
  ```yaml
  server:
    port: 8081  # Changed from 8080
  ```

**If permission error** (log shows "Permission denied"):
- The service needs access to its data directory:
  ```bash
  sudo chown -R myservice:myservice /var/lib/myservice
  ```

**If configuration error** (log shows "Invalid configuration"):
- Validate the configuration:
  ```bash
  myservice check-config
  ```
  Fix the reported errors and retry.
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No troubleshooting documentation for a customer-facing product | HIGH |
| Top-10 support issues not covered in troubleshooting docs | HIGH |
| Troubleshooting entries lack diagnostic steps (go straight to fix) | MEDIUM |
| Error messages not searchable (paraphrased instead of quoted) | MEDIUM |
| Troubleshooting organized by cause instead of symptom | MEDIUM |
| No escalation path documented | LOW |
| Comprehensive troubleshooting with symptoms, diagnostics, and fixes | INFO (positive) |

---

## 5. Error-to-Resolution Mapping

**Category**: Support

Error-to-resolution mapping is the most directly actionable form of troubleshooting documentation. It takes the exact error message a user sees and maps it to a specific resolution. This is distinct from general troubleshooting because it is indexed by the exact string the user encounters.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| All user-facing error messages are documented | HIGH | Undocumented errors send users to support or search engines |
| Error codes exist and are documented with meaning and resolution | MEDIUM | Error codes enable faster lookup than full message text |
| Error messages include enough context to self-diagnose (file path, parameter name, expected vs. actual value) | MEDIUM | "Invalid value" is not diagnosable; "Invalid value for port: 'abc' (expected integer)" is |
| HTTP error responses include structured error bodies, not just status codes | MEDIUM | `{"error": "rate_limit_exceeded", "retry_after": 30}` is actionable; a bare 429 is not |
| Error documentation groups related errors by domain (auth errors, config errors, network errors) | LOW | Grouping helps users browse for their error when they cannot search |
| Error resolution includes both the fix and an explanation of why the error occurred | LOW | Understanding the cause prevents recurrence |

### Detection Patterns

**Error message inventory**: Search the codebase for all user-facing error messages:

```python
# Python
raise ValueError("...")
raise MyError("...")
logger.error("...")
sys.exit("Error: ...")

# Go
fmt.Errorf("...")
log.Fatal("...")
http.Error(w, "...", statusCode)

# JavaScript/TypeScript
throw new Error("...")
console.error("...")
res.status(400).json({error: "..."})
```

Cross-reference each error message against the documentation. Undocumented errors are findings.

**Error codes vs. error messages**: Check whether the product uses structured error codes:

```json
{
  "error": {
    "code": "AUTH_TOKEN_EXPIRED",
    "message": "The authentication token has expired",
    "details": "Token expired at 2024-01-15T14:00:00Z. Request a new token via POST /auth/token."
  }
}
```

If error codes exist, they must be documented with:
- The code string
- When it occurs
- How to resolve it
- Related error codes (if any)

### Common Mistakes

**Mistake**: Documenting error messages generically.

Bad:
```markdown
## Errors

The API may return the following errors:
- 400: Bad request
- 401: Unauthorized
- 404: Not found
- 500: Internal server error
```

This adds nothing beyond the HTTP spec.

Good:
```markdown
## Error Reference

### Authentication Errors

| Code | HTTP Status | Message | Cause | Resolution |
|------|------------|---------|-------|------------|
| `AUTH_TOKEN_MISSING` | 401 | "Authorization header required" | Request has no Authorization header | Add `Authorization: Bearer <token>` header |
| `AUTH_TOKEN_EXPIRED` | 401 | "Token expired" | Token's `exp` claim is in the past | Request a new token via `POST /auth/token` |
| `AUTH_TOKEN_INVALID` | 401 | "Invalid token" | Token signature verification failed | Verify you are using the correct signing key; regenerate the token |
| `AUTH_INSUFFICIENT_SCOPE` | 403 | "Insufficient scope: requires 'admin'" | Token does not have the required scope | Request a token with the `admin` scope |

### Validation Errors

| Code | HTTP Status | Message | Cause | Resolution |
|------|------------|---------|-------|------------|
| `VALIDATION_REQUIRED_FIELD` | 400 | "Field '{field}' is required" | A required field is missing from the request body | Add the missing field to your request |
| `VALIDATION_INVALID_TYPE` | 400 | "Field '{field}': expected {type}, got {actual}" | A field has the wrong data type | Correct the field type per the API reference |
| `VALIDATION_OUT_OF_RANGE` | 400 | "Field '{field}': value must be between {min} and {max}" | A numeric field is outside the allowed range | Adjust the value to fall within the documented range |

### Rate Limiting

| Code | HTTP Status | Message | Cause | Resolution |
|------|------------|---------|-------|------------|
| `RATE_LIMIT_EXCEEDED` | 429 | "Rate limit exceeded" | Too many requests in the current window | Wait for the duration in the `Retry-After` header. See [Rate Limits](#rate-limits) for quotas. |
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| User-facing error messages with no documentation | HIGH |
| Error responses lack context (no field name, no expected value, no error code) | MEDIUM |
| Error codes exist but are not documented | MEDIUM |
| Error documentation restates HTTP spec without product-specific detail | MEDIUM |
| Error resolution missing (documents the error but not how to fix it) | MEDIUM |
| Comprehensive error reference with codes, causes, and resolutions | INFO (positive) |

---

## 6. Known Limitations

**Category**: Transparency

Documenting known limitations is an act of respect for the user's time. When limitations are hidden, users discover them through failure -- often after investing significant effort. Transparent limitation documentation builds trust and enables informed decisions.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Known functional limitations are documented | HIGH | Undiscovered limitations cause production incidents |
| Scale limits are documented (max connections, max file size, max records, max throughput) | HIGH | Scale limits discovered in production are outages |
| Platform or environment restrictions are documented | MEDIUM | "Does not work on ARM" discovered after a 2-hour build is a waste |
| Performance characteristics under load are documented or referenced | MEDIUM | Performance expectations prevent sizing mistakes |
| Compatibility limitations with third-party products are documented | MEDIUM | "Does not work with PostgreSQL 16" saves users from troubleshooting a known incompatibility |
| Workarounds for limitations are provided when they exist | LOW | A documented limitation with a workaround is a minor inconvenience, not a blocker |
| Limitations distinguish between temporary (planned fix) and permanent (by design) | LOW | Users accept permanent limitations; they wait for temporary ones |
| Limitations are discoverable before purchase or adoption decision | HIGH | Limitations hidden behind a registration wall or buried in docs waste evaluator time |

### Common Mistakes

**Mistake**: No limitations section at all.

Every product has limitations. The absence of a limitations section does not mean there are no limitations; it means the user will discover them the hard way.

Good:
```markdown
## Known Limitations

### Scale Limits

| Resource | Limit | Consequence of Exceeding |
|----------|-------|-------------------------|
| Concurrent connections | 10,000 | New connections rejected with "connection refused" |
| File upload size | 100 MB | Upload fails with HTTP 413 |
| Records per table | 10 million | Query performance degrades beyond this; consider sharding |
| API requests per minute | 1,000 per API key | HTTP 429 returned; implement backoff |
| Webhook payload size | 1 MB | Payload truncated silently |

### Platform Restrictions

- **Windows**: Native Windows is not supported. Use WSL2 with Ubuntu 22.04+.
- **ARM64**: Supported on Linux (tested on AWS Graviton). Not tested on
  other ARM64 platforms.
- **Alpine Linux**: The default binary requires glibc. Use the
  `-alpine` variant for musl-based systems.

### Compatibility

- PostgreSQL 13+ required (uses features not available in PG 12)
- Redis 7.0+ required (uses Redis Streams)
- Not compatible with Redis Cluster (uses Lua scripts that span multiple keys)
- MySQL/MariaDB: Not supported. PostgreSQL only.

### Known Issues

| Issue | Workaround | Status |
|-------|-----------|--------|
| Memory leak under sustained high load (#1234) | Restart the service every 24h | Fix planned for v2.4 |
| Slow startup with >1000 config entries (#1456) | Split configuration across multiple files | Investigating |
| Incorrect timezone handling in reports (#1567) | Set `TZ=UTC` explicitly | Fix in v2.3.1 |
```

**Mistake**: Hiding limitations in footnotes or appendices.

Limitations should be discoverable. They belong in:
- The README (summary of major limitations)
- The quickstart (limitations that affect the getting-started experience)
- The relevant feature documentation (limitations of specific features)
- A dedicated "Known Limitations" page (comprehensive list)

### Severity Guidance

| Context | Severity |
|---------|----------|
| Scale limits undocumented for a production-grade product | HIGH |
| Known functional limitations undocumented | HIGH |
| Limitations not discoverable during evaluation phase | HIGH |
| Platform restrictions undocumented | MEDIUM |
| Compatibility limitations with common third-party products undocumented | MEDIUM |
| Known issues without workarounds and without status | MEDIUM |
| Workarounds provided for all documented limitations | INFO (positive) |
| Limitations transparently documented with workarounds and fix timelines | INFO (positive) |

---

## 7. Operational Documentation

**Category**: Operations

Operational documentation covers the lifecycle of a product in production: deployment, monitoring, backup, upgrade, and decommission. For customer-facing products, incomplete operational documentation means the customer's operations team cannot manage the product without filing support tickets for routine tasks.

### What to Check

#### Deployment

| Check | Severity | Rationale |
|-------|----------|-----------|
| Production deployment procedure is documented (not just dev setup) | HIGH | Dev setup and production deployment are fundamentally different |
| Infrastructure requirements are specified (CPU, memory, disk, network) | HIGH | Under-provisioned deployments cause outages; over-provisioned ones waste money |
| High-availability deployment is documented | HIGH | Single-instance deployment docs for a production system invite outages |
| Security hardening steps are documented (TLS, auth, network isolation) | HIGH | Default configurations are rarely production-secure |
| Container orchestration deployment is documented if applicable (Kubernetes, ECS, etc.) | MEDIUM | Most production deployments use orchestration; ad-hoc instructions do not translate |

#### Monitoring

| Check | Severity | Rationale |
|-------|----------|-----------|
| Key metrics to monitor are listed with descriptions | HIGH | Operations teams need to know what to watch |
| Health check endpoint exists and is documented | HIGH | Load balancers and orchestrators need health checks |
| Log format and log levels are documented | MEDIUM | Log parsing requires knowing the format; log levels need tuning guidance |
| Alerting thresholds are recommended | MEDIUM | Metrics without recommended thresholds are not actionable |
| Integration with common monitoring tools is documented (Prometheus, Datadog, CloudWatch) | MEDIUM | Generic metrics docs without integration guidance require custom work |

#### Backup and Recovery

| Check | Severity | Rationale |
|-------|----------|-----------|
| Backup procedure is documented for all persistent data | HIGH | Data without backup procedures is data at risk |
| Recovery procedure is documented with expected RTO/RPO | HIGH | Backup without tested recovery is not backup |
| What to back up (and what not to) is explicitly listed | MEDIUM | Users need to know if they back up the database, the config, the state directory, or all three |
| Backup verification procedure exists | MEDIUM | Untested backups may be corrupt or incomplete |

#### Upgrade

| Check | Severity | Rationale |
|-------|----------|-----------|
| Upgrade procedure is documented for each supported upgrade path | HIGH | Skipping versions may require different steps |
| Breaking changes are highlighted per version | HIGH | Users need to know what breaks before upgrading |
| Rollback procedure from upgrade is documented | HIGH | Upgrades without rollback plans are one-way doors |
| Data migration steps are documented separately from application upgrade | MEDIUM | Mixing them makes partial rollback impossible |
| Downtime requirements during upgrade are documented | MEDIUM | Operations teams need to plan maintenance windows |

### Common Mistakes

**Mistake**: Documenting only how to start the service, not how to run it in production.

Bad:
```markdown
## Deployment

```bash
./myservice start
```
```

Good:
```markdown
## Production Deployment

### Infrastructure Requirements

| Component | Minimum | Recommended | Notes |
|-----------|---------|-------------|-------|
| CPU | 2 cores | 4 cores | CPU-bound during batch processing |
| Memory | 2 GB | 4 GB | Base usage ~500 MB; add 100 MB per 10k concurrent connections |
| Disk | 10 GB | 50 GB | Log volume: ~1 GB/day at INFO level |
| Network | 100 Mbps | 1 Gbps | Throughput-dependent on API traffic |

### Deployment Options

#### Systemd (Linux)

1. Copy the binary and configuration:
   ```bash
   sudo cp myservice /usr/local/bin/
   sudo cp config.yaml /etc/myservice/
   ```

2. Install the systemd unit:
   ```bash
   sudo cp myservice.service /etc/systemd/system/
   sudo systemctl daemon-reload
   sudo systemctl enable myservice
   sudo systemctl start myservice
   ```

3. Verify:
   ```bash
   sudo systemctl status myservice
   curl http://localhost:8080/health
   ```

#### Kubernetes

See [Helm chart documentation](./kubernetes/README.md) for Kubernetes deployment.

#### Docker Compose (development/staging only)

See [Docker Compose setup](./docker/README.md).

### Security Hardening

- [ ] Enable TLS (see [TLS Configuration](#tls-configuration))
- [ ] Restrict bind address to internal network interface
- [ ] Enable authentication on all endpoints
- [ ] Set `LOG_LEVEL=warn` in production (INFO generates high volume)
- [ ] Configure firewall rules to allow only required ports
- [ ] Set resource limits (memory, file descriptors)
```

**Mistake**: No monitoring guidance.

Bad: The product exposes a `/metrics` endpoint with no documentation about what the metrics mean or what to alert on.

Good:
```markdown
## Monitoring

### Key Metrics

| Metric | Type | Description | Alert Threshold |
|--------|------|-------------|-----------------|
| `http_requests_total` | Counter | Total HTTP requests by method and status | Rate > 0 for 5xx: alert |
| `http_request_duration_seconds` | Histogram | Request latency | p99 > 2s for 5 min: alert |
| `db_connections_active` | Gauge | Active database connections | > 80% of max_connections: warn |
| `db_connections_idle` | Gauge | Idle database connections | 0 for 5 min: investigate |
| `queue_depth` | Gauge | Items in processing queue | > 1000 for 10 min: alert |
| `cache_hit_ratio` | Gauge | Cache hit percentage | < 50% for 30 min: investigate |

### Health Check

```
GET /health
```

Response (healthy):
```json
{
  "status": "healthy",
  "checks": {
    "database": "connected",
    "cache": "connected",
    "disk_space": "ok"
  }
}
```

Response (degraded):
```json
{
  "status": "degraded",
  "checks": {
    "database": "connected",
    "cache": "timeout",
    "disk_space": "ok"
  }
}
```

Use HTTP status code for load balancer health checks:
- 200: healthy (all checks pass)
- 503: unhealthy (any critical check fails)

### Grafana Dashboard

Import the provided dashboard from `monitoring/grafana-dashboard.json`.
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| No production deployment documentation (only dev setup) | HIGH |
| No monitoring guidance (no metrics, no health check) | HIGH |
| No backup/recovery procedure for stateful components | HIGH |
| No upgrade procedure or breaking-change documentation | HIGH |
| Infrastructure requirements undocumented | HIGH |
| HA deployment undocumented for production-grade product | HIGH |
| Security hardening steps missing | HIGH |
| Monitoring metrics documented without alerting thresholds | MEDIUM |
| Backup procedure without recovery test/verification | MEDIUM |
| Upgrade procedure without rollback plan | MEDIUM |
| Comprehensive operational documentation covering all lifecycle phases | INFO (positive) |

---

## 8. Release Notes Quality

**Category**: Communication

Release notes are the communication channel between the product team and customers for every version change. Good release notes help customers decide whether to upgrade, prepare for breaking changes, and take advantage of new features. Bad release notes are ignored, causing surprise breakages.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Release notes exist for every version | MEDIUM | Missing release notes for a version is a communication gap |
| Breaking changes are prominently marked and appear first | HIGH | Breaking changes buried at the bottom are missed |
| Migration steps for breaking changes are included or linked | HIGH | A breaking change without migration instructions is a blocker |
| New features are described with enough context to evaluate usefulness | MEDIUM | "Added feature X" without explaining what X does or how to use it is not useful |
| Bug fixes reference the issue or describe the symptom that was fixed | MEDIUM | "Fixed a bug" without context does not help users know if the bug affected them |
| Security fixes are flagged with severity and CVE references when applicable | HIGH | Security fixes need urgency signaling to prompt upgrades |
| Deprecations are announced with timeline and migration path | MEDIUM | Users need lead time to migrate away from deprecated features |
| Version numbering follows SemVer or a documented scheme | MEDIUM | Unpredictable versioning makes it impossible to assess upgrade risk |

### Detection Patterns

**Missing release notes**: Check that a CHANGELOG.md, RELEASES.md, GitHub Releases page, or equivalent exists and has entries for recent versions.

**Breaking change detection**: Look for these signals in the codebase diff between versions:
- Removed or renamed API endpoints
- Changed request/response schemas
- Removed CLI flags or changed their behavior
- Changed default configuration values
- Database schema migrations that are not backward-compatible
- Removed or renamed environment variables

Any of these that are not documented in release notes are findings.

### Common Mistakes

**Mistake**: Release notes that are just a commit log.

Bad:
```markdown
## v2.3.0

- Fixed typo
- Updated deps
- Added new feature
- Refactored auth module
- Fixed tests
- Merged PR #456
- WIP: new endpoint
- Code review fixes
```

Good:
```markdown
## v2.3.0 (2024-01-15)

### Breaking Changes

- **API**: The `GET /users` endpoint now requires authentication.
  Previously, it was publicly accessible. Add an `Authorization` header
  to all requests to this endpoint. See [Authentication](#authentication).

- **Configuration**: The `cache.type` parameter has been renamed to
  `cache.backend`. Update your `config.yaml` before upgrading.

### New Features

- **Webhook support**: Configure webhooks to receive notifications for
  key events (user creation, order completion). See the
  [Webhooks Guide](./webhooks.md).

- **Batch API**: New `POST /batch` endpoint for submitting up to 100
  operations in a single request. Reduces HTTP overhead for bulk operations.

### Bug Fixes

- Fixed: Users with special characters in their email could not log in
  (#1234). The email normalization now handles Unicode correctly.

- Fixed: Memory leak when processing large CSV uploads (#1256). Memory
  usage is now bounded to 2x the file size during processing.

### Security

- **CVE-2024-1234** (HIGH): Fixed SQL injection in the search endpoint.
  All users should upgrade. See [Security Advisory SA-2024-01](./security/SA-2024-01.md).

### Deprecations

- The `v1` API is now deprecated and will be removed in v3.0 (estimated
  Q3 2024). Migrate to `v2` endpoints. See the
  [v1 to v2 Migration Guide](./migration-v1-v2.md).
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| Breaking changes not documented in release notes | HIGH |
| Security fixes not flagged with severity | HIGH |
| Migration steps for breaking changes missing | HIGH |
| Release notes are a raw commit log | MEDIUM |
| New features described without usage context | MEDIUM |
| Bug fixes without symptom description or issue reference | MEDIUM |
| Deprecations without timeline or migration path | MEDIUM |
| No release notes for a released version | MEDIUM |
| Structured release notes with categorized, actionable entries | INFO (positive) |

---

## 9. API Documentation Completeness

**Category**: Completeness

API documentation is the contract between the product and its integrators. Incomplete API documentation means integrators cannot build reliable integrations without reverse-engineering the API through experimentation. For customer-facing APIs, documentation completeness is a hard requirement.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Every endpoint is documented with method, path, description | HIGH | Undocumented endpoints are invisible to integrators |
| Request parameters are documented with type, constraints, required/optional, and example | HIGH | Missing parameter docs force trial-and-error integration |
| Response schema is documented with all fields, types, and example | HIGH | Undocumented response fields are unreliable (may change without notice) |
| Error responses are documented per endpoint (not just globally) | MEDIUM | Different endpoints may have different error conditions |
| Authentication requirements are documented per endpoint | HIGH | Missing auth docs cause confusing 401/403 errors |
| Rate limits are documented per endpoint or per tier | MEDIUM | Undocumented rate limits cause production failures |
| Pagination is documented for list endpoints | HIGH | Unpaginated list endpoints hide data loss or performance cliffs |
| Versioning strategy is documented | MEDIUM | Integrators need to know how API versions are handled |
| SDK/client library examples are provided for major languages | MEDIUM | Raw HTTP examples are harder to use than language-specific examples |
| OpenAPI/Swagger spec is available and matches actual behavior | MEDIUM | Machine-readable specs enable code generation and automated testing |
| Webhook/event documentation covers payload schema, delivery guarantees, and retry behavior | MEDIUM | Webhook consumers need to build reliable handlers |

### Detection Patterns

**Undocumented endpoints**: Compare the API routes defined in the codebase against the documentation. Every route not in the docs is a gap.

```python
# Flask routes
@app.route("/api/v1/users", methods=["GET", "POST"])
@app.route("/api/v1/users/<id>", methods=["GET", "PUT", "DELETE"])
@app.route("/api/v1/users/<id>/roles", methods=["GET", "POST"])  # Documented?
@app.route("/api/v1/admin/stats", methods=["GET"])                # Documented?
```

**Incomplete endpoint documentation**: For each documented endpoint, verify:
```
[ ] Method and path
[ ] Description of what the endpoint does
[ ] Authentication requirements
[ ] Request headers (Content-Type, Authorization, custom headers)
[ ] Path parameters with type and description
[ ] Query parameters with type, default, and description
[ ] Request body schema with all fields, types, and constraints
[ ] Success response status code and body schema
[ ] Error response status codes and body schemas
[ ] Rate limit information
[ ] Example request and response
```

**Stale API documentation**: Compare the documented request/response schemas against the actual code models. Field additions, removals, or type changes that are not reflected in docs are findings.

### Common Mistakes

**Mistake**: Documenting the endpoint without showing a complete request/response example.

Bad:
```markdown
## Create User

`POST /api/v1/users`

Creates a new user. Requires authentication.

### Parameters

- `name` (string): The user's name
- `email` (string): The user's email
```

Good:
```markdown
## Create User

`POST /api/v1/users`

Creates a new user account. Requires `admin` role.

### Authentication

Requires a Bearer token with `admin` scope.

### Request

**Headers**:
- `Authorization: Bearer <token>` (required)
- `Content-Type: application/json` (required)

**Body**:
| Field | Type | Required | Constraints | Description |
|-------|------|----------|-------------|-------------|
| `name` | string | Yes | 1-100 characters | User's display name |
| `email` | string | Yes | Valid email format | User's email (must be unique) |
| `role` | string | No | One of: `viewer`, `editor`, `admin` | Default: `viewer` |

**Example**:
```bash
curl -X POST https://api.example.com/api/v1/users \
  -H "Authorization: Bearer eyJ..." \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Alice Smith",
    "email": "alice@example.com",
    "role": "editor"
  }'
```

### Response

**Success** (`201 Created`):
```json
{
  "id": "usr_abc123",
  "name": "Alice Smith",
  "email": "alice@example.com",
  "role": "editor",
  "created_at": "2024-01-15T14:23:45Z"
}
```

**Errors**:
| Status | Code | Cause |
|--------|------|-------|
| 400 | `VALIDATION_ERROR` | Missing required field or invalid format |
| 401 | `AUTH_TOKEN_MISSING` | No Authorization header |
| 403 | `AUTH_INSUFFICIENT_SCOPE` | Token does not have `admin` scope |
| 409 | `USER_EMAIL_EXISTS` | Email already registered |
| 429 | `RATE_LIMIT_EXCEEDED` | More than 10 create requests per minute |
```

**Mistake**: No pagination documentation for list endpoints.

Bad:
```markdown
## List Users

`GET /api/v1/users`

Returns all users.
```

What happens when there are 100,000 users? Does the endpoint return them all? Is there a limit? How do you get the next page?

Good:
```markdown
## List Users

`GET /api/v1/users`

Returns a paginated list of users.

### Query Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `page` | integer | 1 | Page number (1-indexed) |
| `per_page` | integer | 20 | Items per page (max: 100) |
| `sort` | string | `created_at` | Sort field. One of: `name`, `email`, `created_at` |
| `order` | string | `desc` | Sort order. One of: `asc`, `desc` |

### Response

```json
{
  "data": [
    {"id": "usr_abc123", "name": "Alice", "email": "alice@example.com"}
  ],
  "pagination": {
    "page": 1,
    "per_page": 20,
    "total_pages": 5,
    "total_items": 94
  }
}
```

To retrieve all pages, increment `page` until `page > total_pages`.
```

### Severity Guidance

| Context | Severity |
|---------|----------|
| API endpoints with no documentation | HIGH |
| Request parameters missing type, constraints, or required flag | HIGH |
| Response schema undocumented | HIGH |
| Authentication requirements not documented per endpoint | HIGH |
| Pagination undocumented for list endpoints | HIGH |
| Error responses not documented per endpoint | MEDIUM |
| Rate limits undocumented | MEDIUM |
| No request/response examples | MEDIUM |
| No OpenAPI/Swagger spec | MEDIUM |
| SDK examples not provided for major languages | MEDIUM |
| Complete API documentation with schemas, examples, and error docs | INFO (positive) |

---

## 10. Documentation Accessibility

**Category**: Accessibility

Documentation accessibility means ensuring that all users can access and use the documentation, regardless of how they consume it. This includes screen reader users, users with low vision, users on slow connections, users who search rather than browse, and users who access docs from different devices.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| All images have alt text that conveys the information in the image | MEDIUM | Screen reader users and broken-image scenarios need text alternatives |
| Diagrams have text descriptions or are supplementary to text explanations | MEDIUM | Information conveyed only through diagrams is inaccessible to some users |
| Documentation is searchable (search functionality exists or content is structured for grep/find) | MEDIUM | Users who cannot search must browse linearly, which is slow |
| Color is not the only way information is conveyed ("the red items are errors") | LOW | Color-blind users cannot distinguish color-only signals |
| Code blocks are selectable and copyable (not images of code) | HIGH | Code in images cannot be copied, searched, or read by screen readers |
| Documentation renders correctly on mobile devices | LOW | Developers increasingly read docs on phones during incidents |
| Heading structure enables navigation by screen readers and browser outline tools | LOW | Screen reader users navigate by headings; broken heading hierarchy breaks navigation |
| Links have descriptive text (not "click here" or bare URLs) | LOW | Screen reader users hear link text out of context; "click here" conveys nothing |
| Documentation is available offline or as a downloadable format when relevant | LOW | Users in air-gapped or low-connectivity environments need offline access |
| Language is clear and avoids unnecessary idioms or cultural references | LOW | Non-native English speakers are a significant portion of documentation readers |

### Detection Patterns

**Images without alt text**: Scan for image references missing alt text:
```markdown
![](diagram.png)              <- No alt text
![ ](diagram.png)             <- Empty alt text
![Screenshot](screenshot.png) <- Generic alt text; should describe what the screenshot shows
```

Good:
```markdown
![Architecture diagram showing the web server connected to the API gateway,
which routes to three backend microservices: auth, data, and notification.
All services connect to a shared PostgreSQL database.](architecture.png)
```

**Code as images**: Look for screenshots or images of terminal output or code. These should be replaced with actual code blocks.

**Color-dependent information**: Look for phrases like:
```
"items highlighted in red indicate errors"
"green status means healthy"
"see the blue section for configuration"
```

These should use text labels in addition to color:
```
"items marked with [ERROR] (highlighted in red) indicate errors"
"status: healthy (green) / degraded (yellow) / unhealthy (red)"
```

### Common Mistakes

**Mistake**: Architecture diagram with no text description.

Bad:
```markdown
## Architecture

![Architecture](architecture.png)
```

A screen reader user gets: "Image: Architecture." No information conveyed.

Good:
```markdown
## Architecture

The system consists of three tiers:

1. **Web tier**: Nginx reverse proxy handling TLS termination and static
   asset serving. Routes API requests to the application tier.
2. **Application tier**: Three instances of the API server behind an
   internal load balancer. Each instance connects to the database and
   cache independently.
3. **Data tier**: PostgreSQL primary with one synchronous replica for
   HA. Redis cluster for session storage and caching.

![Architecture diagram illustrating the three-tier layout described above.
Web tier (Nginx) at the top, application tier (3 API server instances)
in the middle, data tier (PostgreSQL + Redis) at the bottom. Arrows show
request flow from top to bottom.](architecture.png)
```

**Mistake**: Documentation only works in a specific rendering environment.

Bad: Documentation that relies on GitHub-specific rendering features (collapsible sections, mermaid diagrams, alerts/callouts) without fallback for users reading the raw Markdown or a different renderer.

Good: Use standard Markdown features as the baseline. When using platform-specific features, ensure the content is understandable without them (e.g., mermaid diagrams accompanied by text descriptions).

### Severity Guidance

| Context | Severity |
|---------|----------|
| Code presented as images (not copyable, not searchable) | HIGH |
| Documentation not searchable (no search feature, no structured content) | MEDIUM |
| Images missing alt text | MEDIUM |
| Diagrams with no text alternative | MEDIUM |
| Color as the only information channel | LOW |
| Documentation does not render on mobile | LOW |
| Broken heading hierarchy affecting screen reader navigation | LOW |
| "Click here" or bare URL links | LOW |
| Fully accessible documentation with alt text, text alternatives, and semantic structure | INFO (positive) |

---

## Quick-Reference Summary

| Category | Check | Severity |
|----------|-------|----------|
| Audience | No audience-specific entry points | HIGH |
| Audience | Important audience segment unserved | HIGH |
| Skill Level | Getting-started fails without product knowledge | HIGH |
| Content Types | No quickstart guide | HIGH |
| Content Types | Reference documentation incomplete | HIGH |
| Troubleshooting | No troubleshooting docs for customer product | HIGH |
| Troubleshooting | Top-10 support issues not covered | HIGH |
| Error Mapping | User-facing errors undocumented | HIGH |
| Limitations | Scale limits undocumented | HIGH |
| Limitations | Known functional limitations undocumented | HIGH |
| Limitations | Limitations not discoverable pre-adoption | HIGH |
| Operations | No production deployment documentation | HIGH |
| Operations | No monitoring guidance | HIGH |
| Operations | No backup/recovery procedure | HIGH |
| Operations | No upgrade procedure | HIGH |
| Operations | Infrastructure requirements undocumented | HIGH |
| Operations | HA deployment undocumented | HIGH |
| Operations | Security hardening missing | HIGH |
| Release Notes | Breaking changes not documented | HIGH |
| Release Notes | Security fixes not flagged | HIGH |
| API Docs | Endpoints undocumented | HIGH |
| API Docs | Request parameters incomplete | HIGH |
| API Docs | Response schema undocumented | HIGH |
| API Docs | Auth requirements missing per endpoint | HIGH |
| API Docs | Pagination undocumented for list endpoints | HIGH |
| Accessibility | Code as images | HIGH |
| Audience | Evaluator audience not considered | MEDIUM |
| Skill Level | Internal jargon in customer docs | MEDIUM |
| Skill Level | Concepts used before introduced | MEDIUM |
| Content Types | Quickstart covers too many features | MEDIUM |
| Content Types | No advanced guides for complex use cases | MEDIUM |
| Troubleshooting | Entries lack diagnostic steps | MEDIUM |
| Troubleshooting | Error messages not searchable (paraphrased) | MEDIUM |
| Error Mapping | Error codes exist but undocumented | MEDIUM |
| Error Mapping | Error responses lack context | MEDIUM |
| Limitations | Platform restrictions undocumented | MEDIUM |
| Limitations | Compatibility limitations undocumented | MEDIUM |
| Operations | Alerting thresholds not recommended | MEDIUM |
| Operations | Backup without recovery verification | MEDIUM |
| Release Notes | New features without usage context | MEDIUM |
| Release Notes | Bug fixes without symptom description | MEDIUM |
| Release Notes | Deprecations without timeline | MEDIUM |
| API Docs | Error responses not per-endpoint | MEDIUM |
| API Docs | Rate limits undocumented | MEDIUM |
| API Docs | No request/response examples | MEDIUM |
| Accessibility | Images missing alt text | MEDIUM |
| Accessibility | Diagrams with no text alternative | MEDIUM |
| Accessibility | Documentation not searchable | MEDIUM |

---

## Review Procedure Summary

When performing customer-facing documentation review:

1. **Identify audiences**: Determine all audience segments the product serves. Verify each has appropriate entry points and content.
2. **Assess skill-level calibration**: Check that each document maintains a consistent skill-level assumption. Flag beginner/expert level mixing.
3. **Verify content type separation**: Confirm quickstart, reference, and advanced guides exist as distinct content types with cross-references.
4. **Evaluate troubleshooting coverage**: Check for troubleshooting sections with symptom-cause-fix structure. Compare against known support issues.
5. **Audit error documentation**: Inventory user-facing error messages in the code. Cross-reference against documented error-to-resolution mappings.
6. **Review known limitations**: Verify that scale limits, platform restrictions, and compatibility limitations are documented and discoverable.
7. **Assess operational documentation**: Check for deployment, monitoring, backup, and upgrade documentation. Verify production-readiness of guidance.
8. **Evaluate release notes**: Check that release notes cover breaking changes, security fixes, and migration steps with appropriate urgency.
9. **Audit API documentation**: Cross-reference API endpoints in code against documentation. Verify schema completeness for request, response, and error.
10. **Check accessibility**: Verify alt text, text alternatives for diagrams, code as text (not images), and searchability.
11. **Classify each finding** using the severity tables above, adjusting for product maturity and customer impact.
