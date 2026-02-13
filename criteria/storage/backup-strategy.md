# Backup Strategy and Data Protection

Reference checklist for the `storage-reviewer` agent when evaluating backup configurations, retention policies, restore procedures, and ransomware protection strategies. This file complements the inline criteria in `review-storage.md` with detailed severity classifications, actionable examples, and remediation patterns.

Backups are the last line of defense against data loss. Every other protection mechanism (RAID, replication, snapshots) can fail simultaneously in certain scenarios: ransomware encrypts live data and replicas, operator error deletes data across all copies, or a software bug corrupts data that is then faithfully replicated. A backup strategy that is not tested is not a backup strategy.

---

## 1. The 3-2-1 Rule

**Category**: Backup Coverage

The 3-2-1 rule is the foundational principle of backup strategy: maintain at least 3 copies of data, on at least 2 different media types, with at least 1 copy offsite. Modern variations extend this to 3-2-1-1-0: add 1 immutable copy and 0 errors (verified restores).

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| At least 3 copies of critical data exist (production + 2 backups) | HIGH | Fewer than 3 copies means a single backup failure leaves only the original |
| Backups use at least 2 different storage media or systems | HIGH | Same media type shares failure modes (firmware bugs, vendor issues) |
| At least 1 backup copy is offsite (different building, region, or cloud provider) | HIGH | Site-level disasters (fire, flood, power) destroy all onsite copies |
| At least 1 backup copy is immutable (cannot be modified or deleted) | HIGH | Ransomware and compromised credentials can delete mutable backups |
| Backup integrity is verified with zero errors | HIGH | Unverified backups may be corrupt and unusable when needed |
| 3-2-1 compliance is documented per data classification | MEDIUM | Different data has different protection requirements |
| Air-gapped backup exists for most critical data | MEDIUM | Network-accessible backups are vulnerable to network-propagating attacks |

### Common Mistakes

**Mistake**: "We replicate to a second site" as the entire backup strategy.

Why it is a problem: Replication propagates deletions, corruption, and ransomware. If an operator runs `DROP TABLE` on the primary, the replica drops it too. If ransomware encrypts files on the primary, the encrypted files replicate to the secondary. Replication is availability, not backup.

**Mistake**: All backups in the same cloud account.

```yaml
# BAD - compromised credentials delete production AND backups
backup:
  destination: s3://prod-account-backups/
  # Same AWS account as production
  # If IAM credentials are compromised, attacker deletes everything

# GOOD - backups in separate account with cross-account access
backup:
  destination: s3://backup-account-backups/
  # Separate AWS account
  # Production credentials cannot delete backups
  # Backup account has its own IAM, its own root credentials
  cross_account_role: arn:aws:iam::123456789012:role/backup-writer
```

**Mistake**: Counting RAID as a "copy."

Why it is a problem: RAID protects against drive failure, not data loss. RAID faithfully stores corrupted data, deleted data, and ransomware-encrypted data. RAID is not a backup. A RAID array is one copy of the data.

### Assessment Checklist

```
For each critical data system, verify:
[ ] Copy 1: Production data (on production storage)
[ ] Copy 2: Backup on different storage (NAS, object storage, tape)
[ ] Copy 3: Offsite backup (different physical location or cloud region)
[ ] At least one copy is immutable
[ ] At least one copy is in a different failure domain (power, network, building)
[ ] Backup integrity has been verified within the last 30 days
```

---

## 2. Backup Coverage

**Category**: Backup Coverage

Backup coverage means identifying every system and data type that requires backup, then ensuring appropriate backup methods are applied to each. Different data types require different backup approaches.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| All databases are backed up with application-consistent methods | CRITICAL | Filesystem-level backup of a running database produces a corrupt backup |
| Database backups use native dump tools or snapshot-integrated methods | HIGH | `cp` of database files while the database is running = unusable backup |
| Configuration files are backed up or managed in version control | MEDIUM | Losing configuration means rebuilding systems from memory |
| Secrets and certificates are backed up separately with appropriate encryption | HIGH | Lost encryption keys or certificates = lost access to encrypted data |
| Infrastructure state (Terraform state, Ansible inventory) is backed up | MEDIUM | Lost IaC state means manual reconciliation of infrastructure |
| Application data directories are identified and included in backup scope | HIGH | Unidentified data directories are unprotected data |
| Backup scope is documented with an explicit inclusion/exclusion list | MEDIUM | Undocumented scope means unknown gaps |
| Backup coverage is reviewed when new services are deployed | MEDIUM | New services are commonly forgotten in backup scope |
| Transient data (caches, temp files, build artifacts) is explicitly excluded | LOW | Backing up transient data wastes storage and backup window |

### Application-Consistent Database Backup Methods

| Database | Correct Backup Method | Wrong Method |
|----------|----------------------|--------------|
| PostgreSQL | `pg_dump`, `pg_basebackup`, WAL archiving, pgBackRest | `cp /var/lib/postgresql/` while running |
| MySQL/MariaDB | `mysqldump --single-transaction`, `xtrabackup`, `mariabackup` | `cp /var/lib/mysql/` while running |
| MongoDB | `mongodump`, filesystem snapshot with `db.fsyncLock()` | `cp /var/lib/mongodb/` while running |
| Redis | `BGSAVE` + copy RDB file, AOF persistence | `cp dump.rdb` while Redis is writing |
| Elasticsearch | Snapshot API to repository | `cp /var/lib/elasticsearch/` while running |
| etcd | `etcdctl snapshot save` | Filesystem copy while running |
| SQL Server | `BACKUP DATABASE`, VSS-aware snapshot | File copy while database is online |

### Common Mistakes

**Mistake**: Filesystem copy of a running database.

```bash
# BAD - produces inconsistent backup
rsync -av /var/lib/postgresql/15/main/ /backup/postgres/

# GOOD - application-consistent backup
pg_basebackup -D /backup/postgres/base -Ft -z -P -X stream

# GOOD - logical backup for portability
pg_dump --format=custom --compress=9 mydb > /backup/postgres/mydb.dump
```

**Mistake**: Not backing up secrets management data.

```yaml
# What must be backed up for secrets:
secrets_backup:
  - vault_unseal_keys     # Without these, Vault is permanently sealed
  - vault_recovery_keys   # For auto-unseal recovery
  - tls_certificates      # Especially private keys and CA certificates
  - ssh_host_keys         # Changing host keys disrupts all SSH clients
  - encryption_keys       # For any at-rest encryption (LUKS, KMS)
  - gpg_keys              # If used for signing or encrypting backups
  - kube_pki              # Kubernetes PKI if not using managed control plane
```

**Mistake**: Not backing up Terraform state.

```hcl
# If the S3 bucket holding Terraform state is deleted:
# - Terraform cannot manage existing infrastructure
# - Manual import of every resource is required
# - Hours to days of recovery work

# Ensure the state bucket itself is backed up or has versioning:
resource "aws_s3_bucket_versioning" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  versioning_configuration {
    status = "Enabled"
  }
}

# Cross-region replication for the state bucket:
resource "aws_s3_bucket_replication_configuration" "terraform_state" {
  bucket = aws_s3_bucket.terraform_state.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-state"
    status = "Enabled"

    destination {
      bucket        = aws_s3_bucket.terraform_state_replica.arn
      storage_class = "STANDARD_IA"
    }
  }
}
```

**Mistake**: Backup scope not reviewed when new services are added.

```yaml
# BAD - backup was configured once, never updated
backup_scope:
  include:
    - /var/lib/postgresql/
    - /etc/
    - /home/
  # New Redis server added 6 months ago - not included
  # New application data in /opt/myapp/data/ - not included

# GOOD - documented scope with last review date
backup_scope:
  last_reviewed: 2024-01-15
  reviewed_by: ops-team
  include:
    - /var/lib/postgresql/        # Primary database
    - /var/lib/redis/             # Session cache (added 2024-01-10)
    - /opt/myapp/data/            # Application uploads (added 2023-11-20)
    - /etc/                       # System configuration
    - /home/                      # User home directories
  exclude:
    - /var/cache/                 # Transient cache
    - /tmp/                       # Temporary files
    - /var/log/                   # Logs shipped to central logging
```

---

## 3. Retention Policies

**Category**: Backup Retention

Retention policies define how long backups are kept. Too short and you cannot recover from delayed-discovery incidents (corruption discovered weeks later). Too long and storage costs grow unbounded, and you may violate data minimization regulations.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Retention policy is defined and documented | HIGH | Without a policy, backups are either kept forever (cost) or deleted too soon (risk) |
| Daily backups are retained for at least 7 days | HIGH | Short-term retention covers common recovery scenarios (accidental deletion, recent corruption) |
| Weekly backups are retained for at least 4 weeks | MEDIUM | Provides recovery points for issues discovered within a month |
| Monthly backups are retained for at least 12 months | MEDIUM | Covers quarterly compliance, annual audits, and seasonal data needs |
| Yearly backups are retained per regulatory requirements | MEDIUM | Financial, healthcare, and government data often requires multi-year retention |
| Retention policy is enforced automatically (not manual deletion) | MEDIUM | Manual retention management leads to either hoarding or premature deletion |
| Retention periods align with regulatory requirements (GDPR, HIPAA, SOX, PCI-DSS) | HIGH | Non-compliance with retention regulations results in fines and legal exposure |
| Retention periods account for data discovery timelines | MEDIUM | If corruption takes 30 days to discover, 7-day retention is insufficient |
| Backup storage costs are monitored and projected | LOW | Unbounded retention creates unbounded cost |

### Retention Schedule Template

```yaml
retention_policy:
  name: "Standard Production"
  last_reviewed: 2024-01-15

  schedules:
    daily:
      frequency: "Every day at 02:00 UTC"
      retain: 14          # 2 weeks of daily granularity
      type: incremental   # Full daily is wasteful for large datasets

    weekly:
      frequency: "Every Sunday at 02:00 UTC"
      retain: 8           # ~2 months of weekly granularity
      type: incremental   # Or differential for easier single-restore

    monthly:
      frequency: "First Sunday of each month"
      retain: 12          # 1 year of monthly granularity
      type: full          # Full backup for independent restore capability

    yearly:
      frequency: "January 1"
      retain: 7           # 7 years (SOX, financial regulations)
      type: full          # Full backup for long-term archival

  storage_tiers:
    daily: hot_storage          # Fast retrieval for recent recovery
    weekly: warm_storage        # Standard retrieval speed
    monthly: cold_storage       # Reduced cost, slower retrieval
    yearly: archive_storage     # Lowest cost, retrieval may take hours

  regulatory_requirements:
    - standard: "SOX"
      minimum_retention: "7 years"
      data_types: ["financial records", "audit logs"]
    - standard: "HIPAA"
      minimum_retention: "6 years"
      data_types: ["patient records", "access logs"]
    - standard: "GDPR"
      maximum_retention: "as defined by purpose"
      data_types: ["personal data"]
      notes: "GDPR requires deletion when purpose is fulfilled"
```

### Common Mistakes

**Mistake**: Keeping only 3 days of backups.

Why it is a problem: Many data corruption or deletion events are not discovered for days or weeks. A ransomware attack that encrypts files on Thursday may not be noticed until Monday. With only 3 days of backups, the most recent clean backup has already been rotated out.

**Mistake**: No automated retention enforcement.

```bash
# BAD - manual deletion that nobody remembers to do
# Result: /backup/ fills up the disk after 6 months

# GOOD - automated retention with restic
restic forget \
  --keep-daily 14 \
  --keep-weekly 8 \
  --keep-monthly 12 \
  --keep-yearly 7 \
  --prune

# GOOD - automated retention with borgbackup
borg prune \
  --keep-daily=14 \
  --keep-weekly=8 \
  --keep-monthly=12 \
  --keep-yearly=7 \
  /backup/borg-repo
```

**Mistake**: GDPR-regulated personal data retained indefinitely in backups.

Why it is a problem: GDPR data minimization requires deletion when the processing purpose is no longer valid. Backups containing personal data must have a defined retention period that aligns with the legitimate processing purpose. Indefinite retention of backups containing personal data is a GDPR violation.

---

## 4. Backup Encryption

**Category**: Backup Security

Backups contain the same sensitive data as production. Unencrypted backups shipped offsite or stored in cloud object storage are a data breach waiting to happen.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Backups are encrypted at rest | HIGH | Unencrypted backup media is a data breach if stolen or accessed |
| Backups are encrypted in transit | HIGH | Unencrypted backup traffic over networks exposes data to interception |
| Encryption keys are stored separately from the backups they protect | CRITICAL | Keys stored alongside encrypted backups = unencrypted backups |
| Encryption key management has a recovery procedure | CRITICAL | Lost encryption keys = permanently inaccessible backups |
| Encryption uses strong algorithms (AES-256, ChaCha20-Poly1305) | MEDIUM | Weak encryption provides false sense of security |
| Client-side encryption is used for cloud-stored backups | MEDIUM | Server-side encryption protects against media theft but not provider access |
| Backup encryption keys are rotated on a schedule | LOW | Key rotation limits the blast radius of a key compromise |
| Key escrow or recovery mechanism exists for the backup encryption keys | HIGH | If the person who set up encryption leaves, can anyone still decrypt? |

### Common Mistakes

**Mistake**: Encryption keys stored in the same location as the backups.

```yaml
# BAD - encryption key sitting next to the encrypted backup
/backup/
  database.dump.enc
  encryption.key        # Defeats the purpose of encryption

# GOOD - key stored in a separate secrets management system
backup:
  encryption:
    algorithm: AES-256-GCM
    key_source: vault://secret/backup/encryption-key
    # Key is in HashiCorp Vault, separate from backup storage
    # Vault has its own backup and access controls
```

**Mistake**: Unencrypted backup transfer.

```bash
# BAD - backup sent over unencrypted connection
rsync /backup/database.dump backup-server:/backup/

# GOOD - encrypted transfer
rsync -e "ssh -i /root/.ssh/backup_key" /backup/database.dump backup-server:/backup/

# GOOD - backup tool with built-in encryption
restic -r sftp:backup-server:/backup backup /data
# restic encrypts before transmission

# GOOD - borgbackup with encryption
borg create --encryption=repokey ssh://backup-server/backup::daily /data
```

**Mistake**: No encryption key recovery plan.

```yaml
# Document the encryption key recovery process
encryption_key_recovery:
  primary_key_location: "HashiCorp Vault (production cluster)"
  recovery_keys:
    - location: "Physical safe in Building A, Room 104"
      type: "Printed paper key"
      holders: ["CTO", "VP Engineering"]
    - location: "Bank safe deposit box #4521"
      type: "Encrypted USB drive"
      passphrase_holders: ["CTO", "Director of Operations"]
  last_tested: 2024-01-15
  test_procedure: "Decrypt a sample backup using only recovery key"
```

### Encryption Configuration Examples

**restic with encryption**:

```bash
# Initialize encrypted repository (encryption is mandatory in restic)
export RESTIC_PASSWORD_COMMAND="vault kv get -field=password secret/backup/restic"
restic init -r s3:s3.amazonaws.com/backup-bucket

# All backups are automatically encrypted with AES-256
restic backup /data
```

**borgbackup with encryption**:

```bash
# Initialize with repokey encryption (key stored in repo, protected by passphrase)
export BORG_PASSCOMMAND="vault kv get -field=passphrase secret/backup/borg"
borg init --encryption=repokey-blake2 ssh://backup-server/backup

# Or use keyfile encryption (key stored locally, must be backed up separately)
borg init --encryption=keyfile-blake2 ssh://backup-server/backup
# CRITICAL: Back up the keyfile from ~/.config/borg/keys/
```

**Cloud backup encryption**:

```hcl
# AWS S3 bucket for backups with server-side encryption (baseline)
resource "aws_s3_bucket_server_side_encryption_configuration" "backup" {
  bucket = aws_s3_bucket.backup.id

  rule {
    apply_server_side_encryption_by_default {
      sse_algorithm     = "aws:kms"
      kms_master_key_id = aws_kms_key.backup.arn
    }
    bucket_key_enabled = true
  }
}

# Deny unencrypted uploads
resource "aws_s3_bucket_policy" "require_encryption" {
  bucket = aws_s3_bucket.backup.id
  policy = jsonencode({
    Statement = [{
      Effect    = "Deny"
      Principal = "*"
      Action    = "s3:PutObject"
      Resource  = "${aws_s3_bucket.backup.arn}/*"
      Condition = {
        StringNotEquals = {
          "s3:x-amz-server-side-encryption" = "aws:kms"
        }
      }
    }]
  })
}
```

---

## 5. Immutable Backups and Ransomware Protection

**Category**: Backup Security

Ransomware attacks routinely target backup infrastructure. Modern ransomware identifies and encrypts or deletes backups before encrypting production data. Immutable backups -- backups that cannot be modified or deleted during a retention period -- are the primary defense.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| At least one backup copy uses immutability (Object Lock, WORM, immutable snapshots) | HIGH | Mutable backups are deleted by ransomware with compromised credentials |
| Object Lock uses Compliance mode (not Governance mode) for critical backups | HIGH | Governance mode can be overridden by IAM users with sufficient permissions |
| Immutability retention period matches backup retention policy | MEDIUM | Immutable backups that expire before retention policy = gap in protection |
| Backup service credentials are separate from production credentials | HIGH | Shared credentials mean compromised production = compromised backups |
| Backup deletion requires MFA or out-of-band approval | MEDIUM | Additional friction prevents automated or hasty backup deletion |
| Backup infrastructure uses separate authentication (different AD/LDAP, separate accounts) | HIGH | Domain compromise typically gives access to all domain-joined systems including backups |
| Network segmentation isolates backup infrastructure | MEDIUM | Backup servers accessible from production network are targets for lateral movement |
| Backup integrity verification runs automatically | HIGH | Tampered backups pass if nobody checks integrity |
| Recovery from immutable backup is tested at least quarterly | HIGH | Untested recovery = unknown recovery |

### Object Lock Configuration

```hcl
# AWS S3 Object Lock (Compliance mode - cannot be overridden by any user including root)
resource "aws_s3_bucket" "immutable_backup" {
  bucket = "immutable-backups"

  object_lock_enabled = true
}

resource "aws_s3_bucket_object_lock_configuration" "backup" {
  bucket = aws_s3_bucket.immutable_backup.id

  rule {
    default_retention {
      mode = "COMPLIANCE"     # COMPLIANCE mode: nobody can delete, not even root account
      days = 30               # 30-day immutability window
    }
  }
}

resource "aws_s3_bucket_versioning" "backup" {
  bucket = aws_s3_bucket.immutable_backup.id
  versioning_configuration {
    status = "Enabled"        # Required for Object Lock
  }
}
```

**Governance vs Compliance mode**:

| Feature | Governance Mode | Compliance Mode |
|---------|----------------|-----------------|
| Can admin override? | Yes, with `s3:BypassGovernanceRetention` permission | No, not even the root account |
| Can retention be shortened? | Yes, by authorized users | No |
| Can objects be deleted? | Yes, by authorized users | No, not until retention expires |
| Use case | Testing, non-critical backups | Regulatory compliance, ransomware protection |
| Severity if used for critical backups | HIGH (can be bypassed) | INFO (correct configuration) |

### WORM Storage Options

| Platform | WORM Mechanism | Configuration |
|----------|---------------|---------------|
| AWS S3 | Object Lock (Compliance mode) | `object_lock_enabled = true` |
| Azure Blob | Immutable Blob Storage | Time-based retention policy with lock |
| GCP Cloud Storage | Bucket Lock + Retention Policy | `retention_policy` with `is_locked = true` |
| MinIO | Object Lock (S3-compatible) | `mc retention set --default COMPLIANCE` |
| Wasabi | Object Lock (S3-compatible) | Same as AWS S3 Object Lock API |
| NetApp ONTAP | SnapLock | SnapLock Compliance or Enterprise mode |
| Dell PowerScale | SmartLock WORM | Compliance or Enterprise mode |
| Veeam | Hardened Repository | Linux-based immutable repository with `chattr +i` |

### Common Mistakes

**Mistake**: Object Lock in Governance mode for critical backups.

```hcl
# BAD - Governance mode can be overridden by IAM users
resource "aws_s3_bucket_object_lock_configuration" "backup" {
  bucket = aws_s3_bucket.backup.id
  rule {
    default_retention {
      mode = "GOVERNANCE"   # Can be bypassed with sufficient IAM permissions
      days = 30
    }
  }
}

# GOOD - Compliance mode for critical backups
resource "aws_s3_bucket_object_lock_configuration" "backup" {
  bucket = aws_s3_bucket.backup.id
  rule {
    default_retention {
      mode = "COMPLIANCE"   # Cannot be overridden by anyone
      days = 30
    }
  }
}
```

**Mistake**: Backup credentials accessible from production servers.

```yaml
# BAD - production application server has credentials to delete backups
# If the server is compromised, backups are compromised
production_server:
  environment:
    AWS_ACCESS_KEY_ID: "AKIABACKUPWRITER"     # Can write AND delete backups
    AWS_SECRET_ACCESS_KEY: "..."

# GOOD - separate backup credentials with write-only access
backup_iam_policy:
  Statement:
    - Effect: Allow
      Action:
        - s3:PutObject                        # Can write
      Resource: "arn:aws:s3:::backup-bucket/*"
    - Effect: Deny
      Action:
        - s3:DeleteObject                     # Cannot delete
        - s3:PutBucketPolicy                  # Cannot change bucket policy
        - s3:DeleteBucket                     # Cannot delete bucket
      Resource:
        - "arn:aws:s3:::backup-bucket"
        - "arn:aws:s3:::backup-bucket/*"
```

**Mistake**: Backup infrastructure on the same domain/authentication system.

Why it is a problem: In a domain compromise scenario (Active Directory, LDAP), the attacker gains access to all domain-joined systems. If the backup server uses the same domain credentials, the attacker can access and destroy backups before deploying ransomware. The backup infrastructure should use separate authentication that is not compromised by a domain-level breach.

### Ransomware Protection Checklist

```
Defense-in-depth for backup infrastructure:

[ ] At least one immutable backup copy (Object Lock, WORM)
[ ] Backup credentials separate from production
[ ] Backup infrastructure on separate authentication domain
[ ] Network segmentation between production and backup infrastructure
[ ] MFA required for backup administrative operations
[ ] Backup deletion requires out-of-band approval (phone call, separate approval system)
[ ] Air-gapped backup for most critical data (tape, offline disk)
[ ] Regular restore testing to verify backup integrity
[ ] Canary files monitored for unexpected encryption/modification
[ ] Backup system logs forwarded to separate SIEM
[ ] Incident response plan covers backup recovery scenario
```

---

## 6. Backup Verification and Restore Testing

**Category**: Backup Verification

An unverified backup is not a backup -- it is a hope. Backup verification confirms that the backup data is intact, complete, and can be restored within the required time window. Without verification, you discover that backups are unusable during the worst possible moment: an actual data loss event.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Automated integrity verification runs on every backup | HIGH | Detects corruption, truncation, or encryption issues immediately |
| Full restore test is performed at least quarterly | HIGH | Verifies end-to-end recoverability including procedures, credentials, and documentation |
| Restore test measures actual RTO (time to restore) | HIGH | Theoretical RTO is meaningless without measured actual RTO |
| Restore test verifies data completeness and correctness | HIGH | A successful restore of corrupt data is not a successful restore |
| Restore test results are documented and reviewed | MEDIUM | Documented results create accountability and track improvements |
| Restore test includes database consistency checks | HIGH | Restored database must pass `CHECKDB`, `pg_dump`, or equivalent integrity check |
| Partial/granular restore capability is verified (single file, single table) | MEDIUM | Most restores are for specific items, not full system recovery |
| Restore documentation is complete enough for someone unfamiliar to execute | MEDIUM | The person who set up backups may not be the person who performs the restore |
| Time to first byte from backup is measured per backup tier | LOW | Cold/archive tier retrieval delays affect RTO |

### Common Mistakes

**Mistake**: Never testing restores.

```yaml
# BAD - "we back up every night" but have never restored
backup_history:
  last_backup: "2024-01-15T02:00:00Z"
  last_restore_test: "never"  # CRITICAL finding

# GOOD - regular restore testing with documented results
restore_testing:
  schedule: "Quarterly (January, April, July, October)"
  last_test:
    date: "2024-01-15"
    database:
      status: "SUCCESS"
      restore_time: "47 minutes"
      data_verified: true
      consistency_check: "PASS"
    files:
      status: "SUCCESS"
      restore_time: "2 hours 15 minutes"
      sample_verification: "10 random files checked - all correct"
    rto_target: "4 hours"
    rto_actual: "2 hours 47 minutes"
    findings:
      - "Restore documentation missing step for DNS cutover"
      - "Database restore requires manual sequence reset"
    action_items:
      - "Update restore runbook with DNS and sequence steps"
```

**Mistake**: Restoring but not verifying data integrity.

```bash
# BAD - restore "succeeds" but data may be corrupt
pg_restore -d mydb /backup/mydb.dump
echo "Restore complete"  # Is it, though?

# GOOD - restore AND verify
pg_restore -d mydb_restore_test /backup/mydb.dump
psql mydb_restore_test -c "SELECT count(*) FROM critical_table;"
psql mydb_restore_test -c "SELECT count(*) FROM users WHERE created_at > now() - interval '24 hours';"
# Compare counts against production
# Run application-specific data integrity checks
```

### Restore Test Procedure Template

```markdown
## Quarterly Restore Test Procedure

### Pre-requisites
1. Isolated restore environment provisioned (not production)
2. Most recent backup identified and located
3. Encryption keys accessible from backup key store
4. Restore runbook available (this document)

### Step 1: Identify Backup
- Date of backup under test: ___
- Backup location: ___
- Backup size: ___
- Encryption key reference: ___

### Step 2: Retrieve Backup
- Start time: ___
- Time to first byte: ___
- Transfer complete time: ___
- Transfer duration: ___

### Step 3: Decrypt and Extract
- Decryption successful: [ ] Yes [ ] No
- Any errors: ___

### Step 4: Restore Database
- Start time: ___
- Completion time: ___
- Duration: ___
- Restore command used: ___

### Step 5: Verify Data Integrity
- Row counts match production: [ ] Yes [ ] No
- Database consistency check passes: [ ] Yes [ ] No
- Sample data spot-check passes: [ ] Yes [ ] No
- Application smoke test passes: [ ] Yes [ ] No

### Step 6: Verify File Restores
- Random sample of 10 files verified: [ ] Yes [ ] No
- File checksums match: [ ] Yes [ ] No
- File permissions correct: [ ] Yes [ ] No

### Step 7: Document Results
- Total RTO (end to end): ___
- RTO target met: [ ] Yes [ ] No
- Issues found: ___
- Action items: ___

### Sign-off
- Tested by: ___
- Reviewed by: ___
- Date: ___
```

---

## 7. RTO and RPO

**Category**: Business Continuity

Recovery Time Objective (RTO) defines the maximum acceptable time to restore service after a disaster. Recovery Point Objective (RPO) defines the maximum acceptable data loss measured in time. These are business requirements, not technical decisions, and every backup architecture must be designed to meet documented RTO/RPO targets.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| RTO is defined and documented for each critical system | HIGH | Without defined RTO, there is no target to design against or measure |
| RPO is defined and documented for each critical system | HIGH | Without defined RPO, backup frequency is arbitrary |
| Backup frequency meets RPO requirement | HIGH | Daily backups with 1-hour RPO = 23 hours of potential data loss |
| Measured restore time meets RTO requirement | HIGH | If restore takes 8 hours but RTO is 4 hours, the backup architecture must change |
| RTO includes all recovery steps (retrieval, decrypt, restore, verify, DNS, warm-up) | MEDIUM | Partial RTO measurement is misleading |
| RTO/RPO are tested at least annually, preferably quarterly | HIGH | Untested RTO/RPO are theoretical |
| RTO/RPO requirements are signed off by business stakeholders | MEDIUM | IT should not unilaterally define business continuity requirements |
| Different tiers of data have different RTO/RPO targets | LOW | Not all data has the same criticality |

### Common RTO/RPO Tiers

| Tier | RPO | RTO | Backup Method | Example Systems |
|------|-----|-----|---------------|-----------------|
| Tier 1 (Mission Critical) | < 1 hour | < 1 hour | Synchronous replication + hot standby | Payment processing, trading systems |
| Tier 2 (Business Critical) | < 4 hours | < 4 hours | Async replication + warm standby + frequent backups | Core databases, ERP, customer-facing APIs |
| Tier 3 (Business Operational) | < 24 hours | < 24 hours | Daily backups + cold standby | Internal tools, reporting, file storage |
| Tier 4 (Non-Critical) | < 72 hours | < 72 hours | Weekly backups | Development environments, test data |

### Common Mistakes

**Mistake**: RPO mismatch with backup frequency.

```yaml
# BAD - RPO says 1 hour but backups run daily
rpo_requirement: 1h
backup_schedule: "Daily at 02:00 UTC"
# Maximum data loss = 24 hours (not 1 hour)

# GOOD - RPO matched to backup/replication frequency
rpo_requirement: 1h
replication: synchronous          # RPO = 0
backup_schedule: "Hourly"         # Backup granularity = 1 hour
wal_archiving: "Continuous"       # Point-in-time recovery within the hour
```

**Mistake**: RTO does not account for full recovery process.

```yaml
# BAD - RTO only measures database restore time
rto_claimed: "2 hours"
rto_components:
  database_restore: "2 hours"
  # Missing: backup retrieval from glacier (4-12 hours!)
  # Missing: DNS propagation (up to TTL)
  # Missing: cache warming (variable)
  # Missing: application configuration (30 min)
  # Missing: verification testing (1 hour)
  # Actual RTO: 8-16 hours

# GOOD - comprehensive RTO breakdown
rto_claimed: "8 hours"
rto_components:
  backup_retrieval: "4 hours"     # S3 Glacier Flexible retrieval
  decryption: "15 minutes"
  database_restore: "2 hours"
  application_config: "30 minutes"
  verification: "45 minutes"
  dns_cutover: "15 minutes"       # Low TTL pre-configured
  cache_warming: "15 minutes"
  total: "8 hours"
```

---

## 8. Snapshot Management

**Category**: Snapshot Management

Snapshots are not backups. They provide point-in-time recovery on the same storage system and are excellent for fast rollback, but they share the fate of the underlying storage. A failed storage array, pool corruption, or accidental pool destruction destroys all snapshots.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Snapshots are not used as the sole backup mechanism | HIGH | Snapshots share the underlying storage; storage failure = snapshot loss |
| Snapshot retention limits are configured | HIGH | Unbounded snapshots consume space and can fill the volume |
| Snapshot space consumption is monitored and alerted | HIGH | Snapshots that fill available space can cause writes to fail or the pool to hang |
| Application-consistent snapshots are used for databases | HIGH | Crash-consistent snapshots may require database recovery after restore |
| Snapshot frequency matches the use case (hourly for active data, less for archives) | MEDIUM | Too frequent = space waste; too infrequent = large recovery gap |
| Old snapshots are automatically pruned | MEDIUM | Manual pruning is forgotten; old snapshots accumulate |
| Snapshot overhead is accounted for in capacity planning | MEDIUM | A volume that is "50% full" may be 90% full when snapshot space is included |
| Snapshots are documented as "not a backup" in runbooks | LOW | Prevents operational confusion between snapshots and backups |

### Common Mistakes

**Mistake**: Snapshots as the only data protection.

```yaml
# BAD - snapshots are the only protection
data_protection:
  snapshots:
    schedule: "Hourly"
    retention: 72  # 72 hourly snapshots
  backup: none     # CRITICAL - no backup exists

# GOOD - snapshots for fast rollback, backups for disaster recovery
data_protection:
  snapshots:
    schedule: "Hourly"
    retention: 48                        # 2 days of hourly snapshots
    purpose: "Fast local rollback"
  backup:
    schedule: "Daily, incremental"
    retention: "14 daily, 8 weekly, 12 monthly"
    destination: "Offsite object storage"
    encryption: "AES-256"
    purpose: "Disaster recovery"
```

**Mistake**: No snapshot space monitoring.

```bash
# ZFS - check snapshot space usage
zfs list -t snapshot -o name,used,refer -s used
# Alert if total snapshot space exceeds 30% of pool free space

# LVM - check snapshot usage
lvs -o lv_name,snap_percent
# Alert if snapshot is > 80% full (snapshot invalidated at 100%)

# AWS EBS - snapshots grow with changes
aws ec2 describe-snapshots --owner-ids self \
  --query 'Snapshots[*].[SnapshotId,VolumeSize,StartTime]'
```

**Mistake**: Crash-consistent snapshots of databases.

```bash
# BAD - snapshot taken while database is actively writing
# Result: recovered database requires crash recovery, may lose recent transactions
lvcreate --snapshot --name db_snap --size 10G /dev/vg0/db_volume

# GOOD - application-consistent snapshot
# PostgreSQL: use pg_start_backup/pg_stop_backup or pg_backup_start/pg_backup_stop (v15+)
psql -c "SELECT pg_backup_start('snapshot');"
lvcreate --snapshot --name db_snap --size 10G /dev/vg0/db_volume
psql -c "SELECT pg_backup_stop();"

# MySQL: flush and lock, snapshot, unlock
mysql -e "FLUSH TABLES WITH READ LOCK;"
lvcreate --snapshot --name db_snap --size 10G /dev/vg0/db_volume
mysql -e "UNLOCK TABLES;"

# MongoDB: fsyncLock, snapshot, fsyncUnlock
mongosh --eval "db.fsyncLock();"
lvcreate --snapshot --name db_snap --size 10G /dev/vg0/db_volume
mongosh --eval "db.fsyncUnlock();"

# VMware: use quiesced snapshots with VMware Tools
# Kubernetes: use VolumeSnapshot with CSI driver supporting app-consistent hooks
```

### Snapshot Space Calculation

```
ZFS snapshot space:
  - Snapshot initially consumes zero space
  - As data changes on the active dataset, blocks referenced by the snapshot
    are preserved instead of freed
  - Snapshot size = sum of all blocks changed since snapshot was taken
  - Deleting many small files after snapshot = snapshot grows
  - Overwriting large files after snapshot = snapshot grows significantly

LVM snapshot space:
  - Fixed-size COW area allocated at snapshot creation
  - If COW area fills up, snapshot is INVALIDATED (data lost)
  - Rule of thumb: allocate snapshot size = expected change rate * snapshot lifetime
  - Monitor with: lvs -o snap_percent

Cloud EBS/disk snapshots:
  - Incremental: only changed blocks stored after first full snapshot
  - Cost proportional to change rate, not volume size
  - Deletion of intermediate snapshots is handled automatically
```

---

## 9. Backup Credential Separation

**Category**: Backup Security

Backup credentials must be isolated from production credentials. If an attacker who compromises the production environment can also delete or encrypt backups, the backup strategy has failed.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Backup service account is separate from production service accounts | HIGH | Compromised production credentials should not grant backup access |
| Backup credentials grant minimum necessary permissions (write, not delete) | HIGH | Principle of least privilege limits blast radius of credential compromise |
| Backup credentials are rotated on schedule | MEDIUM | Long-lived credentials increase exposure window |
| Backup administrative credentials (delete, policy change) require MFA | HIGH | Destructive backup operations should require additional authentication |
| Production systems cannot delete or modify existing backups | HIGH | Production compromise should not enable backup destruction |
| Backup credentials are not stored on production servers | MEDIUM | Credential theft from production server compromises backups |
| Break-glass procedure exists for backup administrative access | MEDIUM | Emergency access must be available but audited |

### Common Mistakes

**Mistake**: Single IAM user for production and backup operations.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:*",
      "Resource": "*"
    }
  ]
}
```

```json
// GOOD - separate policies for backup writer (on production servers)
// and backup administrator (break-glass only)

// Backup Writer Policy (attached to production backup service account)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:PutObject",
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::backup-bucket",
        "arn:aws:s3:::backup-bucket/*"
      ]
    }
  ]
}

// Backup Administrator Policy (separate account, MFA required)
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:DeleteObject",
        "s3:PutBucketPolicy",
        "s3:PutObjectRetention"
      ],
      "Resource": [
        "arn:aws:s3:::backup-bucket",
        "arn:aws:s3:::backup-bucket/*"
      ],
      "Condition": {
        "Bool": {
          "aws:MultiFactorAuthPresent": "true"
        }
      }
    }
  ]
}
```

---

## Quick Reference: Severity Summary

| Severity | Backup Strategy Findings |
|----------|--------------------------|
| CRITICAL | Encryption keys stored alongside encrypted backups; no key recovery procedure; filesystem-level copy of running database as "backup"; no backup exists for critical stateful services |
| HIGH | 3-2-1 rule not met (fewer than 3 copies, no offsite, no different media); no immutable backup copy; backup encryption missing (at rest or in transit); no retention policy defined; daily retention < 7 days; backup frequency does not meet RPO; RTO not tested or measured; restore never tested; snapshots used as sole backup; snapshot space not monitored; backup credentials shared with production; production can delete backups; no automated integrity verification; backup not application-consistent for databases; replication lag monitoring missing for RPO validation; key escrow missing for backup encryption; MFA not required for backup deletion; Object Lock in Governance mode for critical data |
| MEDIUM | Weekly retention < 4 weeks; monthly retention < 12 months; yearly retention does not meet regulatory requirements; no automated retention enforcement; backup scope not documented; backup scope not reviewed on new deployments; client-side encryption not used for cloud backups; immutability retention period mismatch; backup infrastructure on same authentication domain; backup network not segmented; snapshot frequency mismatch; snapshot overhead not in capacity planning; backup credential rotation not scheduled; break-glass procedure not documented; partial restore not verified; RTO does not include all recovery steps; 3-2-1 compliance not documented per data classification; air-gapped backup not present for critical data |
| LOW | Backup encryption key rotation not scheduled; backup storage costs not monitored; transient data not explicitly excluded from backup; time to first byte not measured per tier; different data tiers not defined; snapshots not documented as "not a backup" in runbooks |
| INFO | Well-implemented 3-2-1-1-0 strategy; effective use of immutable storage with Compliance mode; automated restore testing with documented results; comprehensive RTO breakdown with measured components; proper credential separation with least privilege |

---

## References

- NIST SP 800-34 Rev.1: Contingency Planning Guide for Federal Information Systems
- NIST SP 800-209: Security Guidelines for Storage Infrastructure
- CIS Controls v8: Control 11 - Data Recovery
- AWS Well-Architected Framework - Reliability Pillar: https://docs.aws.amazon.com/wellarchitected/latest/reliability-pillar/
- Veeam 3-2-1-1-0 Backup Rule: https://www.veeam.com/blog/321-backup-rule.html
- restic Documentation: https://restic.readthedocs.io/
- borgbackup Documentation: https://borgbackup.readthedocs.io/
- pgBackRest Documentation: https://pgbackrest.org/
