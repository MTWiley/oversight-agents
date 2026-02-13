# RAID, Pool Layouts, and Replication

Reference checklist for the `storage-reviewer` agent when evaluating RAID configurations, ZFS pool layouts, and data replication strategies. This file complements the inline criteria in `review-storage.md` with detailed severity classifications, failure probability calculations, configuration examples, and remediation patterns.

---

## 1. RAID Level Selection

**Category**: RAID Configuration

RAID level selection is one of the most consequential storage decisions. The wrong RAID level for a workload results in either inadequate protection (data loss on failure) or unacceptable performance. RAID selection must account for drive size, drive count, workload type, rebuild time, and the probability of a second drive failure during rebuild.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| RAID 0 is not used for any data that cannot be fully recreated from scratch | CRITICAL | RAID 0 provides zero redundancy; any single drive failure loses the entire array |
| RAID 5 is not used with drives larger than 2 TB in production | HIGH | URE probability during rebuild makes RAID 5 unreliable with large drives |
| RAID 5 is not used with more than 4 drives | HIGH | More drives = longer rebuild = higher URE probability during rebuild |
| RAID 6 or RAID 10 is used for arrays with drives >= 4 TB | HIGH | Dual parity or mirroring is necessary to survive rebuild-time failures with large drives |
| RAID 10 is used for write-intensive workloads (databases, transaction logs) | MEDIUM | RAID 5/6 write penalty (read-modify-write cycle) degrades write-heavy performance |
| RAID 50/60 is used appropriately for large arrays (> 8 drives) | MEDIUM | Spanning RAID 5/6 across sub-arrays distributes rebuild load and limits blast radius |
| Hot spares are configured for all production RAID arrays | HIGH | Without hot spares, rebuild cannot begin until a replacement drive is physically installed |
| RAID controller has battery-backed or flash-backed write cache | MEDIUM | Write cache without battery protection risks data loss on power failure |
| RAID controller firmware is current | LOW | Firmware updates fix data corruption bugs and improve rebuild performance |
| Write-back cache is only enabled with battery/flash protection | HIGH | Write-back without BBU/FBU risks silent data loss on unexpected power loss |

### RAID Level Reference

| RAID Level | Min Drives | Usable Capacity | Fault Tolerance | Write Penalty | Best For |
|------------|-----------|-----------------|-----------------|---------------|----------|
| RAID 0 | 2 | 100% | None | 1x | Scratch/temp data, disposable caches |
| RAID 1 | 2 | 50% | 1 drive | 2x | OS mirrors, boot drives, small critical volumes |
| RAID 5 | 3 | (N-1)/N | 1 drive | 4x (read-modify-write) | Read-heavy, small drives (< 2 TB), small arrays |
| RAID 6 | 4 | (N-2)/N | 2 drives | 6x (read-modify-write) | General purpose, medium to large drives |
| RAID 10 | 4 | 50% | 1 per mirror pair | 2x | Databases, write-heavy, low latency |
| RAID 50 | 6 | Varies | 1 per sub-array | 4x per sub-array | Large arrays needing RAID 5 economics with better rebuild |
| RAID 60 | 8 | Varies | 2 per sub-array | 6x per sub-array | Very large arrays needing maximum protection |

### URE Probability Calculations for RAID 5

Unrecoverable Read Errors (UREs) are the fundamental reason RAID 5 is dangerous with large drives. During a RAID 5 rebuild, every sector on every surviving drive must be read. If a URE occurs during rebuild, the array fails and data is lost.

**Enterprise drives**: URE rate = 1 in 10^15 bits (1 per 1.25 PB read)
**Consumer drives**: URE rate = 1 in 10^14 bits (1 per 12.5 TB read)

**Probability of encountering a URE during RAID 5 rebuild**:

```
Data read during rebuild = (N - 1) * drive_capacity

For a 4-drive RAID 5 array with 8 TB consumer drives:
  Data read = 3 * 8 TB = 24 TB
  URE rate = 1 per 12.5 TB
  Expected UREs = 24 / 12.5 = 1.92
  Probability of at least one URE = 1 - e^(-1.92) = ~85%

For a 4-drive RAID 5 array with 8 TB enterprise drives:
  Data read = 3 * 8 TB = 24 TB
  URE rate = 1 per 1,250 TB
  Expected UREs = 24 / 1250 = 0.019
  Probability of at least one URE = ~1.9%

For a 4-drive RAID 5 array with 2 TB enterprise drives:
  Data read = 3 * 2 TB = 6 TB
  URE rate = 1 per 1,250 TB
  Expected UREs = 6 / 1250 = 0.0048
  Probability of at least one URE = ~0.5%
```

**Rule of thumb**: RAID 5 is acceptable only when the total data read during rebuild is well below the URE threshold. With consumer drives larger than 2 TB, this is almost never the case. With enterprise drives larger than 4 TB, RAID 6 should be the minimum.

### Common Mistakes

**Mistake**: RAID 5 with large consumer drives.

```yaml
# BAD - 6x 12TB consumer drives in RAID 5
# Rebuild reads 60 TB across 5 surviving drives
# With consumer URE rate (1 per 12.5 TB): ~99.2% chance of URE during rebuild
raid_config:
  level: 5
  drives:
    count: 6
    size: 12TB
    type: consumer  # WD Red, Seagate Barracuda, etc.

# GOOD - same drives in RAID 6
raid_config:
  level: 6
  drives:
    count: 6
    size: 12TB
    type: consumer
  # Survives one URE during rebuild; still protected by second parity
```

**Mistake**: RAID 0 for anything that matters.

```yaml
# BAD - RAID 0 for a database volume
raid_config:
  level: 0
  drives: 4
  mount: /var/lib/postgresql

# GOOD - RAID 10 for database volumes
raid_config:
  level: 10
  drives: 4
  mount: /var/lib/postgresql
```

**Mistake**: No hot spare.

```bash
# BAD - RAID 6 without hot spare; rebuild waits for physical drive replacement
mdadm --create /dev/md0 --level=6 --raid-devices=6 /dev/sd[a-f]1

# GOOD - RAID 6 with hot spare; rebuild begins immediately on failure
mdadm --create /dev/md0 --level=6 --raid-devices=6 --spare-devices=1 \
  /dev/sd[a-g]1
```

**Mistake**: Write-back cache without battery protection.

```
# BAD - RAID controller configuration
Write Cache: Enabled (Write-Back)
Battery/Capacitor: Not Present

# GOOD - either option
Write Cache: Enabled (Write-Back)
Battery/Capacitor: Present, Healthy

# OR - safe without battery
Write Cache: Enabled (Write-Through)
Battery/Capacitor: Not Present
```

### Rebuild Time Estimates

Rebuild time is a function of array size, I/O load, and controller/drive speed. During rebuild, the array is in a degraded state and a second failure is catastrophic (RAID 5) or reduces tolerance to zero (RAID 6).

| Array Size (per drive) | RAID Level | Estimated Rebuild Time (idle) | Rebuild Time (under load) |
|----------------------|------------|-------------------------------|---------------------------|
| 2 TB | RAID 5/6 | 4-8 hours | 12-24 hours |
| 4 TB | RAID 5/6 | 8-16 hours | 24-48 hours |
| 8 TB | RAID 5/6 | 16-32 hours | 2-4 days |
| 12 TB | RAID 5/6 | 24-48 hours | 3-7 days |
| 16 TB | RAID 5/6 | 32-64 hours | 4-10 days |

Production arrays under continuous I/O load rebuild significantly slower. A 7-day rebuild window for a RAID 5 array with 16 TB drives is a 7-day window during which a second drive failure means total data loss.

---

## 2. Hardware vs Software RAID

**Category**: RAID Configuration

The choice between hardware and software RAID affects performance, portability, monitoring, and failure modes. Neither is universally better; the decision depends on the environment.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Hardware RAID controllers have current firmware | MEDIUM | Firmware bugs in RAID controllers can cause silent data corruption |
| Hardware RAID controller battery/capacitor status is monitored | HIGH | Failed BBU/FBU with write-back cache = data loss risk on power failure |
| Software RAID (mdadm) has monitoring configured (`mdadm --monitor`) | HIGH | Without monitoring, degraded arrays go unnoticed until a second failure |
| `/etc/mdadm/mdadm.conf` or equivalent is present and current | MEDIUM | Missing config prevents automatic array assembly on boot |
| RAID monitoring is integrated with alerting (email, SNMP, monitoring agent) | HIGH | Degraded array alerts must reach operations staff, not just syslog |
| Hardware RAID metadata format is documented for disaster recovery | LOW | If the controller fails, a replacement controller must understand the metadata |
| Software RAID uses metadata format 1.2 (superblock at start of device) | LOW | Format 1.2 is the modern default; 0.90 has limitations on large arrays |

### Comparison

| Factor | Hardware RAID | Software RAID (mdadm/LVM) |
|--------|--------------|---------------------------|
| CPU overhead | Controller handles parity | Host CPU handles parity (negligible on modern CPUs) |
| Write cache | BBU/FBU backed write cache | Must use filesystem journaling or separate log device |
| Portability | Locked to controller model/vendor | Portable across any Linux system |
| Monitoring | Vendor-specific tools (MegaCLI, storcli, hpssacli) | Standard Linux tools (mdadm, /proc/mdstat) |
| Failure mode | Controller failure = array inaccessible until replacement controller | Drive failures handled in software; no SPOF from controller |
| Boot support | Transparent to OS | Requires initramfs configuration |
| Cost | Additional hardware cost | No additional cost |

### Common Mistakes

**Mistake**: Using hardware RAID but not monitoring the controller battery.

```bash
# MegaCLI - check BBU status
MegaCli64 -AdpBbuCmd -GetBbuStatus -aALL

# storcli - check BBU/CacheCade status
storcli /c0 show bbu

# If BBU is failed/missing and write-back is enabled, this is HIGH severity
```

**Mistake**: Software RAID without email monitoring.

```bash
# BAD - mdadm running but no monitoring
cat /etc/mdadm/mdadm.conf
# No MAILADDR line

# GOOD - monitoring configured
cat /etc/mdadm/mdadm.conf
MAILADDR root@example.com
MAILFROM mdadm@hostname.example.com
```

```bash
# Verify mdadm monitor daemon is running
systemctl status mdmonitor
```

**Mistake**: Using a hardware RAID controller as a passthrough (JBOD mode) for ZFS while still paying for the controller.

Why it is a problem: ZFS needs direct access to drives. A RAID controller in JBOD mode adds a layer of abstraction with no benefit. Use an HBA (Host Bus Adapter) instead, which is simpler, cheaper, and gives ZFS the direct access it requires.

---

## 3. ZFS Pool Layouts

**Category**: ZFS Configuration

ZFS combines filesystem and volume management with built-in checksumming, compression, snapshots, and replication. Its power comes with complexity: wrong vdev layouts, mismatched ashift values, or poorly tuned recordsize can cause permanent performance degradation or data loss.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Pool uses appropriate redundancy (mirror, raidz1, raidz2, raidz3) | HIGH | Single-vdev striped pools (RAID 0 equivalent) lose everything on one drive failure |
| `ashift` matches the physical sector size of the drives (12 for 4Kn/512e drives, 9 for true 512n) | HIGH | Wrong ashift causes permanent write amplification; cannot be changed after pool creation |
| Compression is enabled (`lz4` as default, `zstd` for archival) | MEDIUM | LZ4 compression improves performance for compressible data with negligible CPU cost |
| Deduplication is NOT enabled unless specifically justified with RAM budget | HIGH | ZFS dedup requires ~5 GB of RAM per TB of deduplicated data; OOM kills the pool |
| Scrubs are scheduled (at least monthly, weekly preferred for production) | HIGH | Without scrubs, bitrot and silent corruption go undetected until data is accessed |
| `recordsize` is tuned for the workload (128K default, 16K for databases, 1M for sequential) | MEDIUM | Default recordsize is appropriate for general use but suboptimal for specific workloads |
| SLOG device is enterprise-grade NVMe/SSD with power-loss protection | HIGH | Consumer SSDs without PLP can lose the ZIL on power failure, causing pool corruption |
| L2ARC is only used when the working set exceeds ARC and drives are slow | MEDIUM | L2ARC consumes RAM for index entries (~70 bytes per cached block); misuse wastes RAM |
| `copies=2` is NOT used as a substitute for RAID | MEDIUM | Copies protects against bitrot on the same vdev but not against drive failure |
| Pool has sufficient free space (< 80% used, ideally < 70%) | MEDIUM | ZFS COW performance degrades significantly above 80% capacity |
| `autoexpand=on` is set if drives may be replaced with larger ones | LOW | Without autoexpand, pool does not use additional space from larger replacement drives |
| `autotrim=on` is set for SSD pools | LOW | Without autotrim, SSDs do not receive TRIM commands, degrading performance over time |

### ZFS Vdev Layout Reference

| Layout | Min Drives | Fault Tolerance | Usable Capacity | Best For |
|--------|-----------|-----------------|-----------------|----------|
| mirror | 2 (per vdev) | 1 per mirror | 50% | Databases, boot pools, small fast arrays |
| raidz1 | 3 | 1 per vdev | (N-1)/N | Small arrays (3-5 drives), non-critical data |
| raidz2 | 4 | 2 per vdev | (N-2)/N | General production storage |
| raidz3 | 5 | 3 per vdev | (N-3)/N | Large arrays (8+ drives), critical archives |
| striped mirrors | 4+ (pairs) | 1 per pair | 50% | High-IOPS workloads, databases |

**Pool expansion rules**: A pool can only grow by adding new vdevs or replacing individual drives with larger ones in existing vdevs. You cannot add a drive to an existing raidz vdev (until OpenZFS RAIDZ expansion feature, which is still maturing). Plan vdev layout before pool creation.

### Common Mistakes

**Mistake**: Wrong ashift value.

```bash
# BAD - ashift=9 on a 4Kn or 512e drive (most modern drives)
# This cannot be fixed after pool creation without destroying and recreating the pool
zpool create -o ashift=9 tank raidz2 /dev/sd{a,b,c,d,e,f}

# GOOD - ashift=12 for any modern drive (4K physical sectors)
zpool create -o ashift=12 tank raidz2 /dev/sd{a,b,c,d,e,f}

# VERIFY drive sector size before pool creation
lsblk -o NAME,PHY-SEC
# or
hdparm -I /dev/sda | grep "Sector size"
```

**Mistake**: Enabling deduplication without understanding RAM requirements.

```bash
# BAD - dedup enabled on a system with 32 GB RAM and 50 TB of data
# Dedup table requires ~5 GB RAM per TB = 250 GB RAM needed
# System will thrash, swap, and eventually become unresponsive
zfs set dedup=on tank/data

# GOOD - if dedup is genuinely needed, calculate RAM first
# Formula: RAM_needed = unique_blocks * 320 bytes (DDT entry size)
# For 50 TB with 128K records: ~400M entries * 320 bytes = ~120 GB RAM minimum
# If you cannot afford the RAM, do not enable dedup

# Check existing dedup ratio before enabling (use a test dataset)
zdb -S tank/data
```

**Mistake**: No scrub schedule.

```bash
# BAD - no scrub scheduled; bitrot accumulates silently
crontab -l | grep scrub
# (no output)

# GOOD - weekly scrub for production pools
# /etc/cron.d/zfs-scrub
0 2 * * 0 root /sbin/zpool scrub tank

# Or use the systemd timer (preferred on systemd systems)
systemctl enable zfs-scrub-weekly@tank.timer
```

**Mistake**: SLOG on a consumer SSD without power-loss protection.

```bash
# BAD - consumer NVMe as SLOG (no PLP/capacitor)
zpool add tank log /dev/nvme0n1p1  # Samsung 870 EVO, Crucial P3, etc.

# GOOD - enterprise NVMe with power-loss protection as SLOG
zpool add tank log /dev/nvme0n1p1  # Intel Optane, Samsung PM9A3, etc.

# VERIFY PLP capability
smartctl -a /dev/nvme0n1 | grep "Power Loss"
# Or check vendor specifications
```

**Mistake**: Mismatched vdev sizes in a pool.

```bash
# BAD - different vdev sizes cause uneven data distribution
# Data is distributed across vdevs proportionally to free space
# This means the smaller vdev gets proportionally more I/O per GB
zpool create tank mirror /dev/sda /dev/sdb  # 2 TB mirror
zpool add tank mirror /dev/sdc /dev/sdd     # 8 TB mirror

# GOOD - matched vdev sizes for even distribution
zpool create tank mirror /dev/sda /dev/sdb  # 8 TB mirror
zpool add tank mirror /dev/sdc /dev/sdd     # 8 TB mirror
```

### Recordsize Tuning

| Workload | Recommended Recordsize | Rationale |
|----------|----------------------|-----------|
| General filesystem (default) | 128K | Good balance for mixed workloads |
| PostgreSQL | 16K (matches page size) | Avoids read-modify-write amplification on partial block updates |
| MySQL InnoDB | 16K (matches page size) | Same as PostgreSQL |
| MongoDB (WiredTiger) | 64K | Aligns with WiredTiger's internal page size |
| VM images (block storage) | 64K | Matches typical VM I/O size |
| Large sequential files (video, backups) | 1M | Maximizes throughput; reduces metadata overhead |
| Small files (git repos, source code) | 32K-64K | Reduces wasted space from internal fragmentation |

```bash
# Set recordsize per dataset (not pool-wide)
zfs set recordsize=16K tank/postgres
zfs set recordsize=1M tank/backups
zfs set recordsize=128K tank/general
```

### ZFS Performance Settings

```bash
# Recommended production settings
zfs set compression=lz4 tank           # Enable compression pool-wide
zfs set atime=off tank                 # Disable access time updates
zfs set xattr=sa tank                  # Store extended attributes in inodes
zfs set dnodesize=auto tank            # Auto-size dnodes for metadata-heavy workloads
zfs set relatime=on tank               # If atime must be on, use relatime

# ARC tuning (in /etc/modprobe.d/zfs.conf or kernel parameters)
# Limit ARC to prevent ZFS from consuming all RAM
# Default: ARC uses up to 50% of RAM; adjust based on workload
options zfs zfs_arc_max=17179869184    # 16 GB max ARC
options zfs zfs_arc_min=4294967296     # 4 GB min ARC
```

---

## 4. Replication Strategies

**Category**: Replication

Replication protects data against site-level failures, provides disaster recovery capability, and can serve read traffic from replicas. The choice between synchronous and asynchronous replication defines the tradeoff between data consistency and performance.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Stateful services have a documented replication strategy | HIGH | No replication = single site failure causes data loss |
| Replication factor is at least 3 for critical data | MEDIUM | RF=2 survives one failure; RF=3 survives one failure plus one maintenance event |
| Synchronous replication is used where RPO=0 is required | HIGH | Async replication has a data loss window equal to the replication lag |
| Async replication lag is monitored and alerted on | HIGH | Unmonitored replication lag means unknown RPO |
| Replicas are in separate failure domains (rack, power, site) | HIGH | Replicas in the same rack/power domain can all fail simultaneously |
| Replication network is separated from client data network | MEDIUM | Replication traffic can saturate client network, causing service degradation |
| Replication is encrypted in transit (TLS, WireGuard, IPsec) | HIGH | Unencrypted replication over untrusted networks exposes data |
| Failover procedure is documented and tested | HIGH | Untested failover fails when needed |
| Split-brain prevention is configured (quorum, fencing, witness) | CRITICAL | Split-brain causes divergent writes to both sides, leading to data corruption or loss |
| Replication recovery (re-sync after split) is documented | MEDIUM | Without a re-sync procedure, recovering from a split requires full data copy |

### Synchronous vs Asynchronous Replication

| Factor | Synchronous | Asynchronous |
|--------|-------------|--------------|
| RPO | 0 (no data loss) | Seconds to minutes (replication lag) |
| Write latency impact | Adds round-trip time to every write | No impact on client write latency |
| Network requirement | Low latency, high reliability link | Tolerates higher latency, intermittent connectivity |
| Failure behavior | Writes stall if replica is unreachable (unless degraded mode configured) | Writes continue; replica catches up when available |
| Use case | Financial transactions, source of truth databases | Cross-region DR, log shipping, analytics replicas |
| Risk | Performance degradation on WAN; stalled writes if quorum lost | Data loss up to replication lag on primary failure |

### Ceph CRUSH Rules

Ceph's CRUSH algorithm determines data placement. Proper CRUSH rules ensure replicas are distributed across failure domains.

```bash
# BAD - default CRUSH rule may place replicas on same host
# Check current rules
ceph osd crush rule dump replicated_rule

# GOOD - CRUSH rule that spreads replicas across racks
ceph osd crush rule create-replicated rack_spread default rack host

# Apply to a pool
ceph osd pool set mypool crush_rule rack_spread
```

**CRUSH hierarchy example**:

```
root default
  rack rack01
    host node01
      osd.0
      osd.1
    host node02
      osd.2
      osd.3
  rack rack02
    host node03
      osd.4
      osd.5
    host node04
      osd.6
      osd.7
```

```bash
# Verify failure domain separation
# This should show replicas on different racks
ceph osd map mypool testobject
# Check that primary and replicas are in different failure domains
```

### Common Mistakes

**Mistake**: Replication factor of 2 for critical data.

Why it is a problem: With RF=2, a single node failure leaves data on exactly one copy. If that remaining copy has a disk failure, bitrot, or needs maintenance, data is lost. RF=3 allows one failure plus one planned maintenance event.

```bash
# BAD - RF=2 for critical production data
ceph osd pool set critical_data size 2
ceph osd pool set critical_data min_size 1

# GOOD - RF=3 with min_size=2
ceph osd pool set critical_data size 3
ceph osd pool set critical_data min_size 2
```

**Mistake**: All replicas in the same failure domain.

```yaml
# BAD - Ceph CRUSH tree with all OSDs under one rack
# A rack power failure loses all copies
root default
  rack rack01
    host node01: osd.0, osd.1
    host node02: osd.2, osd.3
    host node03: osd.4, osd.5

# GOOD - OSDs distributed across multiple racks
root default
  rack rack01
    host node01: osd.0, osd.1
  rack rack02
    host node02: osd.2, osd.3
  rack rack03
    host node03: osd.4, osd.5
```

**Mistake**: No split-brain prevention.

```yaml
# BAD - two-node cluster with no witness/quorum
# Network partition = both nodes accept writes = divergent data
cluster:
  nodes:
    - primary
    - secondary
  quorum: none

# GOOD - three-node cluster or two nodes with a witness
cluster:
  nodes:
    - primary
    - secondary
    - witness  # Lightweight node for quorum only
  quorum: majority
```

**Mistake**: Async replication lag not monitored.

```yaml
# BAD - PostgreSQL streaming replication with no lag monitoring
# Replication could be hours behind and nobody would know until failover
primary_conninfo = 'host=primary port=5432'

# GOOD - monitor replication lag and alert
# PostgreSQL replication lag query
# SELECT
#   client_addr,
#   state,
#   pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
#   replay_lag
# FROM pg_stat_replication;

# Alert thresholds
monitoring:
  replication_lag_warning: 30s
  replication_lag_critical: 300s
  replication_lag_bytes_warning: 104857600    # 100 MB
  replication_lag_bytes_critical: 1073741824  # 1 GB
```

### ZFS Replication

ZFS send/receive is one of the most efficient replication mechanisms available, leveraging snapshots for incremental block-level replication.

```bash
# Initial full send
zfs send tank/data@base | ssh backup-host zfs recv backup/data

# Incremental send (only changed blocks since last snapshot)
zfs send -i tank/data@base tank/data@daily-2024-01-15 | \
  ssh backup-host zfs recv backup/data

# Encrypted send (for untrusted remote; data remains encrypted at rest on remote)
zfs send --raw tank/encrypted@snap | ssh remote zfs recv backup/encrypted

# Automated replication tools (preferred over manual scripts)
# - sanoid/syncoid: automated ZFS snapshot and replication management
# - zrepl: daemon-based ZFS replication
# - znapzend: ZFS backup with retention management
```

```bash
# sanoid.conf example for automated snapshot management
[tank/data]
  use_template = production
  recursive = yes

[template_production]
  frequently = 4
  hourly = 24
  daily = 30
  monthly = 12
  yearly = 2
  autoprune = yes
  autosnap = yes

# syncoid for replication (run via cron)
# Replicates incrementally, creates missing snapshots
syncoid --recursive tank/data backup-host:backup/data
```

---

## 5. Monitoring for Degraded Arrays

**Category**: RAID Monitoring

A degraded RAID array is a ticking clock. Without monitoring and alerting, the degraded state persists until a second failure causes data loss. Monitoring must be automated, tested, and integrated with operational alerting.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| RAID array health is monitored by an automated system (not manual checks) | HIGH | Manual checks are forgotten; automated monitoring catches degradation immediately |
| Alerts are sent to a monitored channel (email, PagerDuty, Slack) on degradation | HIGH | Syslog-only alerts are effectively invisible to operations |
| SMART monitoring is enabled for all drives (`smartd` or vendor tool) | MEDIUM | SMART predictive failures give advance warning of drive failures |
| Drive error counters (media errors, predictive failures) are monitored | MEDIUM | Rising error counts precede drive failure; proactive replacement reduces risk |
| Rebuild progress is monitored | MEDIUM | Stalled or very slow rebuilds extend the vulnerability window |
| Spare drive inventory is tracked and physically available | MEDIUM | Hot spares are only useful if replacement drives are physically available |
| RAID monitoring survives reboot (enabled at boot, systemd service, monitoring agent) | MEDIUM | Monitoring that requires manual restart is monitoring that eventually stops |
| Periodic manual verification of alerting (test alerts) | LOW | Verify that the alerting pipeline actually delivers alerts |

### Monitoring Configuration Examples

**mdadm monitoring**:

```bash
# /etc/mdadm/mdadm.conf
MAILADDR ops-team@example.com
MAILFROM mdadm@storage01.example.com

# Enable mdmonitor service
systemctl enable mdmonitor
systemctl start mdmonitor

# Verify monitoring is active
systemctl status mdmonitor

# Test alert delivery
mdadm --monitor --test --oneshot /dev/md0
```

**Check array status programmatically**:

```bash
# Check all arrays
cat /proc/mdstat

# Parse for degraded state
mdadm --detail /dev/md0 | grep "State :"
# Healthy output: "State : clean"
# Degraded output: "State : clean, degraded"
# Rebuilding output: "State : clean, degraded, recovering"
```

**ZFS pool health monitoring**:

```bash
# Check pool health
zpool status -x
# Healthy output: "all pools are healthy"
# Degraded output shows affected vdevs and drives

# Monitor via cron or systemd timer
# /etc/cron.d/zfs-health
*/5 * * * * root /sbin/zpool status -x | grep -v "all pools are healthy" && \
  /sbin/zpool status | mail -s "ZFS DEGRADED on $(hostname)" ops@example.com

# ZFS Event Daemon (zed) for real-time monitoring
# /etc/zfs/zed.d/zed.rc
ZED_EMAIL_ADDR="ops@example.com"
ZED_NOTIFY_VERBOSE=1
ZED_NOTIFY_DATA=1

systemctl enable zfs-zed
systemctl start zfs-zed
```

**Hardware RAID controller monitoring**:

```bash
# MegaRAID (storcli)
storcli /c0 /vall show | grep "State"
# Good: "Optl" (Optimal)
# Bad: "Dgrd" (Degraded), "Offln" (Offline)

# HP SmartArray (ssacli)
ssacli ctrl slot=0 ld all show status
# Good: "OK"
# Bad: "Failed", "Recovering", "Interim Recovery Mode"

# Prometheus integration example (node_exporter textfile collector)
#!/bin/bash
# /etc/cron.d/raid-metrics
# */5 * * * * root /usr/local/bin/raid-metrics.sh > /var/lib/node_exporter/textfile/raid.prom
STATUS=$(mdadm --detail /dev/md0 | grep "State :" | awk '{print $NF}')
if [ "$STATUS" = "clean" ]; then
  echo "raid_array_healthy{device=\"md0\"} 1"
else
  echo "raid_array_healthy{device=\"md0\"} 0"
fi
```

**SMART monitoring**:

```bash
# /etc/smartd.conf
# Monitor all drives, run short self-test weekly, long monthly
# Alert on any attribute change or test failure
DEFAULT -a -o on -S on -s (S/../.././02|L/../../6/03) \
  -W 4,45,55 \
  -m ops@example.com \
  -M exec /usr/share/smartmontools/smartd_warning.sh

# Monitor individual drives
/dev/sda -a -o on -S on -s (S/../.././02|L/../../6/03) -W 4,45,55 -m ops@example.com
/dev/sdb -a -o on -S on -s (S/../.././02|L/../../6/03) -W 4,45,55 -m ops@example.com

# Enable and start
systemctl enable smartd
systemctl start smartd
```

---

## 6. Advanced RAID Topologies

**Category**: RAID Configuration

Large storage arrays (> 8 drives) benefit from nested RAID topologies that limit the blast radius of a rebuild and improve rebuild performance by distributing the load across sub-arrays.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Arrays with > 8 drives use nested RAID (RAID 50, 60, or striped mirrors) not flat RAID 5/6 | MEDIUM | Flat RAID 5/6 across many drives has long rebuild times and high URE risk |
| RAID 50 sub-arrays each have at most 5-6 drives | MEDIUM | Larger sub-arrays increase rebuild time and URE risk within the sub-array |
| RAID 60 is used for arrays > 12 drives with critical data | MEDIUM | RAID 60 provides dual parity per sub-array plus distribution benefits |
| ZFS pools use multiple smaller vdevs instead of one large raidz vdev | MEDIUM | Multiple vdevs parallelize I/O and limit rebuild scope |
| All drives in a vdev or sub-array are the same model and size | LOW | Mismatched drives create performance bottlenecks (slowest drive limits the vdev) |
| Drive firmware is consistent within each vdev or sub-array | LOW | Mixed firmware versions can cause inconsistent behavior during rebuild |

### ZFS Pool Layout Best Practices

```bash
# BAD - single large raidz2 vdev with 12 drives
# Rebuild reads all 10 surviving drives; very slow; high URE risk
zpool create tank raidz2 /dev/sd{a..l}

# GOOD - three 4-drive raidz1 vdevs (equivalent usable capacity, faster IOPS)
zpool create tank \
  raidz1 /dev/sd{a..d} \
  raidz1 /dev/sd{e..h} \
  raidz1 /dev/sd{i..l}

# BETTER - two 6-drive raidz2 vdevs (best balance of capacity and protection)
zpool create tank \
  raidz2 /dev/sd{a..f} \
  raidz2 /dev/sd{g..l}

# BEST for IOPS - six mirror vdevs (50% capacity, maximum performance)
zpool create tank \
  mirror /dev/sda /dev/sdb \
  mirror /dev/sdc /dev/sdd \
  mirror /dev/sde /dev/sdf \
  mirror /dev/sdg /dev/sdh \
  mirror /dev/sdi /dev/sdj \
  mirror /dev/sdk /dev/sdl
```

### RAID 50/60 Layout Examples

```
# RAID 50 with 12 drives (3 sub-arrays of 4 drives each)
# Each sub-array is RAID 5 (3 data + 1 parity)
# Sub-arrays are striped together
# Usable: 75% of each sub-array, total 9/12 = 75%
# Survives: 1 drive failure per sub-array (up to 3 simultaneous if in different sub-arrays)

Sub-array 0: [D] [D] [D] [P]  <- RAID 5
Sub-array 1: [D] [D] [D] [P]  <- RAID 5
Sub-array 2: [D] [D] [D] [P]  <- RAID 5
            |--- Striped ---|

# RAID 60 with 12 drives (2 sub-arrays of 6 drives each)
# Each sub-array is RAID 6 (4 data + 2 parity)
# Usable: 67% of each sub-array, total 8/12 = 67%
# Survives: 2 drive failures per sub-array

Sub-array 0: [D] [D] [D] [D] [P] [P]  <- RAID 6
Sub-array 1: [D] [D] [D] [D] [P] [P]  <- RAID 6
            |------- Striped ---------|
```

---

## Quick Reference: Severity Summary

| Severity | RAID, Pool, and Replication Findings |
|----------|--------------------------------------|
| CRITICAL | RAID 0 on non-disposable data; split-brain possible due to missing quorum/fencing; pool corruption risk from SLOG without power-loss protection on a consumer SSD |
| HIGH | RAID 5 with drives > 2 TB; no hot spares on production arrays; write-back cache without BBU/FBU; wrong ashift on ZFS pool (permanent); dedup enabled without adequate RAM; no scrub schedule; no RAID/pool health monitoring or alerting; no replication strategy for stateful services; replicas in same failure domain; async replication lag not monitored; no failover testing; unencrypted replication over untrusted network; synchronous replication required but not configured |
| MEDIUM | RAID 10 not used for write-intensive database workloads; mismatched vdev sizes; ZFS pool > 80% full; compression not enabled; recordsize not tuned for workload; RF=2 for critical data; replication network shared with client traffic; flat RAID 5/6 across > 8 drives; RAID controller firmware outdated; L2ARC misused; copies=2 used as RAID substitute; no re-sync procedure documented |
| LOW | RAID controller firmware not current; ZFS autoexpand not set; autotrim not set on SSD pools; mismatched drive firmware in vdev; missing mdadm.conf; drive model/size mismatch within vdev; metadata format not documented |
| INFO | Well-designed ZFS pool layout with appropriate vdev sizing; good CRUSH rule design separating failure domains; effective use of sanoid/syncoid for automated replication; proper recordsize tuning per dataset; scrub schedule with monitoring integration |

---

## References

- ZFS Administration Guide: https://openzfs.github.io/openzfs-docs/
- Ceph Documentation - CRUSH Maps: https://docs.ceph.com/en/latest/rados/operations/crush-map/
- mdadm Manual: `man mdadm`, `man mdadm.conf`
- SNIA Best Practices for RAID: https://www.snia.org/
- Backblaze Drive Stats (annual reliability data): https://www.backblaze.com/cloud-storage/resources/hard-drive-test-data
- Dell/HP/Broadcom RAID Controller Documentation for storcli, ssacli, MegaCLI
