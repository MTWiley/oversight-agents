# CI/CD Patterns

Reference checklist for the `devops-reviewer` agent when evaluating CI/CD pipelines. This file complements the inline criteria in `review-devops.md` with deeper guidance, severity classifications, common mistakes, and fix patterns.

Use this checklist during both diff-based and full-repo reviews. Each section maps to a category in the finding schema.

---

## 1. Pipeline Structure

**Category**: CI/CD Pipeline

Well-structured pipelines are readable, maintainable, and predictable. Poor structure leads to hidden dependencies between jobs, accidental deployments, and pipelines that no one dares to modify.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Stages/jobs have clear, descriptive names | LOW | Readability and debugging |
| Logical stage ordering (lint -> test -> build -> deploy) | LOW | Convention and predictability |
| Triggers match branch protection rules | MEDIUM | Misaligned triggers can skip required checks |
| PR/merge request pipelines are separate from deployment pipelines | MEDIUM | Prevents accidental deploys from feature branches |
| Manual approval gates exist before production deployment | HIGH | Missing gates allow unchecked production changes |
| Pipeline definitions use reusable components (templates, composite actions, shared libraries) | MEDIUM | Duplication leads to configuration drift |
| No orphaned or dead jobs (jobs that never trigger) | LOW | Dead code in pipelines causes confusion |

### Common Mistakes

**Mistake**: Using the same pipeline for PRs and production deployment with only branch-name conditionals.

Why it is a problem: A misconfigured conditional can send a feature branch straight to production. Separate pipeline definitions (or clearly separated workflows) are safer.

**Mistake**: Naming jobs `job1`, `job2`, `test`, or `build` without context.

Why it is a problem: When a pipeline fails at 2 AM, the on-call engineer needs to understand what failed from the notification alone. `lint-python` is actionable; `job2` is not.

**Mistake**: No `workflow_dispatch` or manual trigger for deployment pipelines.

Why it is a problem: Without a manual trigger, there is no way to re-deploy a known-good version without pushing a new commit or reverting.

### Fix Patterns

**Reusable workflows (GitHub Actions)**:

```yaml
# .github/workflows/reusable-test.yml
on:
  workflow_call:
    inputs:
      python-version:
        required: true
        type: string
jobs:
  test:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-python@v5
        with:
          python-version: ${{ inputs.python-version }}
      - run: pip install -r requirements.txt
      - run: pytest
```

```yaml
# .github/workflows/ci.yml
on:
  pull_request:
jobs:
  test-3-11:
    uses: ./.github/workflows/reusable-test.yml
    with:
      python-version: "3.11"
  test-3-12:
    uses: ./.github/workflows/reusable-test.yml
    with:
      python-version: "3.12"
```

**Trigger alignment with branch protection**:

```yaml
# Ensures CI runs on PRs to protected branches
on:
  pull_request:
    branches: [main, release/*]
  push:
    branches: [main]  # Post-merge validation
```

---

## 2. Security in CI

**Category**: Secrets Management, CI/CD Pipeline

CI/CD pipelines are a high-value target. A compromised pipeline can exfiltrate secrets, inject malicious code into builds, or deploy backdoored artifacts. Security in CI is not optional -- it is foundational.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| No secrets, tokens, or passwords hardcoded in workflow files | CRITICAL | Direct credential exposure |
| Secrets use the platform's secret management (GitHub Secrets, GitLab CI Variables, Jenkins Credentials) | HIGH | Plaintext secrets in config are extractable |
| Secrets are scoped to environments, not globally available to all workflows | MEDIUM | Limits blast radius of a compromised workflow |
| Dependency vulnerability scanning is present (Dependabot, Snyk, Trivy, Grype, `npm audit`, `pip-audit`) | HIGH | Known vulnerabilities in dependencies are a top attack vector |
| SAST tool is integrated (Semgrep, CodeQL, Bandit, ESLint security rules) | HIGH | Catches security bugs before they reach production |
| Container image scanning is in the pipeline (Trivy, Grype, Snyk Container) | HIGH | Base image vulnerabilities propagate to all deployments |
| Third-party actions/plugins are from verified publishers or pinned to a commit SHA | MEDIUM | Supply chain attacks via compromised actions |
| `GITHUB_TOKEN` permissions are restricted (not default `write-all`) | MEDIUM | Principle of least privilege for CI tokens |
| Workflow files are protected from modification by non-maintainers | MEDIUM | Workflow injection via PR from fork |
| No `pull_request_target` with checkout of PR head (GitHub Actions) | HIGH | Known code injection vector |
| Secrets are not printed or echoed in logs | HIGH | Log exposure of credentials |

### Common Mistakes

**Mistake**: Using `pull_request_target` with `actions/checkout` referencing the PR branch.

Why it is a problem: `pull_request_target` runs with write permissions and access to secrets, even for PRs from forks. If the checkout fetches the fork's code, that code runs with elevated privileges. This is a well-documented code injection vector.

```yaml
# DANGEROUS - do not do this
on: pull_request_target
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          ref: ${{ github.event.pull_request.head.sha }}  # Fork code with secrets access
      - run: make build  # Runs attacker-controlled code
```

**Mistake**: Setting `permissions: write-all` or not restricting `GITHUB_TOKEN` permissions.

```yaml
# BAD - overly broad permissions
permissions: write-all

# GOOD - minimal permissions
permissions:
  contents: read
  pull-requests: write
```

**Mistake**: Secrets passed as command-line arguments, which appear in process listings.

```bash
# BAD - secret visible in ps output and logs
curl -u admin:${{ secrets.API_TOKEN }} https://api.example.com

# BETTER - secret via environment variable
env:
  API_TOKEN: ${{ secrets.API_TOKEN }}
run: |
  curl -u admin:${API_TOKEN} https://api.example.com
```

**Mistake**: No dependency scanning at all.

Why it is a problem: Known CVEs in direct and transitive dependencies are the most common exploitable vulnerability class. Automated scanning catches what manual review cannot.

### Fix Patterns

**Minimal GitHub Actions permissions**:

```yaml
permissions:
  contents: read
  checks: write       # Only if writing check results
  pull-requests: write # Only if commenting on PRs
```

**Dependency scanning (GitHub Actions)**:

```yaml
- name: Run Trivy vulnerability scanner
  uses: aquasecurity/trivy-action@0.28.0
  with:
    scan-type: fs
    scan-ref: .
    severity: CRITICAL,HIGH
    exit-code: 1  # Fail the pipeline on findings
```

**Image scanning**:

```yaml
- name: Build image
  run: docker build -t myapp:${{ github.sha }} .

- name: Scan image
  uses: aquasecurity/trivy-action@0.28.0
  with:
    image-ref: myapp:${{ github.sha }}
    severity: CRITICAL,HIGH
    exit-code: 1
```

---

## 3. Build Reproducibility

**Category**: Build Reproducibility

A build is reproducible when the same commit produces the same artifact regardless of when or where it is built. Without reproducibility, debugging production issues becomes guesswork, rollbacks may not actually restore the previous state, and compliance audits cannot verify what was deployed.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Lockfiles are committed (`package-lock.json`, `poetry.lock`, `go.sum`, `Gemfile.lock`, `Cargo.lock`) | HIGH | Without lockfiles, `install` pulls whatever is newest |
| Dependency versions are pinned in lockfiles, not just in manifests | MEDIUM | Manifests with `^` or `~` ranges are not deterministic |
| Third-party CI actions/plugins are pinned to a SHA or exact version tag | MEDIUM | `@main` or `@latest` can change without notice |
| Build tool and runtime versions are locked (`.tool-versions`, `.node-version`, `.python-version`, `.go-version`) | MEDIUM | Different runtime versions produce different behavior |
| Base images in Dockerfiles are pinned to a digest or specific version tag | MEDIUM | `:latest` is mutable and changes without warning |
| Build commands do not fetch external resources at build time without version pinning | MEDIUM | `curl | bash` in CI is not reproducible |
| CI runner images are pinned or controlled | LOW | `ubuntu-latest` changes quarterly |
| Build timestamps are not baked into artifact identifiers (unless also tagged with commit SHA) | LOW | Timestamp-only versioning prevents reproducibility verification |

### Common Mistakes

**Mistake**: Lockfile not committed; `.gitignore` includes `package-lock.json` or `poetry.lock`.

Why it is a problem: Every build resolves dependencies independently, pulling whatever the registry serves at that moment. A Tuesday build and a Thursday build of the same commit can produce different artifacts.

**Mistake**: Pinning CI actions to a mutable tag instead of a SHA.

```yaml
# BAD - v4 is a moving target
- uses: actions/checkout@v4

# BETTER - SHA is immutable (but note the version in a comment for readability)
- uses: actions/checkout@b4ffde65f46336ab88eb53be808477a3936bae11 # v4.1.1
```

Note: Pinning to a major version tag like `@v4` is acceptable for well-maintained first-party actions (e.g., `actions/checkout`, `actions/setup-node`) where the risk is low and the maintenance burden of SHA pinning is high. Flag `@main` or `@latest` as MEDIUM; flag `@v4` as INFO with a note about SHA pinning for higher assurance.

**Mistake**: Using `curl | bash` or `wget -O- | sh` in CI to install tools.

```bash
# BAD - no version pinning, no integrity check
curl -sSL https://install.example.com | bash

# BETTER - pinned version with checksum verification
curl -sSL -o tool.tar.gz https://releases.example.com/v1.2.3/tool-linux-amd64.tar.gz
echo "abc123expectedsha256  tool.tar.gz" | sha256sum -c
tar xzf tool.tar.gz
```

### Fix Patterns

**Runtime version locking with `.tool-versions` (asdf)**:

```
nodejs 20.11.0
python 3.12.1
terraform 1.7.0
```

**Dockerfile base image pinning**:

```dockerfile
# Pin to digest for maximum reproducibility
FROM node:20.11.0-alpine3.19@sha256:abcdef1234567890...

# Acceptable alternative: pin to specific version tag
FROM node:20.11.0-alpine3.19
```

---

## 4. Deployment Gates

**Category**: Deployment

Deployment gates prevent untested or unapproved changes from reaching production. Missing gates are the difference between "we deploy confidently" and "we deploy and pray."

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Production deployments require manual approval | HIGH | Prevents automated push of broken code to production |
| A staging or pre-production environment exists in the pipeline | HIGH | No staging means production is the first real test |
| Smoke tests or integration tests run after deployment | MEDIUM | Deploys without verification leave failures undetected |
| Rollback mechanism is defined (blue-green, canary, revert workflow) | HIGH | No rollback plan = extended outage on bad deploy |
| Database migrations are reversible or have a rollback script | MEDIUM | Irreversible schema changes are a single point of failure |
| Feature flags are used for risky changes | LOW | Flags allow disabling new features without redeploying |
| Deployment notifications are sent to the team (Slack, email, webhook) | LOW | Team awareness prevents conflicting deployments |
| Production deployment windows are enforced (no Friday deploys without override) | LOW | Convention enforcement reduces risk |

### Common Mistakes

**Mistake**: CI/CD pipeline pushes to production on every merge to `main` with no approval step.

Why it is a problem: A single merged PR with a subtle bug goes straight to production. With an approval gate, the deployer makes a conscious decision and can verify staging first.

**Mistake**: No staging environment -- the pipeline goes from tests to production.

Why it is a problem: Unit and integration tests cannot catch environment-specific issues (network policies, IAM permissions, external service behavior). Staging catches the class of bugs that only manifest in a deployed context.

**Mistake**: No rollback mechanism beyond "revert the commit and re-deploy."

Why it is a problem: Reverting a commit and redeploying takes minutes to hours. Blue-green or canary deployments allow instant rollback by switching traffic. For database changes, a revert may not even be possible without a migration rollback script.

### Fix Patterns

**GitHub Actions environment protection**:

```yaml
jobs:
  deploy-staging:
    environment: staging
    steps:
      - run: ./deploy.sh staging

  smoke-test:
    needs: deploy-staging
    steps:
      - run: ./smoke-tests.sh staging

  deploy-production:
    needs: smoke-test
    environment:
      name: production
      url: https://app.example.com
    # GitHub enforces required reviewers configured on the environment
    steps:
      - run: ./deploy.sh production

  verify-production:
    needs: deploy-production
    steps:
      - run: ./smoke-tests.sh production
```

**GitLab CI manual gate**:

```yaml
deploy-production:
  stage: deploy
  when: manual
  environment:
    name: production
  only:
    - main
  script:
    - ./deploy.sh production
```

---

## 5. Artifact Management

**Category**: CI/CD Pipeline

Build artifacts (container images, binaries, packages) are the bridge between CI and deployment. If artifacts are not signed, checksummed, and stored properly, the supply chain is vulnerable and auditing becomes impossible.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Artifacts are stored in a dedicated registry or artifact store (not just CI cache) | MEDIUM | CI caches are ephemeral; artifacts must persist for rollbacks and audits |
| Container images are pushed to a private registry, not only Docker Hub public | MEDIUM | Public registries expose internal application images |
| Artifacts are tagged with the commit SHA or a unique build identifier | MEDIUM | Traceability from deployed artifact back to source commit |
| Artifact signing is implemented (cosign, Notary, GPG) | MEDIUM | Prevents deployment of tampered artifacts |
| Checksums are generated and stored alongside artifacts | LOW | Integrity verification for downloads and deployments |
| Artifact retention policies are defined | LOW | Prevents unbounded storage growth and cost |
| Old/vulnerable artifacts are cleaned up or marked as deprecated | LOW | Stale artifacts with known CVEs should not be deployable |
| SBOM (Software Bill of Materials) is generated | LOW | Supply chain transparency; increasingly a compliance requirement |

### Common Mistakes

**Mistake**: Tagging container images only with `latest` or branch name.

Why it is a problem: Mutable tags make it impossible to know exactly which build is running. If `latest` is redeployed, the previous version is gone. Tag with the commit SHA and optionally also with a human-readable tag.

```bash
# BAD
docker build -t myapp:latest .
docker push myapp:latest

# GOOD
docker build -t myapp:${GITHUB_SHA} -t myapp:latest .
docker push myapp:${GITHUB_SHA}
docker push myapp:latest
```

**Mistake**: No artifact signing -- anyone with registry write access can push.

Why it is a problem: Without signing, a compromised CI pipeline or registry credential can push a malicious image that is indistinguishable from a legitimate one. Signing with cosign or Notary creates a verifiable chain of custody.

### Fix Patterns

**Image signing with cosign (GitHub Actions)**:

```yaml
- name: Sign image
  uses: sigstore/cosign-installer@v3
- run: cosign sign --yes ${REGISTRY}/myapp:${GITHUB_SHA}
  env:
    COSIGN_EXPERIMENTAL: "true"  # Keyless signing with OIDC
```

**SBOM generation**:

```yaml
- name: Generate SBOM
  uses: anchore/sbom-action@v0
  with:
    image: myapp:${GITHUB_SHA}
    format: spdx-json
    output-file: sbom.spdx.json
```

---

## 6. Pipeline Performance

**Category**: CI/CD Pipeline

Slow pipelines kill developer productivity and encourage people to skip CI. A pipeline that takes 30 minutes to run will be bypassed; one that takes 5 minutes will be used consistently.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Independent jobs run in parallel, not sequentially | LOW | Unnecessary sequential execution wastes time |
| Dependency caching is configured (npm, pip, Maven, Go modules) | MEDIUM | Re-downloading dependencies on every run is wasteful |
| Docker layer caching is used for image builds | LOW | Rebuilding all layers on every push is slow |
| Path filters or change detection prevents unnecessary pipeline runs | MEDIUM | In monorepos, running all tests for a docs change is wasteful |
| Test splitting or parallelization is configured for large test suites | LOW | A 20-minute test suite split across 4 runners finishes in 5 minutes |
| Pipelines avoid redundant work (e.g., building the same image twice) | LOW | Repeated work increases cost and time |
| CI runner selection is appropriate (self-hosted for heavy builds, hosted for light jobs) | LOW | Overprovisioned or underprovisioned runners affect cost and speed |

### Common Mistakes

**Mistake**: All jobs run sequentially even when they have no dependencies.

```yaml
# BAD - sequential when lint and test are independent
jobs:
  lint:
    ...
  test:
    needs: lint  # Test does not actually depend on lint output
    ...

# GOOD - parallel execution
jobs:
  lint:
    ...
  test:
    ...  # No needs: runs in parallel with lint
  deploy:
    needs: [lint, test]  # Only deploy depends on both
```

**Mistake**: No dependency caching.

```yaml
# BAD - installs from scratch every time
steps:
  - run: npm install

# GOOD - cache node_modules
steps:
  - uses: actions/setup-node@v4
    with:
      node-version: 20
      cache: npm
  - run: npm ci
```

### Fix Patterns

**Path filters (GitHub Actions)**:

```yaml
on:
  push:
    paths:
      - 'src/**'
      - 'tests/**'
      - 'package.json'
      - 'package-lock.json'
      - '.github/workflows/ci.yml'
    # Excludes: docs changes, README, etc.
```

**Docker layer caching**:

```yaml
- uses: docker/build-push-action@v5
  with:
    context: .
    cache-from: type=gha
    cache-to: type=gha,mode=max
```

---

## 7. Platform-Specific Checks

**Category**: CI/CD Pipeline

Each CI platform has its own footguns and best practices. This section covers platform-specific issues that the general checks above do not address.

### GitHub Actions

| Check | Severity | Rationale |
|-------|----------|-----------|
| `permissions` block is set at workflow or job level | MEDIUM | Default token has broad permissions |
| `pull_request_target` is not used with PR head checkout | HIGH | Code injection vector (see Section 2) |
| Reusable workflows are used to reduce duplication | LOW | Maintainability |
| `concurrency` is set to cancel in-progress runs on new push | LOW | Prevents queue buildup and wasted compute |
| Action versions use Dependabot or Renovate for updates | LOW | Stale actions miss security patches |
| `CODEOWNERS` file aligns with workflow protection | LOW | Ensures the right people review workflow changes |

**Concurrency pattern**:

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true  # Cancel previous runs for the same branch
```

### GitLab CI

| Check | Severity | Rationale |
|-------|----------|-----------|
| `include` and `extends` are used for template reuse | LOW | Reduces duplication |
| Protected variables are used for production secrets | HIGH | Unprotected variables are available to all branches |
| `rules` are used instead of `only/except` (deprecated) | LOW | Modern syntax is more flexible and readable |
| `needs` keyword is used for DAG-based execution | LOW | Allows parallel execution within stages |
| Pipeline triggers are scoped (not overly broad) | MEDIUM | Prevents unnecessary pipeline runs |
| `artifacts:expire_in` is set | LOW | Prevents unbounded artifact storage |

### Jenkins

| Check | Severity | Rationale |
|-------|----------|-----------|
| Pipeline is defined in `Jenkinsfile` (Pipeline as Code), not configured in the UI | MEDIUM | UI-configured jobs are not version controlled |
| Shared libraries are version-pinned | MEDIUM | `@main` shared libraries break builds unpredictably |
| Credentials are managed via Jenkins Credentials, not hardcoded | CRITICAL | Plaintext credentials in Jenkinsfile are a critical exposure |
| Agents/nodes use labels, not hardcoded node names | LOW | Hardcoded nodes break when infrastructure changes |
| `Jenkinsfile` uses declarative syntax (preferred) or well-structured scripted syntax | LOW | Declarative pipelines are more readable and auditable |
| Credential scoping uses `withCredentials` block | MEDIUM | Limits credential exposure to the minimum required scope |

**Jenkins credentials pattern**:

```groovy
// BAD
sh "curl -u admin:${PASSWORD} https://api.example.com"

// GOOD
withCredentials([usernamePassword(
    credentialsId: 'api-creds',
    usernameVariable: 'API_USER',
    passwordVariable: 'API_PASS'
)]) {
    sh 'curl -u ${API_USER}:${API_PASS} https://api.example.com'
}
```

---

## Quick Reference: Severity Summary

| Severity | CI/CD Findings |
|----------|---------------|
| CRITICAL | Hardcoded secrets in pipeline files; credentials in Jenkins UI or Jenkinsfile without credential store |
| HIGH | No dependency/image scanning; `pull_request_target` code injection risk; no deployment approval gates; no staging environment; no rollback mechanism; secrets not scoped; unprotected GitLab CI variables |
| MEDIUM | Unpinned action/plugin versions; no dependency caching; overly broad token permissions; no path filters in monorepo; lockfile not committed; triggers misaligned with branch protection; pipeline duplication |
| LOW | Sequential jobs that could be parallel; no concurrency control; naming inconsistencies; missing artifact retention; missing SBOM generation; no deployment notifications |
| INFO | Well-structured reusable workflows; good caching strategy; effective use of deployment environments; proper artifact signing |
