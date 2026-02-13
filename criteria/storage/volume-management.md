# Volume Management and Storage Provisioning

Reference checklist for the `storage-reviewer` agent when evaluating logical volume management, filesystem configuration, Kubernetes storage, cloud storage services, and capacity planning. This file complements the inline criteria in `review-storage.md` with detailed severity classifications, configuration examples, and remediation patterns.

---

## 1. LVM (Logical Volume Manager)

**Category**: Volume Management

LVM provides flexible volume management on Linux, allowing dynamic resizing, snapshots, and thin provisioning. Misconfigured LVM -- especially thin pools without monitoring or snapshots without space limits -- leads to silent data loss when the underlying storage fills up.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Thin pool usage is monitored with alerts at 70%, 80%, and 90% | HIGH | A full thin pool silently drops writes or freezes the filesystem |
| Thin pool auto-extend is configured (or manual extension procedure is documented) | HIGH | Without auto-extend, thin pool exhaustion requires immediate manual intervention |
| Volume Group (VG) has documented free space policy (minimum 10-20% free) | MEDIUM | No VG free space = cannot extend any LV, cannot create snapshots |
| LVM snapshot COW space is sized for expected change rate | HIGH | Undersized snapshot COW area causes snapshot invalidation and data loss |
| LVM snapshot retention is limited and monitored | MEDIUM | Forgotten snapshots consume space and degrade performance |
| PV (Physical Volume) metadata is backed up (`vgcfgbackup`) | MEDIUM | Lost LVM metadata = lost volume layout (data recovery becomes complex) |
| `lvs`, `vgs`, `pvs` output is included in monitoring | MEDIUM | Without monitoring, thin pool and VG status are invisible |
| Thin pool chunk size is appropriate for workload | LOW | Default 64K is fine for most workloads; databases may benefit from larger chunks |
| LVM volumes use meaningful names | LOW | Names like `lv_postgres_data` are operationally clearer than `lvol0` |

### Thin Pool Configuration and Monitoring

```bash
# Check thin pool usage
lvs -o lv_name,lv_size,data_percent,metadata_percent vg0/thinpool0
# CRITICAL if data_percent > 95%
# HIGH if data_percent > 85%
# MEDIUM if data_percent > 70%

# Configure thin pool auto-extend in /etc/lvm/lvm.conf
activation {
  thin_pool_autoextend_threshold = 80   # Trigger at 80% full
  thin_pool_autoextend_percent = 20     # Extend by 20% of current size
  monitoring = 1                        # Enable dmeventd monitoring
}

# Verify dmeventd is running (required for auto-extend)
systemctl status dm-event
systemctl enable dm-event
```

### Common Mistakes

**Mistake**: Thin pool without monitoring or auto-extend.

```bash
# BAD - thin pool with no monitoring
# When the thin pool fills, writes silently fail or the filesystem goes read-only
lvcreate -L 100G --type thin-pool --name thinpool0 vg0
lvcreate -V 500G --thin-pool thinpool0 --name data vg0
# 500 GB thin volume on 100 GB pool, no monitoring, no auto-extend

# GOOD - thin pool with auto-extend and monitoring
lvcreate -L 100G --type thin-pool --name thinpool0 vg0

# /etc/lvm/lvm.conf
activation {
  thin_pool_autoextend_threshold = 80
  thin_pool_autoextend_percent = 20
  monitoring = 1
}

# Monitoring alert integration
# Prometheus node_exporter or custom script
#!/bin/bash
DATA_PCT=$(lvs --noheadings -o data_percent vg0/thinpool0 | tr -d ' ')
if (( $(echo "$DATA_PCT > 85" | bc -l) )); then
  echo "CRITICAL: Thin pool vg0/thinpool0 at ${DATA_PCT}%"
  # Send alert
fi
```

**Mistake**: LVM snapshot without adequate COW space.

```bash
# BAD - snapshot with fixed 1 GB COW space on an active database volume
# If more than 1 GB of blocks change, snapshot is INVALIDATED (unusable)
lvcreate --snapshot --name db_snap --size 1G /dev/vg0/db_data

# Check snapshot usage
lvs -o lv_name,snap_percent vg0/db_snap
# If snap_percent reaches 100%, snapshot is invalidated

# GOOD - adequately sized snapshot
# Rule: snapshot size = estimated change rate * snapshot lifetime
# For a database with 100 MB/hour change rate and 24-hour snapshot lifetime:
# Minimum size = 100 MB * 24 = 2.4 GB, allocate 2x = 5 GB
lvcreate --snapshot --name db_snap --size 5G /dev/vg0/db_data

# BETTER - use thin snapshots (share thin pool space, no fixed COW area)
# Thin snapshots do not have a fixed size limit; they consume thin pool space
lvcreate --snapshot --name db_snap --thinpool thinpool0 vg0/db_data
```

**Mistake**: Not backing up LVM metadata.

```bash
# LVM metadata describes the volume layout
# If metadata is lost, recovering data requires manual reconstruction

# Automatic backup location
ls /etc/lvm/backup/        # VG metadata backups
ls /etc/lvm/archive/       # VG metadata archives (history)

# Manual backup
vgcfgbackup vg0
# Backup written to /etc/lvm/backup/vg0

# Restore metadata (in case of metadata corruption)
vgcfgrestore vg0
```

### VG Free Space Policy

```bash
# Check VG free space
vgs -o vg_name,vg_size,vg_free,vg_free_count
# Ensure at least 10-20% free for snapshot space, LV extension, and emergencies

# Example monitoring check
#!/bin/bash
VG_FREE_PCT=$(vgs --noheadings -o vg_free_count,pv_count vg0 | awk '{
  free=$1; total=$1+$2; print (free/total)*100
}')
# More practically:
VG_SIZE=$(vgs --noheadings --nosuffix --units g -o vg_size vg0 | tr -d ' ')
VG_FREE=$(vgs --noheadings --nosuffix --units g -o vg_free vg0 | tr -d ' ')
VG_FREE_PCT=$(echo "scale=2; $VG_FREE / $VG_SIZE * 100" | bc)

if (( $(echo "$VG_FREE_PCT < 10" | bc -l) )); then
  echo "WARNING: VG vg0 has only ${VG_FREE_PCT}% free space"
fi
```

---

## 2. Filesystem Selection and Mount Options

**Category**: Filesystem Configuration

Filesystem selection and mount options directly affect data integrity, performance, and recoverability. The wrong filesystem for a workload, or missing mount options like `noatime` or reserved blocks, creates performance problems or operational risk.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Filesystem type is appropriate for the workload | MEDIUM | XFS for large files, ext4 for general, ZFS/Btrfs for checksumming |
| Mount options include `noatime` or `relatime` for performance-sensitive volumes | LOW | `atime` generates a write for every read; `noatime` eliminates this |
| ext4 reserved blocks are tuned (5% default is excessive for large volumes) | LOW | 5% of a 10 TB volume is 500 GB of reserved space |
| Filesystems are mounted with `nodev,nosuid,noexec` where appropriate | MEDIUM | Security hardening for data volumes that do not need device files or executables |
| `discard` or periodic `fstrim` is configured for SSD volumes | MEDIUM | Without TRIM, SSD performance degrades over time as the FTL runs out of clean blocks |
| Journal mode is appropriate (`data=ordered` for ext4, log bias for XFS) | MEDIUM | Default journal modes are appropriate for most workloads; misconfigurations cause issues |
| Filesystem checks (`fsck`) are scheduled or triggered on boot | LOW | Filesystem corruption is detected and repaired before data loss compounds |
| XFS uses `inode64` for volumes > 2 TB (default since Linux 4.18) | LOW | Without `inode64`, inodes concentrate in the first 2 TB, causing performance issues |

### Filesystem Selection Guide

| Filesystem | Best For | Not Suitable For | Key Features |
|-----------|---------|-------------------|--------------|
| XFS | Large files, databases, high throughput, volumes > 16 TB | Many small files with frequent deletion | Online resize (grow only), parallel I/O, reflinks, no shrink |
| ext4 | General purpose, boot volumes, small to medium volumes | Extremely large volumes (> 50 TB) | Online resize (grow and shrink), mature, broad tool support |
| ZFS | Data integrity critical, mixed workloads, storage servers | Low-memory systems, environments requiring kernel-level support | Checksumming, compression, snapshots, built-in RAID, COW |
| Btrfs | Linux desktops, NAS, development servers | Heavy database workloads, production databases | Checksumming, snapshots, compression, subvolumes, RAID (limited) |

### fstab Best Practices

| Check | Severity | Rationale |
|-------|----------|-----------|
| Devices are identified by UUID or LABEL, not `/dev/sdX` names | HIGH | Device names change on reboot, PCI re-enumeration, or drive replacement |
| Mount options are explicit (not relying on defaults) | LOW | Explicit options document intent and prevent surprise behavior changes on OS upgrades |
| `nofail` is used for non-critical volumes | MEDIUM | Without `nofail`, a failed non-critical volume prevents system boot |
| Swap partitions use UUID | MEDIUM | Wrong swap device can cause data loss (overwriting a data partition with swap) |
| Boot-critical mounts use `0 2` for fsck pass (or `0 0` for non-fsckable filesystems) | LOW | Correct fsck ordering ensures root is checked first |
| The `_netdev` option is used for network-attached volumes (iSCSI, NFS, Ceph RBD) | HIGH | Without `_netdev`, system tries to mount before network is available, causing boot hang |

### Common Mistakes

**Mistake**: Using device names instead of UUIDs in fstab.

```bash
# BAD - device names can change on reboot
# /etc/fstab
/dev/sdb1  /data  ext4  defaults  0  2
/dev/sdc1  /logs  xfs   defaults  0  2
# If a drive is added, removed, or cables are rearranged:
# /dev/sdb may become /dev/sdc, mounting the wrong volume

# GOOD - UUID-based identification
# Get UUIDs
blkid

# /etc/fstab
UUID=a1b2c3d4-e5f6-7890-abcd-ef1234567890  /data  ext4  defaults,noatime  0  2
UUID=f9e8d7c6-b5a4-3210-fedc-ba9876543210  /logs  xfs   defaults,noatime  0  2

# ALSO GOOD - LABEL-based identification
LABEL=data  /data  ext4  defaults,noatime  0  2
LABEL=logs  /logs  xfs   defaults,noatime  0  2
```

**Mistake**: Missing `nofail` for non-critical volumes.

```bash
# BAD - failed mount of /data/cache prevents system boot
UUID=...  /data/cache  ext4  defaults  0  2

# GOOD - system boots even if /data/cache is unavailable
UUID=...  /data/cache  ext4  defaults,nofail  0  2
```

**Mistake**: Missing `_netdev` for network volumes.

```bash
# BAD - iSCSI volume mount without _netdev
# System hangs on boot trying to mount before network is up
UUID=...  /data/iscsi  xfs  defaults  0  2

# GOOD - _netdev delays mount until network is available
UUID=...  /data/iscsi  xfs  defaults,_netdev,nofail  0  2

# NFS mount
nfs-server:/export  /data/nfs  nfs  defaults,_netdev,nofail  0  0
```

**Mistake**: ext4 reserved blocks on large data volumes.

```bash
# BAD - 5% reserved on a 10 TB volume = 500 GB wasted
mkfs.ext4 /dev/sdb1   # Default 5% reserved for root

# GOOD - reduce reserved blocks for data (non-root) volumes
mkfs.ext4 -m 1 /dev/sdb1          # 1% reserved (100 GB on 10 TB)
# Or adjust after creation
tune2fs -m 1 /dev/sdb1

# For root filesystem, keep 5% (or at least 2-3%) to prevent
# the filesystem from filling completely and locking out root
```

### Recommended Mount Options by Use Case

```bash
# Database volume (XFS)
UUID=...  /var/lib/postgresql  xfs  defaults,noatime,logbufs=8,logbsize=256k,allocsize=1g  0  2

# General data volume (ext4)
UUID=...  /data  ext4  defaults,noatime,discard  0  2

# Temporary / scratch volume
UUID=...  /tmp  ext4  defaults,noatime,nodev,nosuid,noexec  0  2

# Container runtime data (XFS with project quotas)
UUID=...  /var/lib/docker  xfs  defaults,noatime,pquota  0  2

# Read-only application binaries
UUID=...  /opt/app  ext4  defaults,noatime,ro  0  2

# SSD volume with TRIM
UUID=...  /data/ssd  ext4  defaults,noatime,discard  0  2
# Or use periodic fstrim instead of continuous discard:
# systemctl enable fstrim.timer
```

---

## 3. Kubernetes Storage

**Category**: Kubernetes Storage

Kubernetes storage is complex because it abstracts physical storage through multiple layers: StorageClass, PersistentVolume (PV), PersistentVolumeClaim (PVC), and CSI drivers. Misconfigurations at any layer cause data loss, stuck pods, or unrecoverable volumes.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| StorageClass reclaim policy is `Retain` for production data | CRITICAL | `Delete` reclaim policy destroys the underlying volume when PVC is deleted |
| PVC size is adequate with growth headroom | MEDIUM | Undersized PVCs cause application failures; resizing is not always seamless |
| Volume expansion is enabled on StorageClass (`allowVolumeExpansion: true`) | MEDIUM | Without expansion, PVC resize requires data migration |
| Access modes match the workload (`ReadWriteOnce`, `ReadWriteMany`, `ReadOnlyMany`) | HIGH | Wrong access mode causes pod scheduling failures or data corruption |
| CSI driver is production-grade and actively maintained | HIGH | Unmaintained CSI drivers have bugs, security issues, and compatibility problems |
| VolumeSnapshotClass is configured for CSI volume snapshots | MEDIUM | Without snapshot support, backup requires application-level methods only |
| Pod disruption budgets account for storage reattachment time | MEDIUM | Volume reattachment can take minutes; PDB must account for this |
| StatefulSet volumeClaimTemplates use appropriate StorageClass | HIGH | Default StorageClass may have wrong reclaim policy, performance, or availability |
| PV binding mode is `WaitForFirstConsumer` for topology-aware storage | MEDIUM | `Immediate` binding can bind PV in wrong availability zone, preventing pod scheduling |
| Storage quotas are configured per namespace | MEDIUM | Without quotas, one team can exhaust cluster storage capacity |

### StorageClass Configuration

```yaml
# BAD - Delete reclaim policy for production database
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage
provisioner: ebs.csi.aws.com
reclaimPolicy: Delete          # Deleting PVC destroys EBS volume!
volumeBindingMode: Immediate   # May bind in wrong AZ

# GOOD - Retain reclaim policy with correct binding mode
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: fast-storage-retain
provisioner: ebs.csi.aws.com
reclaimPolicy: Retain                    # Volume preserved when PVC is deleted
volumeBindingMode: WaitForFirstConsumer  # Binds in same AZ as pod
allowVolumeExpansion: true               # Allows online PVC resize
parameters:
  type: gp3
  iops: "3000"
  throughput: "125"
  encrypted: "true"
  kmsKeyId: "arn:aws:kms:us-east-1:123456789012:key/abc-123"
```

### Common Mistakes

**Mistake**: Using `Delete` reclaim policy for stateful workloads.

```yaml
# When PVC is deleted (accidentally, during cleanup, or via kubectl delete -all):
# Delete policy: PV and underlying volume are DESTROYED. Data is GONE.
# Retain policy: PV becomes "Released", underlying volume preserved. Data is SAFE.

# Check current PVCs and their reclaim policies
# kubectl get pv -o custom-columns=NAME:.metadata.name,RECLAIM:.spec.persistentVolumeReclaimPolicy,STATUS:.status.phase
```

**Mistake**: ReadWriteMany (RWX) with a block storage CSI driver that does not support it.

```yaml
# BAD - EBS does not support ReadWriteMany
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
spec:
  accessModes:
    - ReadWriteMany        # EBS CSI driver does not support this
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 100Gi
# Result: PVC stays Pending forever

# GOOD - use EFS for ReadWriteMany on AWS
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: shared-data
spec:
  accessModes:
    - ReadWriteMany
  storageClassName: efs-sc  # EFS CSI driver supports RWX
  resources:
    requests:
      storage: 100Gi
```

**Mistake**: No volume expansion enabled.

```yaml
# Without allowVolumeExpansion, growing a PVC requires:
# 1. Create new larger PVC
# 2. Stop application
# 3. Copy data from old to new
# 4. Update deployment to use new PVC
# 5. Start application
# 6. Delete old PVC
# This is disruptive and error-prone

# With allowVolumeExpansion:
# 1. Edit PVC to increase size
# 2. For file-system backed volumes, pod restart may be needed (controller-dependent)
kubectl patch pvc data-pvc -p '{"spec": {"resources": {"requests": {"storage": "200Gi"}}}}'
```

**Mistake**: StatefulSet without explicit StorageClass.

```yaml
# BAD - uses default StorageClass (which may have reclaimPolicy: Delete)
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  volumeClaimTemplates:
    - metadata:
        name: data
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 100Gi
        # No storageClassName specified!

# GOOD - explicit StorageClass with known reclaim policy
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: postgres
spec:
  serviceName: postgres
  replicas: 3
  template:
    spec:
      containers:
        - name: postgres
          image: postgres:16.1
          volumeMounts:
            - name: data
              mountPath: /var/lib/postgresql/data
  volumeClaimTemplates:
    - metadata:
        name: data
        labels:
          app: postgres
      spec:
        accessModes: ["ReadWriteOnce"]
        storageClassName: fast-storage-retain  # Explicit, with Retain policy
        resources:
          requests:
            storage: 100Gi
```

### Volume Snapshot Configuration

```yaml
# VolumeSnapshotClass
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshotClass
metadata:
  name: ebs-snapshot-class
driver: ebs.csi.aws.com
deletionPolicy: Retain      # Retain snapshots when VolumeSnapshot object is deleted
parameters:
  tagSpecification_1: "Backup=true"

---
# Create a volume snapshot
apiVersion: snapshot.storage.k8s.io/v1
kind: VolumeSnapshot
metadata:
  name: postgres-data-snap-20240115
spec:
  volumeSnapshotClassName: ebs-snapshot-class
  source:
    persistentVolumeClaimName: data-postgres-0

---
# Restore from snapshot (create PVC from snapshot)
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data-postgres-restored
spec:
  storageClassName: fast-storage-retain
  dataSource:
    name: postgres-data-snap-20240115
    kind: VolumeSnapshot
    apiGroup: snapshot.storage.k8s.io
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 100Gi
```

### Kubernetes Storage Resource Quotas

```yaml
# Prevent namespace from consuming unbounded storage
apiVersion: v1
kind: ResourceQuota
metadata:
  name: storage-quota
  namespace: team-alpha
spec:
  hard:
    requests.storage: "500Gi"                              # Total PVC size limit
    persistentvolumeclaims: "20"                           # Maximum number of PVCs
    fast-storage-retain.storageclass.storage.k8s.io/requests.storage: "200Gi"  # Per StorageClass limit
```

---

## 4. Cloud Storage

**Category**: Cloud Storage

Cloud object storage (S3, GCS, Azure Blob) requires configuration that is fundamentally different from block or file storage. Misconfigured lifecycle policies waste money, missing versioning eliminates recovery options, and public access blocks are the difference between secure storage and a data breach.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Public access is blocked at the bucket/account level | CRITICAL | Public buckets are the most common cause of cloud data breaches |
| Encryption at rest is enabled (SSE-S3, SSE-KMS, or client-side) | HIGH | Unencrypted object storage fails compliance and security requirements |
| Versioning is enabled for buckets containing critical or mutable data | HIGH | Without versioning, overwritten or deleted objects are permanently lost |
| Lifecycle policies are configured to transition/expire objects | MEDIUM | Without lifecycle policies, storage costs grow indefinitely |
| Cross-region replication is configured for disaster recovery data | MEDIUM | Single-region storage is lost in a regional outage |
| Access logging is enabled | MEDIUM | Without access logs, unauthorized access cannot be detected or investigated |
| Bucket policies enforce least privilege | HIGH | Overly permissive bucket policies enable data exfiltration |
| MFA delete is enabled for critical buckets | MEDIUM | MFA delete prevents accidental or malicious permanent deletion |
| Storage class is appropriate for access patterns | LOW | Wrong storage class wastes money (hot for cold data) or degrades performance (cold for hot data) |
| Server access logging or CloudTrail data events are enabled | MEDIUM | Audit trail for all object-level operations |

### Public Access Block Configuration

```hcl
# AWS - account-level public access block (defense in depth)
resource "aws_s3_account_public_access_block" "account" {
  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}

# AWS - bucket-level public access block
resource "aws_s3_bucket_public_access_block" "data" {
  bucket = aws_s3_bucket.data.id

  block_public_acls       = true
  block_public_policy     = true
  ignore_public_acls      = true
  restrict_public_buckets = true
}
```

```bash
# GCS - uniform bucket-level access (prevents ACL-based public access)
gsutil uniformbucketlevelaccess set on gs://my-bucket

# Azure - deny blob public access
az storage account update \
  --name mystorageaccount \
  --resource-group myresourcegroup \
  --allow-blob-public-access false
```

### Lifecycle Policy Configuration

```hcl
# AWS S3 lifecycle policy
resource "aws_s3_bucket_lifecycle_configuration" "data" {
  bucket = aws_s3_bucket.data.id

  rule {
    id     = "archive-old-data"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"    # Infrequent Access after 30 days
    }

    transition {
      days          = 90
      storage_class = "GLACIER_IR"     # Glacier Instant Retrieval after 90 days
    }

    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"   # Deep Archive after 1 year
    }
  }

  rule {
    id     = "cleanup-incomplete-uploads"
    status = "Enabled"

    abort_incomplete_multipart_upload {
      days_after_initiation = 7        # Clean up stale multipart uploads
    }
  }

  rule {
    id     = "expire-old-versions"
    status = "Enabled"

    noncurrent_version_transition {
      noncurrent_days = 30
      storage_class   = "STANDARD_IA"
    }

    noncurrent_version_expiration {
      noncurrent_days = 90              # Delete old versions after 90 days
    }
  }
}
```

### Cloud Storage Class Reference

| AWS Storage Class | Access Pattern | Retrieval Time | Cost (relative) |
|-------------------|---------------|----------------|-----------------|
| S3 Standard | Frequent access | Milliseconds | 1.0x |
| S3 Standard-IA | Infrequent (monthly) | Milliseconds | 0.5x storage, retrieval fee |
| S3 One Zone-IA | Infrequent, non-critical | Milliseconds | 0.4x storage, retrieval fee |
| S3 Glacier Instant Retrieval | Quarterly access | Milliseconds | 0.2x storage, retrieval fee |
| S3 Glacier Flexible Retrieval | Annual access | 1-12 hours | 0.1x storage, retrieval fee |
| S3 Glacier Deep Archive | Compliance archives | 12-48 hours | 0.03x storage, retrieval fee |
| S3 Intelligent-Tiering | Unknown/changing patterns | Milliseconds | Auto-tiered, monitoring fee |

### Common Mistakes

**Mistake**: No lifecycle policy.

```yaml
# BAD - objects accumulate forever
# A bucket with 10 TB/month growth and no lifecycle = 120 TB after a year
# Most of which is never accessed again
bucket:
  name: application-logs
  lifecycle: none

# GOOD - automated tiering and expiration
bucket:
  name: application-logs
  lifecycle:
    - transition: STANDARD_IA after 30 days
    - transition: GLACIER_IR after 90 days
    - expire: after 365 days
    - abort_incomplete_multipart: after 7 days
```

**Mistake**: Versioning enabled without noncurrent version expiration.

```yaml
# BAD - versioning enabled but old versions kept forever
# A 1 GB file overwritten daily = 365 GB of versions per year
versioning: enabled
noncurrent_version_expiration: none  # Versions accumulate forever

# GOOD - versioning with noncurrent version lifecycle
versioning: enabled
noncurrent_version_expiration: 90 days  # Old versions expire after 90 days
noncurrent_version_transition: STANDARD_IA after 30 days
```

**Mistake**: No access logging.

```hcl
# GOOD - S3 server access logging
resource "aws_s3_bucket_logging" "data" {
  bucket = aws_s3_bucket.data.id

  target_bucket = aws_s3_bucket.access_logs.id
  target_prefix = "s3-access-logs/data-bucket/"
}

# GOOD - CloudTrail data events for S3 (more detailed, costlier)
resource "aws_cloudtrail" "s3_audit" {
  name           = "s3-data-events"
  s3_bucket_name = aws_s3_bucket.cloudtrail.id

  event_selector {
    read_write_type           = "All"
    include_management_events = false

    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::sensitive-data-bucket/"]
    }
  }
}
```

**Mistake**: Cross-region replication without considering costs.

Why it is a problem: Cross-region replication replicates every object (including noncurrent versions) to the destination region. Without lifecycle policies on the destination bucket, storage costs double. Without filtering, non-critical data is replicated unnecessarily.

```hcl
# GOOD - selective cross-region replication with filters
resource "aws_s3_bucket_replication_configuration" "critical_data" {
  bucket = aws_s3_bucket.source.id
  role   = aws_iam_role.replication.arn

  rule {
    id     = "replicate-critical"
    status = "Enabled"

    filter {
      prefix = "critical/"     # Only replicate critical data
    }

    destination {
      bucket        = aws_s3_bucket.destination.arn
      storage_class = "STANDARD_IA"   # Lower cost storage class at destination
    }

    delete_marker_replication {
      status = "Enabled"
    }
  }
}
```

---

## 5. Performance Considerations

**Category**: Storage Performance

Storage performance misconfiguration is one of the most common causes of application latency. Using slow storage for databases, missing caching layers, incorrect I/O schedulers, or single-path SAN connections all create performance bottlenecks that are difficult to diagnose at the application layer.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Databases are on NVMe or SSD storage, not spinning disk | HIGH | Database I/O is random; HDDs deliver 100-200 IOPS, NVMe delivers 100K+ IOPS |
| Storage tiering matches data access patterns (hot/warm/cold) | MEDIUM | Hot data on cold storage = poor performance; cold data on hot storage = wasted cost |
| Multipath I/O (MPIO) is configured for SAN-attached storage | HIGH | Without multipath, a single path failure causes I/O errors and downtime |
| I/O scheduler matches the storage type | MEDIUM | Correct scheduler improves latency and throughput |
| Read caching (L2ARC, bcache, dm-cache) is considered for frequently read cold data | LOW | Caching can dramatically improve read performance for working sets larger than RAM |
| IOPS and throughput limits are understood and documented | MEDIUM | Cloud volumes have IOPS and throughput caps that cause latency spikes when exceeded |
| Block size aligns with application I/O patterns | MEDIUM | Misaligned block size causes read-modify-write amplification |
| Write barriers/sync are not disabled on volumes with critical data | HIGH | Disabling write barriers risks data loss on power failure |

### I/O Scheduler Selection

| Scheduler | Storage Type | Use Case |
|-----------|-------------|----------|
| `none` (noop) | NVMe | NVMe has internal scheduling; OS scheduler adds latency |
| `mq-deadline` | SSD, SAN | Good balance of latency and throughput for non-NVMe flash |
| `bfq` | HDD, mixed workloads | Fair queuing; good for desktops and mixed workloads |
| `kyber` | Fast SSD | Low-overhead scheduler for fast storage |

```bash
# Check current scheduler
cat /sys/block/sda/queue/scheduler
# Output: [mq-deadline] kyber bfq none

# Set scheduler for NVMe
echo none > /sys/block/nvme0n1/queue/scheduler

# Persistent via udev rule
# /etc/udev/rules.d/60-io-scheduler.rules
ACTION=="add|change", KERNEL=="nvme[0-9]*", ATTR{queue/scheduler}="none"
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="0", ATTR{queue/scheduler}="mq-deadline"
ACTION=="add|change", KERNEL=="sd[a-z]", ATTR{queue/rotational}=="1", ATTR{queue/scheduler}="bfq"
```

### Multipath I/O Configuration

```bash
# BAD - single path to SAN storage
# If this path fails, all I/O stops
lsscsi
# [2:0:0:0] disk  SAN_VENDOR LUN0  /dev/sdb

# GOOD - multipath configured
# /etc/multipath.conf
defaults {
  polling_interval    10
  path_selector       "round-robin 0"
  path_grouping_policy  multibus
  failback           immediate
  no_path_retry       queue
  rr_min_io           100
}

blacklist {
  devnode "^(ram|raw|loop|fd|md|dm-|sr|scd|st)[0-9]*"
  devnode "^sd[a]$"   # Exclude local boot disk
}

# Verify multipath status
multipath -ll
# mpath0 (36001405abcdef...) dm-0 SAN_VENDOR,LUN0
# size=500G features='0' hwhandler='0' wp=rw
# |-+- policy='round-robin 0' prio=1 status=active
# | `- 2:0:0:0  sdb 8:16  active ready running
# `-+- policy='round-robin 0' prio=1 status=enabled
#   `- 3:0:0:0  sdc 8:32  active ready running
```

### Cloud Volume IOPS and Throughput

```yaml
# AWS EBS volume types and performance
ebs_volumes:
  gp3:
    baseline_iops: 3000
    max_iops: 16000         # Additional IOPS can be provisioned
    baseline_throughput: 125  # MB/s
    max_throughput: 1000      # MB/s
    use_case: "General purpose, most workloads"

  io2:
    max_iops: 64000          # Per volume
    max_throughput: 1000      # MB/s
    iops_per_gb: 500         # Maximum IOPS/GB ratio
    durability: "99.999%"
    use_case: "High-performance databases, latency-sensitive"

  st1:
    max_throughput: 500       # MB/s
    max_iops: 500            # Baseline, burstable
    use_case: "Sequential workloads, data warehouses, log processing"

  # Common mistake: provisioning gp3 for a database that needs 50K IOPS
  # gp3 maxes out at 16K IOPS; io2 supports up to 64K
```

### Common Mistakes

**Mistake**: Database on HDD storage.

```yaml
# BAD - PostgreSQL data on spinning disk
# Random read IOPS on HDD: ~100-200
# PostgreSQL index lookups: potentially thousands of random reads per second
volume:
  type: st1        # Throughput-optimized HDD
  mount: /var/lib/postgresql/data

# GOOD - database on NVMe or SSD
volume:
  type: gp3
  iops: 10000      # Provision based on measured workload
  mount: /var/lib/postgresql/data
```

**Mistake**: No multipath for SAN storage.

Why it is a problem: SAN storage typically has multiple physical paths from the server to the storage array (through different HBAs, switches, and ports). Without multipath, only one path is used. If that path fails (cable, HBA, switch port), all I/O stops. With multipath, I/O continues on the surviving path with no interruption.

**Mistake**: Disabling write barriers for performance.

```bash
# BAD - disabling barriers for performance
mount -o nobarrier /dev/sda1 /data
# Or in fstab:
UUID=...  /data  ext4  defaults,nobarrier  0  2

# Why this is dangerous:
# Write barriers ensure that filesystem metadata writes are committed to stable storage
# before dependent data writes. Without barriers, a power failure can corrupt the filesystem.

# GOOD - keep barriers enabled (default); use better hardware for performance
# If write performance is a concern:
# - Use NVMe/SSD storage (barriers are nearly free on flash)
# - Use a battery-backed RAID controller
# - Tune the filesystem (journal options, mount options)
```

---

## 6. Capacity Planning

**Category**: Capacity Planning

Storage capacity planning is about avoiding two outcomes: running out of space (outage) and over-provisioning (waste). Both require monitoring, trending, and an understanding of growth patterns.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Storage capacity is monitored with alerts at 70%, 80%, and 90% | HIGH | Without alerts, full filesystems cause application failures and data loss |
| Growth rate is tracked and projected | MEDIUM | Without trending, capacity exhaustion is a surprise |
| Thin provisioning overcommit ratio is understood and monitored | HIGH | Thin provisioning risks exhaustion if actual usage exceeds physical capacity |
| Storage quotas are enforced per user, project, or namespace | MEDIUM | Without quotas, one consumer can exhaust shared storage |
| Capacity alerts reach the right people (not just syslog) | HIGH | Syslog alerts are invisible; capacity alerts must be actionable |
| Historical capacity data is retained for trend analysis | LOW | At least 90 days of history enables meaningful growth projection |
| Capacity planning accounts for snapshot overhead | MEDIUM | Snapshots consume space proportional to change rate; not accounted for in naive monitoring |
| Disk space monitoring accounts for reserved blocks (ext4) and ZFS overhead | LOW | Reported "free space" may not reflect actually usable space |

### Capacity Monitoring Configuration

```yaml
# Prometheus alerting rules for storage capacity
groups:
  - name: storage-capacity
    rules:
      - alert: DiskSpaceWarning
        expr: |
          (node_filesystem_avail_bytes{fstype=~"ext4|xfs|zfs"} /
           node_filesystem_size_bytes{fstype=~"ext4|xfs|zfs"}) < 0.20
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Filesystem {{ $labels.mountpoint }} is {{ $value | humanizePercentage }} free"
          description: "Filesystem on {{ $labels.device }} mounted at {{ $labels.mountpoint }} has less than 20% free space."

      - alert: DiskSpaceCritical
        expr: |
          (node_filesystem_avail_bytes{fstype=~"ext4|xfs|zfs"} /
           node_filesystem_size_bytes{fstype=~"ext4|xfs|zfs"}) < 0.10
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "CRITICAL: Filesystem {{ $labels.mountpoint }} is {{ $value | humanizePercentage }} free"
          description: "Filesystem on {{ $labels.device }} mounted at {{ $labels.mountpoint }} has less than 10% free space. Immediate action required."

      - alert: DiskSpacePrediction
        expr: |
          predict_linear(node_filesystem_avail_bytes{fstype=~"ext4|xfs|zfs"}[7d], 30*24*3600) < 0
        for: 1h
        labels:
          severity: warning
        annotations:
          summary: "Filesystem {{ $labels.mountpoint }} predicted to fill within 30 days"
          description: "Based on 7-day growth trend, filesystem at {{ $labels.mountpoint }} will exhaust available space within 30 days."

      - alert: InodePressure
        expr: |
          (node_filesystem_files_free{fstype=~"ext4|xfs"} /
           node_filesystem_files{fstype=~"ext4|xfs"}) < 0.10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Inode usage above 90% on {{ $labels.mountpoint }}"
          description: "Filesystem may run out of inodes before running out of space. This typically indicates a large number of small files."
```

### Thin Provisioning Risk Assessment

```
Thin provisioning allows allocating more virtual storage than physical storage
exists. This is useful for efficiency but creates a commitment risk.

Example:
  Physical storage: 10 TB
  Allocated thin volumes: 50 TB (5:1 overcommit ratio)
  Current actual usage: 6 TB (60% physical utilization)

Risk calculation:
  If actual usage grows at 500 GB/month:
    Time to physical exhaustion: (10 TB - 6 TB) / 0.5 TB = 8 months

  If all volumes grow to 20% of allocated:
    Actual usage needed: 50 TB * 0.20 = 10 TB
    Physical capacity: 10 TB
    Result: physical capacity exhausted

Monitoring requirements:
  - Physical pool utilization (CRITICAL > 85%, HIGH > 70%)
  - Individual volume growth rates
  - Projected exhaustion date
  - Overcommit ratio trends
```

### Common Mistakes

**Mistake**: No capacity monitoring or wrong thresholds.

```yaml
# BAD - alerts only at 95% (too late to respond)
alert:
  disk_space:
    critical: 95%
    # No warning threshold
    # By the time you get the alert, services may already be failing

# GOOD - tiered alerts with actionable thresholds
alert:
  disk_space:
    info: 70%       # Plan capacity extension
    warning: 80%    # Begin capacity extension
    critical: 90%   # Urgent: extend or free space now
    emergency: 95%  # Pager: immediate action, services at risk
```

**Mistake**: Not monitoring inodes.

```bash
# A volume can be 0% full by bytes but 100% full by inodes
# This happens with many small files (mail spools, session files, cache directories)
df -i /data
# Filesystem    Inodes    IUsed    IFree  IUse%  Mounted on
# /dev/sda1     6553600   6553600  0      100%   /data
# Result: no new files can be created, even though the disk has free space

# For ext4, inode count is fixed at mkfs time
# For XFS, inodes are allocated dynamically (less likely to exhaust)
# For ZFS, inodes are unlimited (dataset-level)
```

**Mistake**: Thin provisioning without overcommit monitoring.

```bash
# BAD - thin pool with 5:1 overcommit and no monitoring
# The pool will eventually fill, causing write failures on all volumes simultaneously

# GOOD - monitor overcommit ratio and physical utilization
#!/bin/bash
# LVM thin pool overcommit monitoring
POOL="vg0/thinpool0"
PHYSICAL=$(lvs --noheadings --nosuffix --units g -o lv_size $POOL | tr -d ' ')
VIRTUAL=$(lvs --noheadings --nosuffix --units g -o pool_lv $POOL | \
  xargs -I{} lvs --noheadings --nosuffix --units g -o lv_size vg0/{} | \
  awk '{sum+=$1} END{print sum}')
OVERCOMMIT=$(echo "scale=2; $VIRTUAL / $PHYSICAL" | bc)
echo "Thin pool overcommit ratio: ${OVERCOMMIT}:1"

DATA_PCT=$(lvs --noheadings -o data_percent $POOL | tr -d ' ')
echo "Physical utilization: ${DATA_PCT}%"
```

### Growth Trend Analysis

```bash
# Collect daily storage metrics for trend analysis
# /etc/cron.daily/storage-metrics
#!/bin/bash
DATE=$(date +%Y-%m-%d)
df --output=target,pcent,avail | tail -n +2 | while read MOUNT PCT AVAIL; do
  echo "${DATE},${MOUNT},${PCT},${AVAIL}" >> /var/log/storage-trends.csv
done

# Simple linear projection in Python
# import pandas as pd
# from scipy import stats
#
# df = pd.read_csv('/var/log/storage-trends.csv',
#                  names=['date', 'mount', 'percent', 'available'])
# df['date'] = pd.to_datetime(df['date'])
# df['days'] = (df['date'] - df['date'].min()).dt.days
#
# for mount in df['mount'].unique():
#     subset = df[df['mount'] == mount]
#     slope, intercept, _, _, _ = stats.linregress(subset['days'], subset['available'])
#     if slope < 0:
#         days_until_full = -intercept / slope
#         print(f"{mount}: estimated {days_until_full:.0f} days until full")
```

---

## 7. Quotas and Access Control

**Category**: Storage Access Control

Storage quotas prevent resource exhaustion by individual users, projects, or applications. Without quotas, a single runaway process, misconfigured application, or misbehaving user can fill a shared filesystem and cause an outage for all consumers.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| User or group quotas are enabled on shared filesystems | MEDIUM | Without quotas, one user can exhaust shared storage |
| Project quotas are configured for multi-tenant environments | MEDIUM | Tenants should not be able to affect each other's storage availability |
| Quota alerts notify users before they hit hard limits | LOW | Soft limits with warnings prevent surprise failures |
| XFS project quotas are used for directory-based quota enforcement | LOW | XFS project quotas are more flexible than user/group quotas for application use |
| ZFS dataset quotas (`quota`, `refquota`) are set per dataset | MEDIUM | Without quotas, one dataset can consume the entire pool |
| Kubernetes storage quotas are set per namespace | MEDIUM | Without quotas, one team can consume all cluster storage |
| NFS export options enforce appropriate access controls | HIGH | Overly permissive NFS exports allow unauthorized data access |

### ZFS Quota Configuration

```bash
# Quota vs Refquota:
# quota: limits total space including snapshots
# refquota: limits space excluding snapshots (usually what you want)

# Set refquota per dataset
zfs set refquota=100G tank/users/alice
zfs set refquota=200G tank/projects/webapp

# Set reservation (guaranteed minimum space)
zfs set refreservation=50G tank/databases/postgres

# Monitor quota usage
zfs list -o name,used,refer,quota,refquota tank/users
```

### XFS Project Quotas

```bash
# Enable project quotas at mount time
# /etc/fstab
UUID=...  /data  xfs  defaults,pquota  0  2

# Define projects
# /etc/projects
10:/data/webapp
11:/data/logs

# /etc/projid
webapp:10
logs:11

# Initialize and set quotas
xfs_quota -x -c 'project -s webapp' /data
xfs_quota -x -c 'limit -p bhard=100g webapp' /data
xfs_quota -x -c 'project -s logs' /data
xfs_quota -x -c 'limit -p bhard=50g logs' /data

# Check usage
xfs_quota -x -c 'report -pbih' /data
```

### NFS Export Security

```bash
# BAD - overly permissive NFS export
# /etc/exports
/data  *(rw,no_root_squash,sync)
# Allows: any host, root access, read-write

# GOOD - restrictive NFS export
# /etc/exports
/data  10.0.1.0/24(rw,root_squash,sync,no_subtree_check)
/data/readonly  10.0.2.0/24(ro,root_squash,sync,no_subtree_check)
# Restricted to specific subnets, root squashed, explicit access mode
```

---

## Quick Reference: Severity Summary

| Severity | Volume Management Findings |
|----------|---------------------------|
| CRITICAL | StorageClass reclaimPolicy is `Delete` for production data; public access not blocked on cloud object storage; encryption keys stored alongside encrypted backups on cloud storage |
| HIGH | Thin pool not monitored; thin pool auto-extend not configured; snapshot COW space undersized (LVM); no capacity monitoring or alerting; devices in fstab by name instead of UUID; `_netdev` missing for network volumes; database on HDD storage; no multipath for SAN; write barriers disabled on production data; cloud storage unencrypted; cloud storage versioning disabled; overly permissive bucket policies; CSI driver unmaintained; PVC access mode mismatch; StatefulSet without explicit StorageClass; thin provisioning overcommit not monitored; capacity alerts not sent to actionable channel; NFS export overly permissive (no_root_squash, world-accessible) |
| MEDIUM | VG free space policy not documented; LVM snapshot retention not managed; LVM metadata not backed up; filesystem type inappropriate for workload; `nodev,nosuid,noexec` missing on data volumes; TRIM not configured for SSDs; `nofail` missing for non-critical volumes; PVC undersized without expansion enabled; volume binding mode `Immediate` for topology-aware storage; no Kubernetes storage quotas; no lifecycle policy on cloud storage; cross-region replication missing for DR; access logging disabled; MFA delete not enabled; I/O scheduler mismatch; IOPS limits not documented; block size misaligned; growth rate not tracked; snapshot overhead not in capacity planning; ZFS dataset quotas missing; project quotas missing in multi-tenant; journal mode inappropriate |
| LOW | Thin pool chunk size not tuned; LVM volume names not meaningful; `noatime`/`relatime` missing; ext4 reserved blocks not tuned; fsck schedule not configured; XFS inode64 not set on old kernels; storage class not optimized for access pattern; historical capacity data not retained; reserved blocks not accounted for in monitoring; soft quota warnings not configured; mount options not explicit; inodes not monitored |
| INFO | Well-configured thin pool with monitoring and auto-extend; appropriate filesystem selection with documented rationale; StorageClass with Retain policy and volume expansion; lifecycle policies with tiered storage transitions; multipath with round-robin and failover; capacity prediction alerts with trend analysis; effective use of XFS project quotas for multi-tenant isolation |

---

## References

- LVM Administrator's Guide: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_and_managing_logical_volumes/
- XFS Documentation: https://xfs.wiki.kernel.org/
- ext4 Documentation: https://ext4.wiki.kernel.org/
- Kubernetes Storage Documentation: https://kubernetes.io/docs/concepts/storage/
- Kubernetes CSI Drivers List: https://kubernetes-csi.github.io/docs/drivers.html
- AWS S3 Best Practices: https://docs.aws.amazon.com/AmazonS3/latest/userguide/security-best-practices.html
- AWS EBS Volume Types: https://docs.aws.amazon.com/ebs/latest/userguide/ebs-volume-types.html
- Linux Device Mapper (DM) Multipath: https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux/9/html/configuring_device_mapper_multipath/
- CIS Benchmarks for Linux: https://www.cisecurity.org/benchmark/distribution_independent_linux
