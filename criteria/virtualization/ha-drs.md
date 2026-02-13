# High Availability, DRS, and Disaster Recovery

Reference checklist for the `virtualization-reviewer` agent when evaluating high availability (HA), Distributed Resource Scheduler (DRS), Fault Tolerance (FT), replication, disaster recovery (DR), and cluster design across VMware vSphere, Proxmox VE, and KVM/libvirt environments. This file complements the inline criteria in `review-virtualization.md` with detailed checkpoints, severity classifications, platform-specific configurations, and remediation guidance.

Covers: HA admission control, host isolation response, restart priorities, heartbeat datastores, VMCP (APD/PDL), DRS automation levels, affinity/anti-affinity rules, migration thresholds, resource pools, VM-host rules, Fault Tolerance, SRM and replication, RPO/RTO planning, failover procedures, DR testing, cluster sizing (N+1/N+2), host homogeneity, EVC, and network design for HA clusters.

---

## 1. vSphere HA Configuration

**Category**: High Availability

vSphere HA protects VMs against host failures by restarting them on surviving hosts. A misconfigured HA cluster creates a false sense of protection -- failures are detected but VMs cannot be restarted due to insufficient capacity, incorrect isolation response, or missing heartbeat paths.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 1.1 | HA is enabled on all production clusters | **HIGH** | vSphere |
| 1.2 | Admission control is enabled (not disabled or set to "Do not power on VMs") | **HIGH** | vSphere |
| 1.3 | Admission control policy is appropriate: dedicated failover host, percentage-based, or slot-based | **HIGH** | vSphere |
| 1.4 | Admission control reserves capacity for at least one host failure (N+1 minimum) | **HIGH** | vSphere |
| 1.5 | Host isolation response is set appropriately (`Shutdown` or `Leave powered on`, not `Power off`) | **HIGH** | vSphere |
| 1.6 | VM restart priority is configured per VM based on business criticality (infrastructure VMs first) | **MEDIUM** | vSphere |
| 1.7 | VM restart priority orders infrastructure services (AD, DNS, vCenter) before application VMs | **MEDIUM** | vSphere |
| 1.8 | Orchestrated restart dependencies are configured for multi-tier applications | **LOW** | vSphere |
| 1.9 | Heartbeat datastores are configured (minimum 2 datastores, preferably on different storage arrays) | **HIGH** | vSphere |
| 1.10 | Datastore heartbeat selection is set to a specific list of datastores, not automatic selection only | **MEDIUM** | vSphere |
| 1.11 | HA advanced options are reviewed and justified (das.isolationAddress, das.heartbeatDsPerHost) | **LOW** | vSphere |
| 1.12 | HA host monitoring is enabled | **HIGH** | vSphere |
| 1.13 | VM Component Protection (VMCP) is enabled for datastore accessibility | **HIGH** | vSphere |

### What to Look For

```powershell
# PowerCLI - Check HA configuration
Get-Cluster | Select-Object Name, HAEnabled, HAAdmissionControlEnabled,
    HAFailoverLevel, HAIsolationResponse, HARestartPriority

# Check admission control policy details
Get-Cluster | ForEach-Object {
    $_.ExtensionData.Configuration.DasConfig | Select-Object *
}

# Check heartbeat datastores
Get-Cluster | ForEach-Object {
    $_.ExtensionData.Configuration.DasConfig.HeartbeatDatastore
}
```

### Common Mistakes

**Mistake**: Admission control disabled to allow powering on more VMs.

Why it is a problem: Without admission control, every VM slot is consumed. When a host fails, surviving hosts lack CPU and memory capacity to restart the failed VMs. The cluster detects the failure and attempts restart, but VMs fail to power on due to insufficient resources. This turns a single host failure into a prolonged multi-VM outage.

```
# BAD - Admission control disabled
Cluster "Production" -> HA -> Admission Control: Disabled

# GOOD - Percentage-based admission control with N+1 capacity
Cluster "Production" -> HA -> Admission Control: Enabled
    Policy: Cluster resource percentage
    Reserved failover CPU capacity: 25%      # For a 4-host cluster = 1 host
    Reserved failover Memory capacity: 25%
```

**Mistake**: Host isolation response set to `Power Off`.

Why it is a problem: Network partitions can cause a host to appear isolated even when it is healthy and running VMs. If the isolation response is `Power Off`, VMs are abruptly terminated (not gracefully shut down), risking data corruption. If the VMs are then restarted on other hosts while still running on the "isolated" host (split-brain), data corruption is almost certain.

```
# BAD - abrupt power off on isolation
Host Isolation Response: Power Off

# GOOD - graceful shutdown or leave powered on (evaluate per environment)
Host Isolation Response: Shut down and restart VMs
# OR
Host Isolation Response: Leave powered on
# (Use "Leave powered on" when storage is shared and VMCP will handle storage-based isolation)
```

**Mistake**: Only one heartbeat datastore available or heartbeat datastores on the same storage array.

Why it is a problem: HA uses both network and datastore heartbeats to determine host state. If the only heartbeat datastore becomes unavailable (storage failure, path issue), HA cannot distinguish between a dead host and a network-isolated host. This causes false positive host failure declarations, triggering unnecessary VM restarts and potential split-brain.

---

## 2. VM Component Protection (VMCP)

**Category**: Storage Availability

VMCP protects VMs when the underlying datastore becomes inaccessible. Without VMCP, VMs with All Paths Down (APD) or Permanent Device Loss (PDL) conditions continue running in a degraded state, accumulating I/O errors. VMCP can trigger VM restart on hosts with datastore access, preventing prolonged VM degradation.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 2.1 | VMCP is enabled on all production HA clusters | **HIGH** | vSphere |
| 2.2 | APD (All Paths Down) response is configured: `Issue events`, `Power off and restart VMs (conservative)`, or `Power off and restart VMs (aggressive)` | **HIGH** | vSphere |
| 2.3 | PDL (Permanent Device Loss) response is configured: `Issue events` or `Power off and restart VMs` | **HIGH** | vSphere |
| 2.4 | APD timeout is set appropriately (default 140 seconds; adjust based on storage failover time) | **MEDIUM** | vSphere |
| 2.5 | APD response delay is configured to allow storage path recovery before VM restart | **MEDIUM** | vSphere |
| 2.6 | VMCP behavior is tested with planned storage failover to validate response time | **MEDIUM** | vSphere |

### Common Mistakes

**Mistake**: VMCP disabled, leaving VMs vulnerable to prolonged storage failures.

```
# BAD - VMCP disabled, VMs accumulate I/O errors silently
VM Component Protection: Disabled

# GOOD - VMCP enabled with conservative APD response
VM Component Protection:
    PDL Failure Condition: Power off and restart VMs
    APD Failure Condition: Power off and restart VMs (conservative)
    APD Delay: 180 seconds    # Allow time for storage path recovery
    APD Reaction: Reset (moderate)
```

**Mistake**: APD timeout shorter than the storage failover time.

Why it is a problem: If the storage array takes 120 seconds to complete a failover but the APD timeout is 60 seconds, VMCP triggers VM restart before storage is available. The restarted VMs then fail to power on because the storage is still transitioning. Coordinate APD timeout with the storage team to match or exceed the worst-case storage failover time.

---

## 3. vSphere DRS Configuration

**Category**: Resource Optimization

DRS balances VM workloads across hosts in a cluster by recommending or automatically executing vMotion migrations. Misconfigured DRS causes excessive migrations (VM churn), unbalanced clusters, or ignored placement constraints. Resource pools, affinity rules, and automation levels must align with operational requirements.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 3.1 | DRS is enabled on all production clusters | **MEDIUM** | vSphere |
| 3.2 | DRS automation level is set appropriately (`Fully Automated` for most clusters, `Partially Automated` for sensitive workloads) | **MEDIUM** | vSphere |
| 3.3 | DRS migration threshold is tuned (default 3 is moderate; lower = less migration, higher = more aggressive) | **LOW** | vSphere |
| 3.4 | Anti-affinity rules are configured for redundant infrastructure VMs (AD DCs, DNS, load balancers, application tiers) | **HIGH** | vSphere |
| 3.5 | Anti-affinity rules are set to `Must` (hard rule) for critical separation requirements, not `Should` (soft rule) | **HIGH** | vSphere |
| 3.6 | Affinity rules are configured only with documented justification (licensing, performance, latency requirements) | **LOW** | vSphere |
| 3.7 | VM-host rules are configured for workloads with placement requirements (licensing, compliance, GPU affinity) | **MEDIUM** | vSphere |
| 3.8 | Resource pools have explicit reservations and limits, not just default shares | **MEDIUM** | vSphere |
| 3.9 | Resource pool hierarchy is flat (avoid deeply nested pools that complicate capacity planning) | **LOW** | vSphere |
| 3.10 | Resource pools are not used solely for folder-like organization (use VM folders instead) | **LOW** | vSphere |
| 3.11 | DRS rules do not conflict with each other (conflicting rules cause unpredictable behavior) | **HIGH** | vSphere |
| 3.12 | Predictive DRS is evaluated for workloads with predictable patterns (requires vRealize Operations) | **INFO** | vSphere |

### What to Look For

```powershell
# PowerCLI - Check DRS configuration
Get-Cluster | Select-Object Name, DrsEnabled, DrsMode, DrsAutomationLevel

# Check anti-affinity rules
Get-DrsRule -Cluster "Production" | Where-Object { $_.Type -eq "VMAntiAffinity" } |
    Select-Object Name, Enabled, Mandatory, VMIds

# Check VM-host rules
Get-DrsVMHostRule -Cluster "Production" |
    Select-Object Name, Enabled, Type, VMGroup, VMHostGroup

# Check resource pools
Get-ResourcePool -Location (Get-Cluster "Production") |
    Select-Object Name, CpuSharesLevel, CpuReservationMHz, CpuLimitMHz,
    MemSharesLevel, MemReservationMB, MemLimitMB
```

### Common Mistakes

**Mistake**: Anti-affinity rules set to `Should` (soft) for critical VM pairs like domain controllers.

Why it is a problem: A `Should` (soft) anti-affinity rule is a preference, not a requirement. DRS may place both domain controllers on the same host during initial placement, maintenance mode migrations, or when the cluster is resource-constrained. If that host fails, both DCs are lost simultaneously. Use `Must` (hard) rules for VMs that must never share a host.

```
# BAD - soft anti-affinity, DRS can override
DRS Rule "Separate-DCs": Type=VMAntiAffinity, Mandatory=False (Should)
    VMs: DC-01, DC-02

# GOOD - hard anti-affinity, DRS enforced
DRS Rule "Separate-DCs": Type=VMAntiAffinity, Mandatory=True (Must)
    VMs: DC-01, DC-02
```

**Mistake**: Resource pools with no reservations used as organizational containers.

Why it is a problem: Resource pools without reservations do not guarantee resources. VMs in resource pools with only `Shares` compete with each other proportionally when the cluster is contended, but shares are relative within a pool. A deeply nested pool hierarchy with default shares creates non-intuitive resource distribution. Use VM folders for organization and resource pools only when explicit resource guarantees are needed.

**Mistake**: Conflicting DRS rules that DRS cannot satisfy simultaneously.

```
# CONFLICT EXAMPLE
Rule 1 (Must): VM-A must run on Host-Group-1
Rule 2 (Must): VM-A and VM-B must be kept together (affinity)
Rule 3 (Must): VM-B must run on Host-Group-2
# DRS cannot satisfy all three rules -- VM-A and VM-B cannot be together
# if they are forced to different host groups.
```

---

## 4. Fault Tolerance

**Category**: Continuous Availability

vSphere Fault Tolerance provides zero-downtime protection by maintaining a live shadow VM on a secondary host using lockstep execution. FT has strict requirements and significant resource overhead. It is appropriate only for VMs that cannot tolerate any downtime, not as a general-purpose HA mechanism.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 4.1 | FT is used only for VMs with documented zero-downtime requirements (not as a substitute for HA) | **MEDIUM** | vSphere |
| 4.2 | FT logging network uses a dedicated VMkernel adapter with sufficient bandwidth (10 GbE minimum) | **HIGH** | vSphere |
| 4.3 | FT-protected VMs meet platform limits (check maximum vCPU and memory per vSphere version) | **MEDIUM** | vSphere |
| 4.4 | FT secondary VM placement is on a host in a different failure domain where possible | **MEDIUM** | vSphere |
| 4.5 | FT performance impact is understood and accepted by the VM owner (2x resource consumption) | **LOW** | vSphere |
| 4.6 | FT is not combined with snapshots, Storage DRS, or vSAN stretched clusters (unsupported combinations) | **HIGH** | vSphere |
| 4.7 | FT logging network latency is monitored; high latency degrades VM performance | **MEDIUM** | vSphere |

### Common Mistakes

**Mistake**: Enabling FT on VMs that do not require zero-downtime protection.

Why it is a problem: FT doubles CPU and memory consumption (secondary VM runs in lockstep), requires dedicated 10 GbE bandwidth, and restricts VM features (no snapshots, limited vCPU/memory). HA restart (typically 1-3 minutes of downtime) is sufficient for the vast majority of workloads and has no ongoing performance overhead.

**Mistake**: FT logging on a shared or low-bandwidth network.

Why it is a problem: FT logging transmits every CPU instruction and memory page to the secondary host in real time. On a congested or low-bandwidth network, the primary VM throttles execution to match the logging throughput, causing severe application performance degradation. Use a dedicated 10 GbE or faster network for FT logging.

---

## 5. Replication and Disaster Recovery

**Category**: Disaster Recovery

Replication and DR planning ensure workload recovery when an entire site or cluster is lost. RPO (Recovery Point Objective) defines acceptable data loss; RTO (Recovery Time Objective) defines acceptable downtime. Without tested DR procedures, replication is just disk usage -- it does not constitute a recovery capability.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 5.1 | RPO and RTO are defined per application/service tier with business stakeholder sign-off | **HIGH** | All |
| 5.2 | Replication technology matches the RPO requirement (synchronous for RPO=0, asynchronous for RPO>0) | **HIGH** | All |
| 5.3 | vSphere Replication or SRM is configured for VMs requiring cross-site DR | **MEDIUM** | vSphere |
| 5.4 | Recovery plans are documented with explicit runbook steps, not just "failover to DR site" | **HIGH** | All |
| 5.5 | Recovery plans include network reconfiguration (IP changes, DNS updates, firewall rules) | **HIGH** | All |
| 5.6 | Recovery plans include dependency ordering (storage, directory services, DNS, database, application, load balancer) | **HIGH** | All |
| 5.7 | DR failover is tested at minimum quarterly with documented results | **HIGH** | All |
| 5.8 | DR failback procedures are documented and tested | **MEDIUM** | All |
| 5.9 | RPO compliance is monitored: replication lag is tracked with alerting on SLA breach | **HIGH** | All |
| 5.10 | DR site capacity matches production requirements (not undersized to save cost without documented acceptance) | **MEDIUM** | All |
| 5.11 | Replication encryption is enabled for cross-site traffic traversing untrusted networks | **HIGH** | All |
| 5.12 | Backup and replication are complementary (replication does not replace backup; both are needed) | **HIGH** | All |
| 5.13 | Proxmox replication (ZFS-based) is configured with appropriate schedule for local HA | **MEDIUM** | Proxmox |
| 5.14 | KVM environments use libvirt-based or storage-level replication for DR | **MEDIUM** | KVM |

### Common Mistakes

**Mistake**: Replication configured but DR failover never tested.

Why it is a problem: Untested DR is not DR -- it is an assumption. Common failure modes discovered only during testing include: replication lag exceeding RPO, missing network configuration at the DR site, application dependencies not included in the recovery plan, DNS propagation delays, license servers unavailable at DR, and incompatible storage at the DR site. Test quarterly at minimum.

**Mistake**: Using replication as a substitute for backup.

Why it is a problem: Replication faithfully copies corruption, ransomware encryption, accidental deletion, and logical errors to the DR site. If an administrator deletes a database, the deletion replicates. Backups provide point-in-time recovery independent of the live state. Both replication and backup are required.

**Mistake**: RPO and RTO defined by IT without business stakeholder input.

Why it is a problem: IT may set an RPO of 1 hour for a financial system that the business requires RPO of 0 (synchronous replication). Alternatively, IT may invest in expensive synchronous replication for a system where the business would accept 4 hours of data loss. RPO/RTO must be agreed with the business and documented.

### Example Correct Configuration

**vSphere Replication RPO monitoring with vRealize/Aria Operations**:

```
# SRM Recovery Plan structure
Recovery Plan: "DR-Failover-Production"
    Priority Group 1 (Highest): Infrastructure
        - AD Domain Controllers
        - DNS Servers
        - Certificate Authority
    Priority Group 2: Database Tier
        - SQL Server Primary
        - PostgreSQL Primary
    Priority Group 3: Application Tier
        - App Server 01-04
        - API Gateway
    Priority Group 4: Web Tier
        - Web Server 01-04
        - Load Balancer VIP reconfiguration
    Post-Recovery Steps:
        - Update DNS A records for external services
        - Verify certificate validity at DR site
        - Run application health checks
        - Notify monitoring system of site switch
        - Update firewall rules for DR site source IPs
```

**Proxmox ZFS replication for local HA**:

```bash
# Configure ZFS replication from node pve1 to pve2, every 15 minutes
# Proxmox GUI: Datacenter -> Replication -> Add
# Or via CLI:
pvesr create-local-job 100-0 pve2 --schedule "*/15" --rate 50
# VM ID 100, job 0, replicate to pve2, every 15 min, rate limit 50 MB/s

# Verify replication status
pvesr status
pvesr list
```

---

## 6. Proxmox HA Configuration

**Category**: High Availability

Proxmox VE provides HA through the `ha-manager` service, which monitors VMs and containers and restarts them on surviving nodes when a host fails. The HA stack uses corosync for cluster communication and fencing (via watchdog or IPMI) to prevent split-brain. Proper configuration of fencing, HA groups, and resource priorities is essential.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 6.1 | Proxmox HA is enabled and `ha-manager` service is running on all cluster nodes | **HIGH** | Proxmox |
| 6.2 | Fencing is configured and tested (hardware watchdog or IPMI-based fencing) | **CRITICAL** | Proxmox |
| 6.3 | Fencing watchdog device is `/dev/watchdog` and the appropriate kernel module is loaded | **HIGH** | Proxmox |
| 6.4 | HA groups are configured to control VM placement preferences across nodes | **MEDIUM** | Proxmox |
| 6.5 | HA group `restricted` flag is set appropriately (when true, VMs run only on group members) | **MEDIUM** | Proxmox |
| 6.6 | HA group `nofailback` flag is considered (prevents automatic migration back to preferred node after recovery) | **LOW** | Proxmox |
| 6.7 | VM/container HA state is set to `started` for resources that must be automatically restarted | **HIGH** | Proxmox |
| 6.8 | HA resource request-state recovery policy is configured (`relocate` for host failure) | **MEDIUM** | Proxmox |
| 6.9 | Shared storage is available on all HA nodes for HA-managed VMs (local storage VMs cannot be migrated) | **CRITICAL** | Proxmox |
| 6.10 | At least 3 nodes exist in the cluster to maintain quorum after a single node failure | **HIGH** | Proxmox |
| 6.11 | Quorum device (QDevice) is configured for 2-node clusters (if 2-node cluster is unavoidable) | **HIGH** | Proxmox |

### Example Correct Configuration

**Proxmox HA with watchdog fencing**:

```bash
# Verify watchdog device
ls -l /dev/watchdog
# Should exist and be accessible

# Check loaded watchdog module
lsmod | grep -i watchdog
# Typically: softdog, iTCO_wdt, or ipmi_watchdog

# Load hardware watchdog module if available
modprobe ipmi_watchdog

# /etc/default/pve-ha-manager
# Watchdog module should be configured here or loaded at boot

# Configure HA group
ha-manager groupadd prod-group --nodes pve1,pve2,pve3 --restricted 1

# Add VM to HA management
ha-manager add vm:100 --group prod-group --state started --max-relocate 3 --max-restart 3

# Verify HA status
ha-manager status
```

**Proxmox 2-node cluster with QDevice**:

```bash
# On an external QDevice host (lightweight Linux VM, NOT a cluster member)
apt install corosync-qdevice-net

# On each Proxmox node
pvecm qdevice setup <qdevice-ip>
pvecm status
# Should show 3 votes total: 1 per node + 1 from QDevice
```

### Common Mistakes

**Mistake**: Proxmox HA enabled without fencing configured.

Why it is a problem: Without fencing, a network partition causes split-brain. Both sides of the partition believe they own the HA-managed VMs and attempt to run them. Two instances of the same VM accessing shared storage simultaneously causes data corruption. Fencing (watchdog or IPMI) ensures that a node that loses quorum is forcibly rebooted before its VMs are started elsewhere.

**Mistake**: HA-managed VMs stored on local storage.

Why it is a problem: When a node fails, Proxmox HA attempts to start the VM on a surviving node. If the VM disk is on the failed node's local storage (not shared storage like Ceph, NFS, or iSCSI), the surviving node cannot access the disk and the VM cannot start. All HA-managed VMs must be on shared storage accessible from every node in the HA group.

---

## 7. KVM/libvirt High Availability

**Category**: High Availability

KVM/libvirt environments typically use external HA frameworks (Pacemaker/Corosync, oVirt, or Kubernetes with KubeVirt) for VM high availability. Unlike vSphere and Proxmox, libvirt itself does not provide built-in HA. The HA implementation must handle fencing, resource management, and VM restart with the same rigor as commercial alternatives.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 7.1 | An HA framework is deployed (Pacemaker/Corosync, oVirt, or equivalent) for production KVM VMs | **HIGH** | KVM |
| 7.2 | STONITH/fencing is configured and tested in the HA cluster | **CRITICAL** | KVM |
| 7.3 | Fencing uses reliable mechanisms (IPMI, iLO, iDRAC, PDU) not just software watchdog for production | **HIGH** | KVM |
| 7.4 | Shared storage is available for all HA-managed VM disk images (Ceph RBD, NFS, iSCSI, GFS2) | **CRITICAL** | KVM |
| 7.5 | Pacemaker resource agents for libvirt VMs are configured with appropriate timeouts and monitor intervals | **MEDIUM** | KVM |
| 7.6 | Quorum policy is configured to prevent split-brain (quorum device for even-node clusters) | **HIGH** | KVM |
| 7.7 | Live migration is configured and tested between cluster nodes | **MEDIUM** | KVM |
| 7.8 | libvirt hooks or systemd units are not conflicting with HA framework control of VMs | **MEDIUM** | KVM |

### Example Correct Configuration

**Pacemaker/Corosync HA for KVM VM**:

```bash
# Configure fencing via IPMI
pcs stonith create fence-node1 fence_ipmilan \
    ipaddr=10.0.99.1 login=admin passwd=<password> \
    pcmk_host_list="kvm-node1" \
    lanplus=1 power_timeout=30

pcs stonith create fence-node2 fence_ipmilan \
    ipaddr=10.0.99.2 login=admin passwd=<password> \
    pcmk_host_list="kvm-node2" \
    lanplus=1 power_timeout=30

# Configure VM as Pacemaker resource
pcs resource create vm-critical-app VirtualDomain \
    hypervisor="qemu:///system" \
    config="/etc/libvirt/qemu/critical-app.xml" \
    migration_transport=tls \
    meta allow-migrate=true \
    op start timeout=120s \
    op stop timeout=120s \
    op monitor interval=30s timeout=30s

# Set VM ordering and colocation
pcs constraint order start shared-storage then start vm-critical-app
pcs constraint colocation add vm-critical-app with shared-storage INFINITY
```

---

## 8. Cluster Design and Sizing

**Category**: Capacity Planning

Cluster design determines the blast radius of failures and the capacity available for VM restart. Undersized clusters cannot survive host failures. Heterogeneous clusters complicate DRS balancing and EVC compatibility. Network and storage design must support the cluster's HA and migration requirements.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 8.1 | Clusters are sized for N+1 host failures minimum (N+2 for mission-critical environments) | **HIGH** | All |
| 8.2 | Failover capacity is validated: total VM resource demand fits on N-1 (or N-2) hosts | **HIGH** | All |
| 8.3 | Host hardware is homogeneous within a cluster (same CPU family, memory, NIC configuration) | **MEDIUM** | All |
| 8.4 | EVC (Enhanced vMotion Compatibility) mode is set to the baseline CPU of the cluster | **MEDIUM** | vSphere |
| 8.5 | EVC mode is set before adding VMs to avoid migration compatibility issues | **LOW** | vSphere |
| 8.6 | Cluster size does not exceed platform-supported maximums (vSphere: 96 hosts, Proxmox: 32 nodes) | **MEDIUM** | All |
| 8.7 | Cluster boundaries align with failure domains (do not span clusters across unreliable WAN links) | **HIGH** | All |
| 8.8 | Network bandwidth supports concurrent vMotion during host failure scenarios | **MEDIUM** | All |
| 8.9 | Storage bandwidth supports concurrent VM restart I/O storm during host failure | **MEDIUM** | All |
| 8.10 | Management cluster is separate from workload clusters (vCenter, monitoring, backup infrastructure) | **HIGH** | vSphere |
| 8.11 | Stretched clusters (metro clusters) have symmetric storage and network capacity at both sites | **HIGH** | All |
| 8.12 | Stretched clusters have witness/tiebreaker at a third site or use asymmetric quorum | **CRITICAL** | All |

### What to Look For

```powershell
# PowerCLI - Validate N+1 capacity
$cluster = Get-Cluster "Production"
$hosts = Get-VMHost -Location $cluster
$totalCPU = ($hosts | Measure-Object -Property CpuTotalMhz -Sum).Sum
$totalMem = ($hosts | Measure-Object -Property MemoryTotalGB -Sum).Sum
$usedCPU = ($hosts | Measure-Object -Property CpuUsageMhz -Sum).Sum
$usedMem = ($hosts | Measure-Object -Property MemoryUsageGB -Sum).Sum
$hostCount = $hosts.Count
$perHostCPU = $totalCPU / $hostCount
$perHostMem = $totalMem / $hostCount

Write-Host "N+1 capacity after losing largest host:"
Write-Host "  Available CPU: $($totalCPU - $perHostCPU) MHz, Used: $usedCPU MHz"
Write-Host "  Available Mem: $([math]::Round($totalMem - $perHostMem, 1)) GB, Used: $([math]::Round($usedMem, 1)) GB"

if ($usedCPU -gt ($totalCPU - $perHostCPU)) {
    Write-Warning "INSUFFICIENT CPU for N+1 failover!"
}
if ($usedMem -gt ($totalMem - $perHostMem)) {
    Write-Warning "INSUFFICIENT MEMORY for N+1 failover!"
}
```

### Common Mistakes

**Mistake**: Cluster at 90% capacity with N+1 admission control enabled but insufficient capacity for failover.

Why it is a problem: Admission control reserves capacity but only based on the configured policy. If the cluster has 4 hosts and admission control reserves 25% but actual VM load consumes 85% of total capacity, a host failure leaves only 75% of original capacity for 85% of original load. VMs will not restart. Monitor the gap between reserved capacity and actual utilization.

**Mistake**: Mixed CPU generations in a cluster without EVC.

Why it is a problem: VMs started on newer CPU hosts use instruction sets (AVX-512, newer SSE extensions) not available on older hosts. When DRS attempts to vMotion these VMs to older hosts (during load balancing or host maintenance), the migration fails with a CPU compatibility error. EVC masks advanced CPU features to the baseline of the cluster, ensuring migration compatibility.

```
# EVC configuration
# Set EVC to the lowest common CPU generation in the cluster
# Example: Cluster has Intel Sapphire Rapids and Ice Lake hosts
# Set EVC to: Intel Ice Lake Generation (the lower of the two)
```

**Mistake**: Stretched cluster without a witness at a third site.

Why it is a problem: In a two-site stretched cluster, a site failure leaves the surviving site with 50% of the cluster nodes. Without a witness/tiebreaker at a third site, the surviving site cannot achieve quorum and the entire cluster goes down. The witness provides the deciding vote to maintain quorum at the surviving site.

---

## 9. Network Design for HA Clusters

**Category**: Network Architecture

HA cluster reliability depends on network design. Heartbeat networks, management networks, and migration networks must be redundant and properly separated. A single network failure should not cause the cluster to declare host failures or prevent VM migration.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 9.1 | HA heartbeat/management network uses redundant paths (NIC teaming, multiple uplinks, dual switches) | **HIGH** | All |
| 9.2 | Multiple isolation addresses are configured to avoid false isolation declarations | **MEDIUM** | vSphere |
| 9.3 | Corosync redundant links use physically separate network paths | **HIGH** | Proxmox |
| 9.4 | vMotion/migration network has sufficient bandwidth for concurrent migrations during failover | **MEDIUM** | All |
| 9.5 | Management network does not share uplinks with high-bandwidth traffic (vMotion, storage, VM traffic) without QoS | **MEDIUM** | All |
| 9.6 | Network redundancy is validated: pulling a single cable or disabling a single switch does not cause host isolation | **HIGH** | All |
| 9.7 | HA additional isolation addresses point to highly available infrastructure (default gateway, multiple switches) | **MEDIUM** | vSphere |

### Common Mistakes

**Mistake**: Single uplink for management/heartbeat network.

Why it is a problem: A single cable, NIC, or switch failure causes the host to lose heartbeat connectivity. HA declares the host as failed and restarts all VMs on other hosts, even though the host and its VMs are perfectly healthy. This causes unnecessary downtime and I/O storms on shared storage.

**Mistake**: vSphere HA isolation address pointing only to the default gateway.

```
# BAD - single isolation address (default: cluster default gateway)
# If the gateway is on the same switch that failed, isolation is falsely declared

# GOOD - multiple isolation addresses on different network paths
das.isolationaddress0 = 10.0.1.1     # Default gateway
das.isolationaddress1 = 10.0.1.2     # Second switch management IP
das.isolationaddress2 = 10.0.2.1     # Different subnet gateway
das.usedefaultisolationaddress = false
```

---

## Review Procedure Summary

When evaluating HA, DRS, and DR configuration:

1. **Assess HA configuration**: Verify HA is enabled with admission control, appropriate isolation response, and configured heartbeat datastores on every production cluster.
2. **Evaluate VMCP**: Confirm VM Component Protection is enabled with APD/PDL responses aligned to storage failover characteristics.
3. **Review DRS settings**: Check automation level, migration threshold, and verify anti-affinity rules for critical VM pairs use hard (`Must`) rules.
4. **Validate resource pools**: Confirm resource pools have explicit reservations, are not overcommitted, and are not used as organizational containers.
5. **Assess Fault Tolerance**: If FT is in use, verify it is justified, the logging network is dedicated and sufficient, and unsupported feature combinations are avoided.
6. **Review DR posture**: Confirm RPO/RTO is defined with business sign-off, replication matches RPO requirements, recovery plans are documented with dependency ordering, and DR is tested quarterly.
7. **Verify cluster sizing**: Validate N+1 (or N+2) capacity exists after losing the largest host, hosts are homogeneous, EVC is configured, and cluster boundaries align with failure domains.
8. **Check Proxmox HA**: Verify fencing is configured and tested, VMs are on shared storage, quorum is achievable after single-node failure, and HA groups are properly configured.
9. **Check KVM HA**: Verify STONITH fencing, shared storage for VM disks, HA framework resource agents, and quorum policy.
10. **Validate network redundancy**: Confirm heartbeat, management, and migration networks have redundant paths and that no single failure causes host isolation.

---

## Quick Reference: Severity Summary

| Severity | HA/DRS/DR Findings |
|----------|-------------------|
| CRITICAL | No fencing configured in Proxmox or KVM HA cluster; HA-managed VMs on local (non-shared) storage; stretched cluster without witness/tiebreaker at third site; STONITH disabled in Pacemaker cluster |
| HIGH | HA disabled on production cluster; admission control disabled; host isolation response set to `Power Off`; no heartbeat datastores configured; VMCP disabled; anti-affinity rules using soft (`Should`) instead of hard (`Must`) for critical VMs; DRS rules conflict; RPO/RTO undefined; DR never tested; replication lag not monitored; recovery plans undocumented; cluster undersized for N+1; management cluster not separated; fencing not tested; no quorum device for even-node cluster; single uplink for heartbeat network; replication unencrypted over WAN |
| MEDIUM | VM restart priorities not configured; heartbeat datastore on single storage array; APD timeout mismatched with storage failover; DRS automation level inappropriate; resource pools without reservations; FT logging on shared network; DR failback not documented; DR site undersized; host hardware heterogeneous without EVC; insufficient migration bandwidth; management sharing uplinks without QoS; Proxmox HA group not optimally configured; Proxmox replication schedule too infrequent |
| LOW | No orchestrated restart dependencies; DRS migration threshold at default; affinity rules without documentation; resource pool hierarchy too deep; FT performance impact not acknowledged; EVC not set before VM deployment; Proxmox `nofailback` not evaluated |
| INFO | Well-documented DR runbook with tested procedures; anti-affinity rules covering all infrastructure VM pairs; N+2 cluster sizing with validated capacity model; DRS predictive enabled with vRealize Operations integration; recovery plan with automated health checks |

---

## References

- VMware vSphere 8 Availability Guide (HA, DRS, FT)
- VMware vSphere HA Deepdive (Duncan Epping)
- VMware Knowledge Base: VMCP (APD/PDL) configuration
- VMware DRS Performance Best Practices
- VMware Site Recovery Manager Administration Guide
- Proxmox VE Administration Guide: High Availability
- Proxmox VE Wiki: Fencing, QDevice, HA Manager
- Red Hat High Availability Add-On Administration Guide (Pacemaker/Corosync)
- CIS VMware ESXi 8.0 Benchmark (HA/DRS sections)
- NIST SP 800-34: Contingency Planning Guide for Federal Information Systems
- DISA STIG for VMware vSphere (availability controls)
