# Infrastructure as Code Hygiene

Reference checklist for the `devops-reviewer` agent when evaluating Infrastructure as Code. This file complements the inline criteria in `review-devops.md` with deeper guidance, severity classifications, common mistakes, and fix patterns.

The primary focus is Terraform and OpenTofu. Notes for CloudFormation and Pulumi are included where patterns diverge significantly.

---

## 1. State Management

**Category**: IaC State

Terraform state is the source of truth for what infrastructure exists. Misconfigured state management leads to state corruption, concurrent modification conflicts, and accidental destruction of resources.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Remote state backend is configured (S3, GCS, Azure Blob, Terraform Cloud) | HIGH | Local state files are lost with the machine and cannot be shared |
| State locking is enabled (DynamoDB for S3, native for GCS/Azure Blob) | HIGH | Without locking, concurrent `apply` operations corrupt state |
| State encryption at rest is enabled | MEDIUM | State contains resource attributes, sometimes including secrets |
| `*.tfstate` and `*.tfstate.backup` are in `.gitignore` | HIGH | State files contain sensitive data and must not be committed |
| State backend configuration does not contain hardcoded credentials | CRITICAL | Backend credentials in code are equivalent to hardcoded secrets |
| State is segmented per environment (separate state files or workspaces) | MEDIUM | A single state file for all environments means one `destroy` affects everything |
| State is not manually edited | MEDIUM | Manual state edits bypass Terraform's reconciliation logic |
| `terraform state` commands in CI have appropriate safeguards | MEDIUM | Automated state manipulation without review is risky |

### Common Mistakes

**Mistake**: Using local state in a team or CI/CD context.

Why it is a problem: Local state lives on a single machine. If two people run `terraform apply` concurrently, one will overwrite the other's changes. If the machine is lost, the state is gone and Terraform no longer knows what it manages.

**Mistake**: Remote state without locking.

```hcl
# BAD - S3 backend without DynamoDB locking
terraform {
  backend "s3" {
    bucket = "my-tf-state"
    key    = "prod/terraform.tfstate"
    region = "us-east-1"
  }
}

# GOOD - S3 backend with DynamoDB locking
terraform {
  backend "s3" {
    bucket         = "my-tf-state"
    key            = "prod/terraform.tfstate"
    region         = "us-east-1"
    dynamodb_table = "terraform-locks"
    encrypt        = true
  }
}
```

**Mistake**: State files committed to version control.

Why it is a problem: Terraform state often contains sensitive attributes (database passwords, private keys, access tokens) in plaintext. Once committed, these are in the repository history permanently even if later removed.

### Fix Patterns

**.gitignore for Terraform**:

```gitignore
# Terraform state
*.tfstate
*.tfstate.*
.terraform/
.terraform.lock.hcl  # Some teams commit this; either is fine if consistent

# Crash logs
crash.log
crash.*.log

# Variable files that may contain secrets
*.tfvars
!terraform.tfvars.example

# Override files
override.tf
override.tf.json
*_override.tf
*_override.tf.json
```

Note: Whether to commit `.terraform.lock.hcl` is a team decision. Committing it ensures provider version reproducibility across the team. Not committing it avoids merge conflicts when multiple people update providers. Flag the absence of a decision (neither committed nor gitignored) as LOW.

---

## 2. Module Design

**Category**: IaC Hygiene

Modules are the unit of reuse in Terraform. Poor module design leads to copy-paste sprawl, inconsistent infrastructure, and a codebase where changing one thing requires touching twenty files.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Reusable patterns are extracted into modules, not duplicated | MEDIUM | Copy-paste infrastructure drifts over time |
| Module sources are pinned to a version tag, not `main` or `master` | MEDIUM | Unpinned modules change behavior without notice |
| Registry modules use version constraints | MEDIUM | `version = ">= 1.0"` can pull breaking changes |
| Variables have `description` attributes | LOW | Self-documenting modules reduce onboarding friction |
| Variables have `type` constraints | LOW | Type checking catches misconfigurations early |
| Variables use `validation` blocks for business rules | MEDIUM | Prevents invalid values from reaching the provider |
| Outputs have `description` attributes | LOW | Output discoverability |
| Sensitive outputs are marked with `sensitive = true` | HIGH | Prevents accidental logging of secrets |
| Module composition is preferred over monolithic modules | LOW | Small, composable modules are easier to test and reuse |
| Module README or inline documentation exists | LOW | Consumers need to understand inputs, outputs, and behavior |

### Common Mistakes

**Mistake**: Copy-pasting resource blocks across environments instead of using modules.

Why it is a problem: When a security fix is needed (e.g., enabling encryption), it must be applied in every copy. Missed copies become vulnerabilities.

**Mistake**: Referencing a module from a Git repository without a version pin.

```hcl
# BAD - source points to default branch
module "vpc" {
  source = "git::https://github.com/org/terraform-modules.git//vpc"
}

# GOOD - pinned to a specific tag
module "vpc" {
  source = "git::https://github.com/org/terraform-modules.git//vpc?ref=v2.3.1"
}

# ALSO GOOD - Terraform Registry with version constraint
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "5.4.0"
}
```

**Mistake**: No variable validation for values that have known constraints.

```hcl
# BAD - accepts any string
variable "environment" {
  type = string
}

# GOOD - validates against allowed values
variable "environment" {
  type        = string
  description = "Deployment environment"

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "Environment must be one of: dev, staging, prod."
  }
}
```

### Fix Patterns

**Module structure convention**:

```
modules/
  vpc/
    main.tf          # Resources
    variables.tf     # Input variables
    outputs.tf       # Output values
    versions.tf      # Required providers and versions
    README.md        # Usage documentation
```

**Marking sensitive outputs**:

```hcl
output "database_password" {
  value       = aws_db_instance.main.password
  description = "The master password for the RDS instance"
  sensitive   = true
}
```

---

## 3. Resource Management

**Category**: IaC Hygiene

How resources are named, tagged, and lifecycle-managed determines whether infrastructure is auditable, cost-attributable, and resistant to accidental destruction.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| All resources are tagged with standard tags (environment, team/owner, project, managed-by) | MEDIUM | Untagged resources are unattributable for cost and ownership |
| Naming conventions are consistent across resources | LOW | Inconsistent names make automation and searching harder |
| `lifecycle { prevent_destroy = true }` is set on stateful resources (databases, storage, state buckets) | HIGH | Protects against accidental `terraform destroy` |
| `lifecycle { create_before_destroy = true }` is used where downtime is unacceptable | MEDIUM | Default destroy-then-create causes outage windows |
| Data sources are used instead of hardcoded IDs, ARNs, or IP addresses | MEDIUM | Hardcoded identifiers break when infrastructure changes |
| Resources use `depends_on` only when implicit dependencies are insufficient | LOW | Overuse of `depends_on` hides dependency issues |
| `count` and `for_each` are used instead of duplicated resource blocks | MEDIUM | Duplicate blocks are error-prone and hard to maintain |
| Resource imports are documented when adopting existing infrastructure | LOW | Import context helps future maintainers understand state |

### Common Mistakes

**Mistake**: No lifecycle protection on production databases.

```hcl
# BAD - a mistyped terraform destroy removes the database
resource "aws_db_instance" "production" {
  # ...
}

# GOOD - protected from accidental destruction
resource "aws_db_instance" "production" {
  # ...

  lifecycle {
    prevent_destroy = true
  }
}
```

**Mistake**: Hardcoded AMI IDs, VPC IDs, or subnet IDs.

```hcl
# BAD - breaks when AMI is deprecated or in a different region
resource "aws_instance" "app" {
  ami           = "ami-0abcdef1234567890"
  instance_type = "t3.medium"
}

# GOOD - dynamically resolves the current AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]
  }
}

resource "aws_instance" "app" {
  ami           = data.aws_ami.amazon_linux.id
  instance_type = "t3.medium"
}
```

**Mistake**: No tagging strategy or inconsistent tags.

```hcl
# BAD - no tags
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"
}

# GOOD - standard tags applied via default_tags or per resource
resource "aws_s3_bucket" "data" {
  bucket = "my-data-bucket"

  tags = {
    Environment = var.environment
    Team        = "platform"
    Project     = "data-pipeline"
    ManagedBy   = "terraform"
  }
}
```

### Fix Patterns

**Provider-level default tags (AWS)**:

```hcl
provider "aws" {
  region = var.region

  default_tags {
    tags = {
      Environment = var.environment
      Team        = var.team
      ManagedBy   = "terraform"
      Repository  = "github.com/org/infra"
    }
  }
}
```

---

## 4. Security

**Category**: IaC Policy

IaC security is about ensuring that the infrastructure defined in code enforces least privilege, encrypts data, and does not create overly permissive access paths.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| IAM policies follow least privilege (no `Action: "*"` or `Resource: "*"`) | HIGH | Wildcard permissions grant far more access than intended |
| IAM policies use specific resource ARNs, not wildcards | MEDIUM | Resource wildcards allow access to unintended resources |
| Security groups and firewall rules are restrictive (no `0.0.0.0/0` on management ports) | CRITICAL | Open management ports are directly exploitable |
| Encryption is enabled for storage (S3, EBS, RDS, GCS) | HIGH | Unencrypted storage is a compliance and security failure |
| Encryption keys use customer-managed keys (CMK) for sensitive workloads | MEDIUM | AWS-managed keys are acceptable for many workloads; CMK for sensitive data |
| Sensitive values are not in `.tf` files or `.tfvars` committed to the repo | CRITICAL | Secrets in version control are permanently exposed |
| State encryption is enabled (see Section 1) | MEDIUM | State contains sensitive resource attributes |
| Outputs containing secrets are marked `sensitive = true` | HIGH | Prevents leaking secrets in CLI output and CI logs |
| Public access is explicitly disabled where not required (S3 public access block, RDS public accessibility) | HIGH | Default-public resources are a common misconfiguration |
| VPC flow logs or cloud audit logging is enabled | MEDIUM | Without logging, security incidents cannot be investigated |

### Common Mistakes

**Mistake**: Wildcard IAM permissions.

```hcl
# BAD - grants all actions on all resources
resource "aws_iam_policy" "app" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = "*"
      Resource = "*"
    }]
  })
}

# GOOD - specific actions on specific resources
resource "aws_iam_policy" "app" {
  policy = jsonencode({
    Statement = [{
      Effect   = "Allow"
      Action   = [
        "s3:GetObject",
        "s3:PutObject"
      ]
      Resource = "arn:aws:s3:::my-bucket/app-data/*"
    }]
  })
}
```

**Mistake**: Security groups open to `0.0.0.0/0` on SSH or RDP.

```hcl
# BAD - SSH open to the internet
resource "aws_security_group_rule" "ssh" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = ["0.0.0.0/0"]
  security_group_id = aws_security_group.app.id
}

# GOOD - SSH restricted to bastion or VPN CIDR
resource "aws_security_group_rule" "ssh" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = [var.vpn_cidr]
  security_group_id = aws_security_group.app.id
}
```

**Mistake**: S3 bucket without public access block.

```hcl
# GOOD - explicitly block public access
resource "aws_s3_bucket_public_access_block" "data" {
  bucket = aws_s3_bucket.data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

### Fix Patterns

**RDS encryption and restricted access**:

```hcl
resource "aws_db_instance" "production" {
  engine               = "postgres"
  instance_class       = "db.r6g.large"
  storage_encrypted    = true
  kms_key_id           = aws_kms_key.rds.arn
  publicly_accessible  = false

  vpc_security_group_ids = [aws_security_group.rds.id]

  lifecycle {
    prevent_destroy = true
  }
}
```

---

## 5. Code Quality

**Category**: IaC Hygiene

Terraform code quality directly affects maintainability and the confidence with which teams make changes. Formatting, validation, and structural consistency reduce the risk of misconfigurations.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| `terraform fmt` is enforced in CI (formatting check) | LOW | Consistent formatting reduces diff noise |
| `terraform validate` runs in CI | MEDIUM | Catches syntax and configuration errors before plan |
| A `terraform plan` step exists in CI for PRs | MEDIUM | Plan output shows what will change before apply |
| `terraform plan` output is posted to PRs for review | LOW | Reviewers can see infrastructure impact without running locally |
| Provider versions are constrained in `versions.tf` | MEDIUM | Unconstrained providers pull latest, risking breaking changes |
| Terraform version is constrained via `required_version` | MEDIUM | Different Terraform versions can produce different plans |
| Files follow conventional structure (`main.tf`, `variables.tf`, `outputs.tf`, `versions.tf`) | LOW | Convention aids navigation |
| No `terraform.tfvars` with real values committed (use `.tfvars.example`) | HIGH | Tfvars files often contain environment-specific secrets |
| HCL is used over JSON format for readability | LOW | JSON-formatted Terraform is harder to review |
| Comments explain non-obvious design decisions | LOW | Why a CIDR is `/24` or why a specific instance type was chosen |

### Common Mistakes

**Mistake**: No provider version constraint.

```hcl
# BAD - uses whatever provider version is installed
provider "aws" {
  region = var.region
}

# GOOD - constrained provider version
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.30"
    }
  }
}
```

**Mistake**: No CI validation step -- code goes from PR to apply without automated checks.

A minimal Terraform CI pipeline should run: `fmt -check`, `init`, `validate`, and `plan`.

### Fix Patterns

**Terraform CI pipeline (GitHub Actions)**:

```yaml
name: Terraform
on:
  pull_request:
    paths:
      - 'terraform/**'

jobs:
  validate:
    runs-on: ubuntu-22.04
    defaults:
      run:
        working-directory: terraform
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
        with:
          terraform_version: 1.7.0

      - name: Format check
        run: terraform fmt -check -recursive

      - name: Init
        run: terraform init -backend=false

      - name: Validate
        run: terraform validate

      - name: Plan
        run: terraform plan -no-color
        env:
          AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
          AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
```

---

## 6. Environment Separation

**Category**: IaC Hygiene

Mixing environments in a single state or configuration creates a blast radius problem: a mistake targeting development can destroy production. Proper separation ensures that environments are isolated and independently manageable.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Environments use separate state files or state backends | HIGH | Shared state means one `destroy` affects all environments |
| Workspaces are used appropriately (not as a substitute for separate state backends in production) | MEDIUM | Workspaces share the same backend; separate accounts/projects are stronger isolation |
| Environment-specific variables are isolated (separate `.tfvars`, variable files, or directory structures) | MEDIUM | Variable bleed between environments causes misconfigurations |
| Production uses a separate cloud account, project, or subscription | MEDIUM | Account-level isolation is the strongest boundary |
| Drift detection is configured or scheduled | LOW | Undetected drift means reality diverges from code |
| Changes to production are gated by plan review | HIGH | Applying without reviewing the plan risks unintended changes |
| Terraform workspaces are not the sole mechanism for environment separation in production | MEDIUM | Workspaces share backend credentials; a leaked credential compromises all environments |

### Common Mistakes

**Mistake**: A single state file for dev, staging, and production.

Why it is a problem: `terraform destroy` in the wrong context deletes production. `terraform apply` with the wrong variable file deploys dev settings to production.

**Mistake**: Using only Terraform workspaces for production isolation.

Why it is a problem: Workspaces share the same backend and credentials. If the backend credential is compromised, all environments are compromised. Workspaces are fine for non-production environments; production should have a separate backend.

### Fix Patterns

**Directory-based environment separation**:

```
terraform/
  modules/           # Shared modules
    vpc/
    rds/
  environments/
    dev/
      main.tf        # References modules
      variables.tf
      terraform.tfvars
      backend.tf     # Separate state backend
    staging/
      main.tf
      variables.tf
      terraform.tfvars
      backend.tf
    prod/
      main.tf
      variables.tf
      terraform.tfvars
      backend.tf     # Different account, different state
```

**Workspace-based separation (acceptable for non-production)**:

```bash
terraform workspace select dev
terraform plan -var-file=envs/dev.tfvars

terraform workspace select staging
terraform plan -var-file=envs/staging.tfvars
```

**Drift detection schedule (GitHub Actions)**:

```yaml
name: Terraform Drift Detection
on:
  schedule:
    - cron: '0 8 * * 1-5'  # Weekdays at 8 AM

jobs:
  drift:
    runs-on: ubuntu-22.04
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3
      - run: terraform init
      - run: terraform plan -detailed-exitcode
        # Exit code 2 = drift detected
```

---

## CloudFormation Notes

When reviewing CloudFormation templates, apply the same principles with these platform-specific adjustments:

| Terraform Concept | CloudFormation Equivalent | Notes |
|-------------------|--------------------------|-------|
| Remote state | CloudFormation manages state natively | No action needed; state is in AWS |
| State locking | Built-in via stack operations | No action needed |
| Modules | Nested stacks, AWS SAM | Check that nested stacks use versioned S3 URLs |
| `prevent_destroy` | `DeletionPolicy: Retain` or `DeletionPolicy: Snapshot` | Must be set on stateful resources |
| `terraform fmt` | `cfn-lint` | Linting is critical; CFN templates are verbose and error-prone |
| `terraform validate` | `aws cloudformation validate-template` | Basic syntax check only; `cfn-lint` catches more |
| Provider version pinning | `AWSTemplateFormatVersion` | Less of a concern; API changes are managed by AWS |

**CloudFormation-specific checks**:

- `DeletionPolicy` is set on RDS, S3, DynamoDB, and EFS resources (HIGH)
- `UpdateReplacePolicy` is set where replacement would cause data loss (MEDIUM)
- Stack parameters use `AllowedValues` and `AllowedPattern` for validation (MEDIUM)
- `Fn::ImportValue` is used for cross-stack references instead of hardcoded values (MEDIUM)
- Drift detection is scheduled or triggered post-deploy (LOW)

---

## Pulumi Notes

When reviewing Pulumi programs, apply the same principles with these platform-specific adjustments:

| Terraform Concept | Pulumi Equivalent | Notes |
|-------------------|-------------------|-------|
| Remote state | Pulumi Service backend or self-managed (S3, GCS) | Check that a remote backend is configured |
| State locking | Built-in with Pulumi Service; manual with self-managed | Ensure locking is available |
| Modules | Standard language packages/modules | Review as regular code plus infrastructure concerns |
| `prevent_destroy` | `protect: true` in resource options | Must be set on stateful resources |
| `terraform fmt` | Language-specific formatters (black, prettier, gofmt) | Standard code formatting applies |
| Variable validation | Language-native type checking and validation | Leverage the type system; Pulumi Config for runtime validation |
| Provider version pinning | Package manager version constraints (npm, pip, Go modules) | Same lockfile discipline as application code |

**Pulumi-specific checks**:

- `protect: true` is set on stateful resources (HIGH)
- Stack configuration uses `pulumi config set --secret` for sensitive values (HIGH)
- Secrets provider is configured (not default passphrase in production) (MEDIUM)
- Component resources are used for reusable infrastructure patterns (MEDIUM)
- `Pulumi.yaml` specifies runtime version (LOW)

---

## Quick Reference: Severity Summary

| Severity | IaC Findings |
|----------|-------------|
| CRITICAL | Secrets or credentials hardcoded in `.tf` files, `.tfvars`, or state backend config; security groups open to `0.0.0.0/0` on management ports |
| HIGH | No remote state or no state locking; no lifecycle protection on production databases; wildcard IAM permissions; sensitive outputs not marked sensitive; shared state across all environments; no plan review before production apply; unencrypted storage |
| MEDIUM | Unpinned module/provider versions; no variable validation; no tagging strategy; hardcoded resource IDs; no `terraform validate` in CI; workspaces as sole production isolation; no encryption at rest for state; missing public access blocks |
| LOW | Missing variable/output descriptions; formatting not enforced; inconsistent naming; no drift detection; HCL style issues; missing comments on non-obvious decisions |
| INFO | Well-structured module hierarchy; effective use of data sources; good tagging strategy with default_tags; proper state isolation pattern |
