# Server Configuration

Reference checklist for the `compute-reviewer` agent when evaluating physical and virtual server configuration. This file complements the inline criteria in `review-compute.md` with detailed guidance on BIOS/UEFI settings, BMC/IPMI management configuration, OS-level tuning, and physical security controls. Includes vendor-specific examples for Dell iDRAC, HPE iLO, and Supermicro IPMI.

---

## 1. BIOS/UEFI Configuration

**Category**: BIOS/UEFI Settings

BIOS/UEFI settings are the foundation of server security and performance. Misconfigured BIOS settings can disable hardware security features, degrade performance, or introduce boot-time attack vectors. These settings are often overlooked because they require out-of-band access and are invisible to OS-level tooling.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| UEFI Secure Boot is enabled | HIGH | Without Secure Boot, unsigned or tampered bootloaders and kernels can execute, enabling bootkits and rootkits |
| TPM 2.0 is enabled and activated | HIGH | TPM is required for measured boot, disk encryption key sealing (BitLocker, LUKS with Clevis), and remote attestation |
| TPM ownership is established and not using default owner password | MEDIUM | An unowned or default-password TPM can be claimed by an attacker for key storage |
| Boot order is restricted to intended devices (no USB, no PXE unless required) | MEDIUM | Unrestricted boot order allows booting from rogue USB devices or unauthorized PXE servers |
| Boot order is password-protected against modification | MEDIUM | Without a BIOS password, physical access enables boot order changes |
| UEFI firmware password (admin/setup password) is set | MEDIUM | Prevents unauthorized BIOS setting changes via local console or BMC |
| Legacy/CSM boot is disabled (UEFI-only mode) | MEDIUM | CSM (Compatibility Support Module) disables Secure Boot protections and enables legacy boot attacks |
| Virtualization extensions are enabled (VT-x/AMD-V) | HIGH | Required for hypervisors, containers, and modern isolation features (KVM, Hyper-V, VMware ESXi) |
| VT-d / AMD-Vi (IOMMU) is enabled | MEDIUM | Required for PCI passthrough, SR-IOV, and DMA protection; prevents DMA attacks from PCIe devices |
| Hardware-assisted virtualization for directed I/O is enabled | MEDIUM | Needed for secure device assignment in virtualized environments |
| Hyperthreading/SMT setting matches workload requirements | MEDIUM | Hyperthreading may need to be disabled for security-sensitive workloads (mitigating MDS/L1TF) or enabled for throughput workloads |
| NUMA configuration matches physical topology and workload | MEDIUM | Incorrect NUMA settings (e.g., node interleaving enabled when NUMA-aware scheduling is needed) degrade memory-intensive workload performance |
| Memory operating mode is set appropriately (Independent, Mirroring, or Sparing) | MEDIUM | Independent mode maximizes capacity; Mirroring provides fault tolerance for critical workloads; wrong mode wastes capacity or risks data |
| Performance profile matches workload (Performance, Balanced, Power Saving, Custom) | MEDIUM | A power-saving profile on a latency-sensitive server causes inconsistent response times |
| C-states and P-states are configured appropriately for workload type | LOW | Deep C-states save power but add wake-up latency; inappropriate for real-time or latency-sensitive workloads |
| Memory frequency is set to maximum supported speed | LOW | Auto-detection may downclock memory if DIMM populations are mixed or suboptimal |
| Memory error correction (ECC) is enabled | HIGH | Non-ECC memory in production servers risks silent data corruption |
| Intel TXT or AMD SEV is enabled where attestation or encrypted VMs are required | MEDIUM | Required for trusted computing and confidential computing workloads |
| SR-IOV is enabled if virtual function passthrough is used | LOW | Must be enabled at BIOS level before OS/hypervisor can use SR-IOV NICs |

### Common Mistakes

**Mistake**: Secure Boot disabled because "it caused issues during initial OS install."

Why it is a problem: Secure Boot may require enrolling custom keys (MOK) for third-party kernel modules (DKMS drivers, NVIDIA), but disabling it permanently removes boot-time integrity verification. Modern Linux distributions fully support Secure Boot with MOK enrollment.

**Mistake**: NUMA node interleaving enabled on a database server.

Why it is a problem: Node interleaving distributes memory accesses evenly across all NUMA nodes, which sounds good but defeats NUMA-aware memory allocation. Database engines (PostgreSQL, MySQL, Oracle) and JVMs are NUMA-aware and perform better with interleaving disabled, allowing the OS scheduler to place processes near their memory.

```
# BAD - BIOS setting
NUMA Node Interleaving: Enabled
# Results in: all memory access has uniform latency (but higher average)

# GOOD - BIOS setting
NUMA Node Interleaving: Disabled
# Results in: local NUMA node access is fast; OS/application manages placement
```

**Mistake**: Performance profile set to "Power Saving" or "OS Controlled" on latency-sensitive workloads.

```
# BAD - BIOS System Profile
System Profile: Energy Efficient
# CPU frequency scales down during idle, causing latency spikes on wake

# GOOD - BIOS System Profile for latency-sensitive workloads
System Profile: Performance
# or
System Profile: Custom
  CPU Power Management: Maximum Performance
  C-States: Disabled
  Turbo Boost: Enabled
  Energy Efficient Policy: Performance
```

**Mistake**: VT-d/IOMMU disabled, preventing SR-IOV and device passthrough.

Why it is a problem: Without IOMMU, PCIe devices assigned to VMs can perform unrestricted DMA to host memory. This is both a security vulnerability (DMA attacks) and a functional limitation (no SR-IOV, no VFIO passthrough).

### Vendor-Specific BIOS Access

**Dell PowerEdge**: Press F2 during POST or configure via iDRAC -> BIOS Settings. Export/import BIOS profiles via `racadm` or Redfish.

```bash
# Dell: Export BIOS configuration via racadm
racadm get BIOS

# Dell: Set Secure Boot via racadm
racadm set BIOS.SecureBoot.SecureBootMode Enabled

# Dell: Set System Profile via racadm
racadm set BIOS.SysProfileSettings.SysProfile PerfOptimized
```

**HPE ProLiant**: Press F9 during POST or configure via iLO -> BIOS/Platform Configuration. Use the RESTful Interface Tool (ilorest) for automation.

```bash
# HPE: Get BIOS settings via ilorest
ilorest get --select Bios.

# HPE: Set Secure Boot via ilorest
ilorest set SecureBootStatus=Enabled --select Bios. --commit

# HPE: Set workload profile
ilorest set WorkloadProfile=Virtualization-MaxPerformance --select Bios. --commit
```

**Supermicro**: Press DEL during POST or configure via IPMI web interface -> BIOS Settings. Use SUM (Supermicro Update Manager) for automation.

```bash
# Supermicro: Get BIOS settings via SUM
sum -c GetBiosCfg --file bios_current.xml

# Supermicro: Apply BIOS configuration
sum -c ChangeBiosCfg --file bios_desired.xml
```

---

## 2. BMC/IPMI/iDRAC/iLO Configuration

**Category**: BMC Management Security

The Baseboard Management Controller (BMC) provides out-of-band management capabilities including remote console, power control, hardware monitoring, and firmware updates. A compromised BMC gives an attacker persistent, OS-independent control of the server -- it is effectively a hardware backdoor if misconfigured.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| BMC is on a dedicated, isolated management network (not the production data network) | CRITICAL | BMC on the data network exposes the most privileged management interface to application-level attackers |
| BMC is not accessible from the internet | CRITICAL | Internet-exposed BMCs are routinely exploited; IPMI protocol has well-known vulnerabilities |
| Default BMC credentials are changed (no `admin/admin`, `ADMIN/ADMIN`, `root/calvin`) | CRITICAL | Default credentials are the first thing attackers try; they are published in vendor documentation |
| BMC firmware is updated to the latest stable version | HIGH | BMC firmware vulnerabilities (e.g., CVE-2019-6260 USBAnywhere, CVE-2023-34329 iLO bypass) enable remote takeover |
| IPMI over LAN (IPMI v2.0 RMCP+) uses strong passwords (14+ characters) | HIGH | IPMI 2.0 cipher zero vulnerability allows authentication bypass; strong passwords mitigate brute force |
| IPMI cipher suite 0 (no authentication) is disabled | CRITICAL | Cipher suite 0 allows unauthenticated IPMI access; must be explicitly disabled |
| SNMPv3 is used instead of SNMPv1/v2c | HIGH | SNMPv1/v2c community strings are sent in plaintext; SNMPv3 provides authentication and encryption |
| SNMP community strings are not default (`public`, `private`) | CRITICAL | Default community strings allow read/write access to device configuration and sensitive data |
| BMC web interface uses HTTPS with a valid certificate (not self-signed in production) | MEDIUM | Self-signed certificates are vulnerable to MITM; internal CA-signed certificates are acceptable |
| BMC web interface has session timeout configured (15 minutes or less) | LOW | Long sessions increase risk of unauthorized access from unattended terminals |
| Unused BMC services are disabled (Telnet, VNC if not needed, shared NIC mode) | MEDIUM | Every enabled service increases the attack surface |
| IPMI serial-over-LAN (SOL) is restricted to authorized users | MEDIUM | SOL provides full console access equivalent to physical serial port |
| System Event Log (SEL) forwarding is configured to a central syslog/SIEM | MEDIUM | BMC event logs are small and overwrite; forwarding ensures hardware events are retained and correlated |
| BMC alerting is configured for hardware events (temperature, fan, PSU, memory errors) | MEDIUM | Without alerting, hardware degradation goes unnoticed until failure |
| BMC user accounts follow least privilege (separate admin and read-only accounts) | MEDIUM | A single shared admin account prevents audit trail and violates least privilege |
| LDAP/AD integration for BMC authentication uses secure bind (LDAPS) | MEDIUM | LDAP clear-text bind exposes domain credentials |
| BMC virtual media is disabled or restricted when not in use | LOW | Virtual media can be used to boot from unauthorized images |
| BMC shared NIC mode is disabled (dedicated management NIC used) | HIGH | Shared NIC mode places BMC traffic on the data network, defeating network isolation |
| IPMI LAN channel privilege level is set appropriately | MEDIUM | Default may grant ADMINISTRATOR to all LAN users |
| BMC SSL/TLS configuration disables weak protocols (SSLv3, TLS 1.0, TLS 1.1) | MEDIUM | Weak TLS versions have known exploits (POODLE, BEAST) |
| Redfish API uses token-based authentication with session management | LOW | Basic auth over Redfish sends credentials with every request |

### Common Mistakes

**Mistake**: BMC accessible on the production data VLAN.

Why it is a problem: If an application-level vulnerability allows network access, the attacker can reach the BMC. BMC protocols (IPMI, Redfish) are designed for trusted management networks and have a history of severe vulnerabilities. A compromised BMC provides persistent access that survives OS reinstallation.

```
# BAD - BMC on production VLAN (no isolation)
Server NIC1: 10.0.1.50/24  (production)
Server BMC:  10.0.1.51/24  (same subnet as production)

# GOOD - BMC on dedicated management VLAN with firewall rules
Server NIC1: 10.0.1.50/24   (production VLAN 100)
Server BMC:  172.16.0.50/24  (management VLAN 999, ACL-restricted)

# Firewall rules for management VLAN
permit tcp 172.16.0.0/24 -> jump-host:443     # HTTPS to BMC web
permit udp 172.16.0.0/24 -> syslog:514        # SEL forwarding
deny   any 10.0.0.0/8    -> 172.16.0.0/24     # No production -> management
```

**Mistake**: Default credentials left on BMC after rack-and-stack.

```
# BAD - Dell iDRAC default
Username: root
Password: calvin

# BAD - HPE iLO default (printed on pull-out tag, often photographed)
Username: Administrator
Password: <tag-printed-password>

# BAD - Supermicro IPMI default
Username: ADMIN
Password: ADMIN

# GOOD - strong, unique, vault-managed credentials
Username: idrac-admin
Password: <32-char random from HashiCorp Vault / CyberArk>
```

**Mistake**: IPMI cipher suite 0 left enabled.

Why it is a problem: Cipher suite 0 allows authentication bypass -- any user can connect without credentials. This is a well-known vulnerability (CVE-2013-4786) that is trivially exploitable with standard tools like `ipmitool`.

```bash
# CHECK - verify cipher suite 0 is disabled
ipmitool -I lanplus -H 172.16.0.50 -U admin -P password \
  raw 0x06 0x54 0x00 0x00 0x00

# Dell iDRAC: Disable cipher suite 0 via racadm
racadm set iDRAC.IPMILan.CipherSuiteDisable 0

# HPE iLO: Disable via ilorest
ilorest set IPMI/CipherSuiteZero=Disabled --select ManagerNetworkService. --commit
```

**Mistake**: SNMPv1/v2c with default community strings on BMC.

```
# BAD - default SNMP community
Community String: public (read)
Community String: private (read-write)

# GOOD - SNMPv3 with authentication and encryption
SNMPv3 User: monitoring-user
Auth Protocol: SHA-256
Auth Key: <strong-key>
Privacy Protocol: AES-256
Privacy Key: <strong-key>
```

### Vendor-Specific BMC Configuration

**Dell iDRAC Configuration via racadm**:

```bash
# Change default password
racadm set iDRAC.Users.2.Password "$(vault read -field=password secret/idrac/server01)"

# Configure dedicated NIC (not shared)
racadm set iDRAC.NIC.Selection Dedicated
racadm set iDRAC.IPv4.Address 172.16.0.50
racadm set iDRAC.IPv4.Netmask 255.255.255.0
racadm set iDRAC.IPv4.Gateway 172.16.0.1

# Configure syslog forwarding
racadm set iDRAC.SysLog.Server1 syslog.mgmt.example.com
racadm set iDRAC.SysLog.Port1 514
racadm set iDRAC.SysLog.SysLogEnable Enabled

# Disable IPMI over LAN if Redfish is sufficient
racadm set iDRAC.IPMILan.Enable Disabled

# Configure SNMP v3
racadm set iDRAC.SNMP.AgentEnable Enabled
racadm set iDRAC.SNMP.TrapFormat SNMPv3

# Disable Telnet
racadm set iDRAC.Telnet.Enable Disabled

# Set session timeout (seconds)
racadm set iDRAC.WebServer.Timeout 900

# Disable TLS 1.0 and 1.1
racadm set iDRAC.WebServer.TLSProtocol "TLS 1.2 and Higher"
```

**HPE iLO Configuration via ilorest**:

```bash
# Change administrator password
ilorest set Password="$(vault read -field=password secret/ilo/server01)" \
  --select AccountService. --filter UserName=Administrator --commit

# Configure dedicated iLO NIC
ilorest set NICEnabled=True --select EthernetInterface. \
  --filter Id=1 --commit

# Configure SNMP alerting
ilorest set AlertDestinations=["syslog.mgmt.example.com"] \
  --select SnmpService. --commit

# Enable SNMPv3
ilorest set SNMPv1Traps=False --select SnmpService. --commit

# Configure syslog remote server
ilorest set RemoteSyslogEnabled=True --select ManagerNetworkService. --commit
ilorest set RemoteSyslogServer="syslog.mgmt.example.com" \
  --select ManagerNetworkService. --commit

# Set minimum TLS version
ilorest set MinimumTLSVersion=TLS1.2 --select SecurityService. --commit

# Set session timeout
ilorest set SessionTimeoutMinutes=15 --select ManagerService. --commit
```

**Supermicro IPMI Configuration via ipmitool**:

```bash
# Change default password (user ID 2 is typically ADMIN)
ipmitool -I lanplus -H 172.16.0.50 -U ADMIN -P ADMIN \
  user set password 2 "$(vault read -field=password secret/ipmi/server01)"

# Set channel privilege level
ipmitool -I lanplus -H 172.16.0.50 -U admin -P password \
  channel setaccess 1 2 privilege=4  # 4=Administrator

# Configure SOL (Serial over LAN)
ipmitool -I lanplus -H 172.16.0.50 -U admin -P password \
  sol set enabled true 1

# Configure PEF (Platform Event Filtering) for alerting
ipmitool -I lanplus -H 172.16.0.50 -U admin -P password \
  pef policy set 1 destination 1

# Disable unused LAN channels
ipmitool -I lanplus -H 172.16.0.50 -U admin -P password \
  channel setaccess 2 2 callin=off
```

---

## 3. OS-Level Tuning

**Category**: OS Performance Tuning

OS-level tuning ensures that the hardware capabilities configured in BIOS are utilized effectively by the operating system. Mismatched OS settings can negate BIOS optimizations, leave performance on the table, or introduce stability issues.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| CPU frequency governor matches workload requirements (`performance` for latency-sensitive, `schedutil` or `ondemand` for mixed) | MEDIUM | Wrong governor causes either wasted power or latency spikes from frequency scaling |
| `tuned` profile is active and matches workload type | MEDIUM | `tuned` coordinates multiple kernel parameters; running without it or with the wrong profile leaves performance unconfigured |
| Hugepages are configured for applications that benefit (databases, VMs, DPDK) | MEDIUM | Standard 4KB pages cause TLB thrashing for large memory workloads; hugepages reduce TLB misses by 100-1000x |
| Transparent Hugepages (THP) is configured appropriately (disabled for databases, enabled or `madvise` for general workloads) | MEDIUM | THP can cause latency spikes from background defragmentation; databases strongly recommend disabling |
| `irqbalance` is running for general workloads or IRQ affinity is manually pinned for latency-critical workloads | MEDIUM | Without IRQ distribution, a single CPU handles all interrupts, becoming a bottleneck |
| Network sysctl parameters are tuned for server workloads (`net.core.somaxconn`, `net.ipv4.tcp_max_syn_backlog`, `net.core.netdev_max_backlog`) | MEDIUM | Default kernel values are conservative; high-throughput servers need larger buffers and backlogs |
| File descriptor limits are raised for server applications (`nofile` in `/etc/security/limits.conf` or systemd) | MEDIUM | Default limit of 1024 is insufficient for servers handling thousands of concurrent connections |
| `vm.swappiness` is set appropriately (low for databases, moderate for general workloads) | MEDIUM | High swappiness causes databases to swap active pages; too low prevents the kernel from reclaiming truly idle pages |
| NUMA-aware memory allocation is configured (`numactl`, `vm.zone_reclaim_mode`) | MEDIUM | Cross-NUMA memory access is 1.5-3x slower; NUMA-unaware allocation degrades performance unpredictably |
| I/O scheduler matches storage type (`none`/`noop` for NVMe/SSD, `mq-deadline` for spinning disks) | LOW | Wrong scheduler adds unnecessary overhead (elevator sorting for SSDs) or fails to optimize (no merging for HDD) |
| Kernel parameters for security hardening are set (`kernel.randomize_va_space=2`, `kernel.kptr_restrict=1`, `net.ipv4.conf.all.rp_filter=1`) | MEDIUM | CIS Benchmark Level 1 hardening parameters; missing hardening increases attack surface |
| `kernel.panic` and `kernel.panic_on_oops` are configured for automatic recovery | LOW | Without panic-on-oops, a kernel oops can leave the system in an unstable state without rebooting |
| `vm.dirty_ratio` and `vm.dirty_background_ratio` are tuned for workload I/O patterns | LOW | Default values may cause write stalls on high-I/O workloads or excessive memory usage for dirty pages |
| CPU microcode is loaded at boot (latest available version) | HIGH | Microcode updates patch CPU vulnerabilities (Spectre, Meltdown, MDS); without them, hardware mitigations are incomplete |
| Kernel is configured with appropriate speculative execution mitigations | HIGH | Mitigations must be enabled for multi-tenant or security-sensitive workloads; may be selectively disabled for dedicated single-tenant systems after risk assessment |
| `systemd` resource controls (CPUQuota, MemoryMax, TasksMax) are set for services | MEDIUM | Prevents a single service from consuming all system resources |

### Common Mistakes

**Mistake**: Running a database server with Transparent Hugepages enabled.

Why it is a problem: THP causes the kernel to periodically compact memory to create hugepages, introducing unpredictable latency spikes of 10-100ms. Every major database vendor (Oracle, PostgreSQL, MongoDB, Redis) recommends disabling THP.

```bash
# BAD - THP enabled (default on most distributions)
$ cat /sys/kernel/mm/transparent_hugepage/enabled
[always] madvise never

# GOOD - THP disabled for database workloads
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always madvise [never]

# Persistent configuration via tuned profile or kernel parameter
# /etc/tuned/database/tuned.conf
[vm]
transparent_hugepages=never

# Or via kernel boot parameter
GRUB_CMDLINE_LINUX="transparent_hugepage=never"
```

**Mistake**: Using `performance` CPU governor on servers that are mostly idle.

Why it is a problem: The `performance` governor locks all CPUs at maximum frequency, consuming maximum power even when idle. For servers that are not latency-sensitive, `schedutil` (kernel-driven, since Linux 4.7) provides near-`performance` latency with significant power savings.

```bash
# Check current governor
cat /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor

# Set governor via tuned profile
# /etc/tuned/latency-sensitive/tuned.conf
[cpu]
governor=performance
energy_perf_bias=performance
min_perf_pct=100

# For general workloads
# /etc/tuned/balanced-server/tuned.conf
[cpu]
governor=schedutil
energy_perf_bias=balance-performance
```

**Mistake**: Default sysctl values on a high-connection server.

```bash
# BAD - kernel defaults (too low for busy servers)
net.core.somaxconn = 4096
net.ipv4.tcp_max_syn_backlog = 1024
net.core.netdev_max_backlog = 1000
vm.max_map_count = 65530
fs.file-max = 65536

# GOOD - tuned for high-connection server workloads
# /etc/sysctl.d/99-server-tuning.conf
net.core.somaxconn = 65535
net.ipv4.tcp_max_syn_backlog = 65535
net.core.netdev_max_backlog = 65535
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_fin_timeout = 15
net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_intvl = 30
net.ipv4.tcp_keepalive_probes = 5
vm.max_map_count = 262144
fs.file-max = 2097152
fs.nr_open = 2097152
```

**Mistake**: Static hugepages allocated but not used, wasting memory; or hugepages needed but not allocated.

```bash
# Check current hugepage allocation
grep -i huge /proc/meminfo
#   HugePages_Total:     0
#   HugePages_Free:      0
#   Hugepagesize:      2048 kB

# Allocate hugepages for a database server (e.g., 16GB worth of 2MB pages)
# /etc/sysctl.d/99-hugepages.conf
vm.nr_hugepages = 8192     # 8192 * 2MB = 16GB
vm.hugetlb_shm_group = 1001  # GID of the database group

# For DPDK or virtualization workloads needing 1GB hugepages
# Kernel boot parameter:
GRUB_CMDLINE_LINUX="default_hugepagesz=1G hugepagesz=1G hugepages=32"
```

### Fix Patterns

**Comprehensive tuned profile for database workloads**:

```ini
# /etc/tuned/database-optimized/tuned.conf
[main]
summary=Optimized for database server workloads
include=throughput-performance

[cpu]
governor=performance
energy_perf_bias=performance
min_perf_pct=100
force_latency=1

[vm]
transparent_hugepages=never

[sysctl]
vm.swappiness=10
vm.dirty_ratio=15
vm.dirty_background_ratio=5
vm.zone_reclaim_mode=0
kernel.sched_migration_cost_ns=5000000
kernel.sched_autogroup_enabled=0
net.core.somaxconn=65535
net.ipv4.tcp_max_syn_backlog=65535
fs.file-max=2097152

[disk]
readahead=>4096
```

```bash
# Activate the profile
tuned-adm profile database-optimized

# Verify active profile
tuned-adm active
# Current active profile: database-optimized

# Verify settings applied
tuned-adm verify
```

**NUMA-aware application launching**:

```bash
# Check NUMA topology
numactl --hardware

# Run a database process pinned to NUMA node 0
numactl --cpunodebind=0 --membind=0 /usr/bin/mysqld

# Alternatively, use systemd unit configuration
# /etc/systemd/system/mysqld.service.d/numa.conf
[Service]
ExecStart=
ExecStart=/usr/bin/numactl --interleave=all /usr/sbin/mysqld
# Note: --interleave=all is preferred for MySQL; PostgreSQL prefers --membind
```

**Systemd resource controls for services**:

```ini
# /etc/systemd/system/myapp.service.d/resources.conf
[Service]
CPUQuota=200%
MemoryMax=4G
MemoryHigh=3G
TasksMax=4096
LimitNOFILE=65535
LimitNPROC=4096
```

---

## 4. Physical Security Controls

**Category**: Physical Security

Physical access to a server can bypass all software security controls. Chassis intrusion detection, USB port control, and boot device restrictions form the last line of defense against physical attack vectors.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| Chassis intrusion detection is enabled in BIOS/BMC | LOW | Detects physical tampering; useful for compliance (PCI-DSS physical security requirements) |
| Chassis intrusion alerts are forwarded to monitoring/SIEM | LOW | Detection without alerting is useless |
| USB ports are disabled in BIOS for servers that do not need them | MEDIUM | USB ports enable BadUSB attacks, unauthorized boot media, and data exfiltration |
| Front-panel USB ports are disabled or physically blocked | MEDIUM | Front-panel ports are accessible even in locked racks when the door is ajar |
| Internal USB ports used for hypervisor boot (ESXi, Proxmox) are the only enabled ports | LOW | Minimize attack surface to only necessary USB devices |
| Boot from removable media is disabled in BIOS when not needed for provisioning | MEDIUM | Prevents booting from unauthorized USB or optical media |
| Server is in a locked rack or cage with controlled access | MEDIUM | Physical access control is required for compliance frameworks (SOC 2, PCI-DSS, HIPAA) |
| Physical console access requires authentication (BIOS password, BMC credential) | MEDIUM | An unlocked console provides full system access |
| Debug/diagnostic ports (serial, JTAG) are disabled or physically protected | LOW | Debug ports can provide low-level system access bypassing OS controls |

### Common Mistakes

**Mistake**: USB ports left enabled on production servers that never use them.

Why it is a problem: USB ports are an attack vector for BadUSB (firmware-level keystroke injection), boot media attacks, and data exfiltration. Production servers managed entirely via BMC and network have no legitimate need for USB ports.

```bash
# Dell iDRAC: Disable USB ports
racadm set BIOS.IntegratedDevices.InternalUsb Off
racadm set BIOS.IntegratedDevices.UsbPorts AllOff
# Keep USB enabled only if needed for hypervisor boot from internal USB

# HPE iLO: Configure USB via BIOS settings
ilorest set InternalSDCardSlot=Disabled --select Bios. --commit
ilorest set UsbControl=ExternalUsbDisabled --select Bios. --commit

# Linux: Disable USB storage at kernel level (defense in depth)
echo "blacklist usb-storage" > /etc/modprobe.d/disable-usb-storage.conf
echo "install usb-storage /bin/true" >> /etc/modprobe.d/disable-usb-storage.conf
```

**Mistake**: Chassis intrusion detection available but not enabled or not forwarded.

```bash
# Dell iDRAC: Enable chassis intrusion detection and alerting
racadm set System.ChassisIntrusion.IntrusionDetection Enabled
racadm set System.ChassisIntrusion.IntrusionAction LogOnly
# Configure alert forwarding (SNMP trap or email)
racadm set iDRAC.SNMP.Alert.1.Enabled Enabled
racadm set iDRAC.SNMP.Alert.1.Destination monitoring.example.com

# HPE iLO: Chassis intrusion is reported via SEL
# Forward SEL to syslog for central alerting
ilorest set RemoteSyslogEnabled=True --select ManagerNetworkService. --commit
```

---

## 5. Configuration Drift and Compliance

**Category**: Configuration Compliance

Server configurations drift over time due to manual changes, emergency fixes, and inconsistent provisioning. Detecting and remediating drift is essential for maintaining security baselines and compliance posture.

### What to Check

| Check | Severity | Rationale |
|-------|----------|-----------|
| BIOS configuration is baselined and version-controlled (exported JSON/XML profiles) | MEDIUM | Without a baseline, drift cannot be detected |
| BMC configuration is managed via automation (Ansible, Redfish scripts), not manual web UI | MEDIUM | Manual configuration is error-prone and creates inconsistency across fleet |
| OS-level configuration is managed by a configuration management tool (Ansible, Puppet, Chef, Salt) | MEDIUM | Manual sysctl and tuning changes drift and are lost on reprovisioning |
| CIS Benchmark compliance is assessed regularly (CIS-CAT, OpenSCAP, Lynis) | MEDIUM | CIS Benchmarks provide an industry-standard security baseline |
| Configuration changes to BMC and BIOS are logged and auditable | MEDIUM | Required for SOC 2, PCI-DSS, and HIPAA compliance |
| Server build process is documented and reproducible | MEDIUM | Undocumented builds create "snowflake" servers that cannot be rebuilt consistently |
| BIOS settings are compared across fleet for consistency | LOW | Configuration variance across identical hardware indicates manual changes or incomplete automation |
| Kernel parameter changes are persisted correctly (sysctl.d, grub, tuned) and survive reboot | LOW | Temporary changes via `sysctl -w` are lost on reboot, creating a false sense of compliance |

### Common Mistakes

**Mistake**: Tuning parameters applied via `sysctl -w` but not persisted.

```bash
# BAD - change is lost on reboot
sysctl -w net.core.somaxconn=65535

# GOOD - persisted via sysctl.d
echo "net.core.somaxconn = 65535" > /etc/sysctl.d/99-somaxconn.conf
sysctl --system  # Reload all sysctl files
```

**Mistake**: BIOS configuration managed manually via web UI with no export/baseline.

```bash
# Dell: Export BIOS profile for baselining
racadm get BIOS > /var/lib/bios-baseline/server01_bios_$(date +%Y%m%d).txt

# Dell: Export via Redfish API
curl -sk -u admin:password \
  https://idrac-server01/redfish/v1/Systems/System.Embedded.1/Bios \
  | jq '.Attributes' > bios_baseline.json

# HPE: Export via ilorest
ilorest get --json --select Bios. > bios_baseline.json

# Compare against baseline
diff bios_baseline.json bios_current.json
```

### Fix Patterns

**Ansible playbook for server baseline compliance**:

```yaml
# roles/server-baseline/tasks/main.yml
---
- name: Set sysctl parameters
  ansible.posix.sysctl:
    name: "{{ item.key }}"
    value: "{{ item.value }}"
    state: present
    sysctl_file: /etc/sysctl.d/99-baseline.conf
    reload: true
  loop: "{{ server_sysctl_params | dict2items }}"

- name: Configure tuned profile
  ansible.builtin.command:
    cmd: tuned-adm profile {{ tuned_profile }}
  changed_when: true

- name: Disable Transparent Hugepages
  ansible.builtin.copy:
    dest: /etc/systemd/system/disable-thp.service
    content: |
      [Unit]
      Description=Disable Transparent Huge Pages
      DefaultDependencies=no
      After=sysinit.target local-fs.target
      Before=basic.target

      [Service]
      Type=oneshot
      ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/enabled'
      ExecStart=/bin/sh -c 'echo never > /sys/kernel/mm/transparent_hugepage/defrag'

      [Install]
      WantedBy=basic.target
  notify: Enable disable-thp service

- name: Set file descriptor limits
  ansible.builtin.copy:
    dest: /etc/security/limits.d/99-nofile.conf
    content: |
      * soft nofile 65535
      * hard nofile 65535

- name: Ensure CPU microcode is installed
  ansible.builtin.package:
    name: "{{ 'intel-microcode' if ansible_processor[1] is match('.*Intel.*') else 'amd64-microcode' }}"
    state: latest

- name: Run CIS benchmark assessment
  ansible.builtin.command:
    cmd: oscap xccdf eval --profile xccdf_org.ssgproject.content_profile_cis
         --results /var/log/oscap/cis-results.xml
         --report /var/log/oscap/cis-report.html
         /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml
  register: oscap_result
  failed_when: false
  changed_when: false
```

**OpenSCAP compliance scanning**:

```bash
# Install OpenSCAP and SCAP Security Guide
dnf install -y openscap-scanner scap-security-guide

# Run CIS Level 1 Server assessment
oscap xccdf eval \
  --profile xccdf_org.ssgproject.content_profile_cis_server_l1 \
  --results /var/log/oscap/cis-results-$(date +%Y%m%d).xml \
  --report /var/log/oscap/cis-report-$(date +%Y%m%d).html \
  /usr/share/xml/scap/ssg/content/ssg-rhel9-ds.xml

# Generate remediation script from failed checks
oscap xccdf generate fix \
  --fix-type bash \
  --result-id "" \
  /var/log/oscap/cis-results-$(date +%Y%m%d).xml \
  > /var/log/oscap/remediation.sh
```

---

## Quick Reference: Severity Summary

| Severity | Server Configuration Findings |
|----------|-------------------------------|
| CRITICAL | BMC on production network (not isolated); BMC accessible from internet; default BMC credentials (`root/calvin`, `ADMIN/ADMIN`); default SNMP community strings (`public/private`); IPMI cipher suite 0 enabled |
| HIGH | Secure Boot disabled; TPM disabled; virtualization extensions disabled; ECC memory disabled; BMC firmware outdated with known CVEs; IPMI using weak passwords; SNMPv1/v2c in use; BMC shared NIC mode; CPU microcode not loaded; speculative execution mitigations missing |
| MEDIUM | Boot order unrestricted; no BIOS password; CSM/legacy boot enabled; IOMMU disabled; NUMA misconfigured; wrong memory operating mode; wrong performance profile; Hyperthreading/SMT not evaluated for workload; THP enabled for databases; wrong CPU governor; default sysctl values; missing tuned profile; no IRQ balancing; sysctl hardening missing; USB ports enabled unnecessarily; BMC self-signed certs; unused BMC services enabled; SOL unrestricted; SEL not forwarded; no BMC alerting; BMC accounts not least-privilege; LDAP bind unencrypted; configuration not baselined; no configuration management; no CIS assessment; systemd resource limits missing |
| LOW | C-states not optimized; memory frequency not maximized; SR-IOV not enabled; BMC session timeout too long; BMC virtual media not restricted; chassis intrusion not enabled; debug ports not disabled; I/O scheduler mismatch; kernel panic behavior not configured; dirty ratio not tuned; fleet BIOS consistency not checked; sysctl changes not persisted; Redfish using basic auth |
| INFO | Well-configured tuned profile matching workload; NUMA-aware application placement documented; CIS Benchmark Level 2 compliance verified; BMC configuration fully automated via Ansible/Redfish |

---

## References

- CIS Benchmarks: [CIS Distribution-specific benchmarks](https://www.cisecurity.org/cis-benchmarks) (RHEL, Ubuntu, SLES)
- CIS Benchmark for Dell EMC PowerEdge Servers
- CIS Benchmark for Independent BIOS Settings
- NIST SP 800-123: Guide to General Server Security
- NIST SP 800-147: BIOS Protection Guidelines
- Dell iDRAC RACADM CLI Guide (support.dell.com)
- HPE iLO RESTful API Documentation (developer.hpe.com)
- Supermicro IPMI User Guide and SUM CLI Reference
- Red Hat Performance Tuning Guide
- Linux Kernel Documentation: NUMA, hugepages, sysctl
- DMTF Redfish API Specification (dmtf.org/standards/redfish)
