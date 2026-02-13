# Hypervisor Configuration and Hardening

Reference checklist for the `virtualization-reviewer` agent when evaluating hypervisor configuration across VMware vSphere/ESXi, Proxmox VE, and KVM/libvirt environments. This file complements the inline criteria in `review-virtualization.md` with detailed checkpoints, severity classifications, platform-specific examples, and remediation guidance.

Covers: SSH access, firewall policy, NTP/time synchronization, syslog forwarding, SNMP configuration, lockdown mode, VMkernel network separation, certificate management, directory services integration, local emergency accounts, firmware posture, and platform-specific hardening for Proxmox (corosync, Ceph, web UI) and KVM/libvirt (virtio, SELinux/sVirt, CPU pinning, NUMA, hugepages, storage pools).

---

## 1. SSH and Remote Shell Access

**Category**: Remote Access Security

SSH on hypervisor hosts should be disabled by default and enabled only for troubleshooting. Persistent SSH access expands the attack surface and creates a direct path to the hypervisor kernel. All three major platforms share this concern, though the operational impact of disabling SSH varies.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 1.1 | SSH service is disabled on ESXi hosts during normal operation | **HIGH** | vSphere |
| 1.2 | ESXi Shell (TSM) is disabled unless actively troubleshooting | **HIGH** | vSphere |
| 1.3 | SSH timeout is configured (`UserVars.ESXiShellInteractiveTimeOut` and `UserVars.ESXiShellTimeOut`) | **MEDIUM** | vSphere |
| 1.4 | SSH warning banner is configured to display legal notice | **LOW** | All |
| 1.5 | SSH root login is restricted (`PermitRootLogin no` or `prohibit-password`) | **HIGH** | Proxmox, KVM |
| 1.6 | SSH password authentication is disabled in favor of key-based authentication | **MEDIUM** | Proxmox, KVM |
| 1.7 | SSH listen address is bound to management network interface only | **MEDIUM** | All |
| 1.8 | SSH idle timeout is configured (`ClientAliveInterval`, `ClientAliveCountMax`) | **LOW** | Proxmox, KVM |
| 1.9 | SSH authorized keys are audited and limited to authorized personnel | **HIGH** | All |
| 1.10 | SSH protocol version 1 is explicitly disabled | **MEDIUM** | All |

### What to Look For

```bash
# ESXi SSH service status
vim-cmd hostsvc/enable_ssh        # Should NOT be called unconditionally
esxcli system settings advanced list -o /UserVars/ESXiShellInteractiveTimeOut
esxcli system settings advanced list -o /UserVars/ESXiShellTimeOut

# Proxmox / KVM SSH configuration
grep -E "^PermitRootLogin|^PasswordAuthentication|^ListenAddress|^Protocol" /etc/ssh/sshd_config

# Ansible / IaC patterns that enable SSH persistently
# BAD: enabling SSH as part of baseline
esxcli network firewall ruleset set --ruleset-id sshServer --enabled true
```

### Common Mistakes

**Mistake**: Leaving SSH enabled on ESXi hosts after troubleshooting.

Why it is a problem: ESXi SSH access bypasses vCenter RBAC and audit logging. An attacker with SSH access has root-level control of the hypervisor, including the ability to access VM memory, disk files, and cryptographic keys.

**Mistake**: Using password authentication for SSH on Proxmox or KVM hosts.

```bash
# BAD - password authentication enabled
PasswordAuthentication yes

# GOOD - key-based authentication only
PasswordAuthentication no
PubkeyAuthentication yes
```

---

## 2. Firewall Configuration

**Category**: Network Security

Hypervisor firewalls must follow a default-deny posture, permitting only the management protocols required for operation. ESXi uses its own firewall subsystem; Proxmox and KVM hosts use iptables/nftables or pve-firewall.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 2.1 | ESXi firewall is enabled and default policy is set to drop | **HIGH** | vSphere |
| 2.2 | ESXi firewall rulesets are restricted to specific source IP ranges (not `allowedAll`) | **HIGH** | vSphere |
| 2.3 | Only required firewall rulesets are enabled on ESXi | **MEDIUM** | vSphere |
| 2.4 | Proxmox `pve-firewall` is enabled at datacenter and node level | **HIGH** | Proxmox |
| 2.5 | Proxmox firewall default input policy is DROP | **HIGH** | Proxmox |
| 2.6 | KVM hosts have iptables/nftables rules restricting management access to authorized networks | **HIGH** | KVM |
| 2.7 | VXLAN, Geneve, or overlay network ports are restricted to cluster members only | **MEDIUM** | All |
| 2.8 | IPMI/iLO/iDRAC management ports are not exposed to production networks | **CRITICAL** | All |

### Common Mistakes

**Mistake**: ESXi firewall rulesets with `allowedAll` set to true.

```bash
# BAD - allows any source IP to access the service
esxcli network firewall ruleset set --ruleset-id httpClient --allowed-all true

# GOOD - restrict to management subnet
esxcli network firewall ruleset set --ruleset-id httpClient --allowed-all false
esxcli network firewall ruleset allowedip add --ruleset-id httpClient --ip-address 10.0.1.0/24
```

**Mistake**: Proxmox firewall disabled at the datacenter level, nullifying node-level rules.

```ini
# /etc/pve/firewall/cluster.fw
[OPTIONS]
# BAD - firewall disabled
enable: 0

# GOOD - firewall enabled with default deny
enable: 1
policy_in: DROP
policy_out: ACCEPT
```

---

## 3. Time Synchronization (NTP/PTP)

**Category**: Time Services

Accurate time is critical for hypervisor environments. Clock skew causes vMotion failures, SAML authentication errors, certificate validation failures, log correlation problems, and replication inconsistencies. All VMs inherit time sensitivity from their host.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 3.1 | NTP or PTP is configured on all hypervisor hosts | **HIGH** | All |
| 3.2 | At least two independent NTP sources are configured for redundancy | **MEDIUM** | All |
| 3.3 | NTP sources are internal stratum-1 or stratum-2 servers, not public pool servers in production | **MEDIUM** | All |
| 3.4 | ESXi NTP service is set to start with the host (`ntpd` policy: `on`) | **MEDIUM** | vSphere |
| 3.5 | NTP authentication is enabled (symmetric key or autokey) where supported | **LOW** | All |
| 3.6 | Time drift monitoring and alerting is configured | **LOW** | All |
| 3.7 | VM Tools time synchronization is configured appropriately (enabled or disabled with rationale) | **MEDIUM** | vSphere |

### Common Mistakes

**Mistake**: No NTP configured on ESXi hosts.

```bash
# Check ESXi NTP configuration
esxcli system ntp get

# Configure NTP on ESXi
esxcli system ntp set --server=ntp1.corp.example.com --server=ntp2.corp.example.com
esxcli system ntp set --enabled=true
```

**Mistake**: Using public NTP pools in production environments behind firewalls.

Why it is a problem: Public NTP pools require outbound UDP 123 through the firewall and introduce a dependency on external infrastructure. If the firewall blocks NTP or the pool is unreachable, time drift begins silently. Use internal NTP servers synchronized to GPS or authoritative sources.

```yaml
# Proxmox - /etc/chrony/chrony.conf
# BAD - public pools in an enterprise environment
pool pool.ntp.org iburst

# GOOD - internal NTP infrastructure
server ntp1.corp.example.com iburst
server ntp2.corp.example.com iburst
```

---

## 4. Syslog Forwarding

**Category**: Logging and Audit

Hypervisor logs must be forwarded to a central logging system for security monitoring, incident response, and compliance. Local-only logs are lost when hosts fail and cannot be correlated across the environment.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 4.1 | Remote syslog is configured on all ESXi hosts | **HIGH** | vSphere |
| 4.2 | Syslog uses TCP or TLS transport, not UDP only | **MEDIUM** | All |
| 4.3 | Syslog includes all relevant log sources (hostd, vpxa, vmkernel, fdm, vobd) | **MEDIUM** | vSphere |
| 4.4 | vCenter Server Appliance logs are forwarded to central syslog | **HIGH** | vSphere |
| 4.5 | Proxmox host logs are forwarded via rsyslog or systemd-journal-remote | **HIGH** | Proxmox |
| 4.6 | KVM host logs (libvirtd, QEMU, audit) are forwarded to central log system | **HIGH** | KVM |
| 4.7 | Log rotation is configured to prevent disk exhaustion on the hypervisor | **MEDIUM** | All |
| 4.8 | Syslog connectivity is monitored and alerting is configured for forwarding failures | **MEDIUM** | All |

### Common Mistakes

**Mistake**: ESXi syslog configured with UDP only and no TLS.

```bash
# BAD - UDP syslog, no encryption, unreliable delivery
esxcli system syslog config set --loghost=udp://syslog.corp.example.com:514

# GOOD - TLS syslog for encrypted, reliable delivery
esxcli system syslog config set --loghost=ssl://syslog.corp.example.com:6514
esxcli system syslog reload
```

**Mistake**: Syslog partition filling up because remote forwarding silently failed.

Why it is a problem: ESXi scratch partitions and /var/log are small. Without remote forwarding and local rotation, logs fill the partition, causing host instability. Without monitoring on the forwarding pipeline, failures are discovered only during incident response when the logs are needed.

---

## 5. SNMP Configuration

**Category**: Monitoring Security

SNMP is used for hypervisor monitoring but introduces significant security risk when misconfigured. SNMPv1 and v2c transmit community strings in cleartext. Default or well-known community strings are trivially exploitable for reconnaissance and, with write access, for configuration manipulation.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 5.1 | SNMP v1 and v2c are disabled; only SNMPv3 is used | **HIGH** | All |
| 5.2 | Default community strings (`public`, `private`) are not configured | **CRITICAL** | All |
| 5.3 | SNMPv3 uses `authPriv` security level (authentication + encryption) | **HIGH** | All |
| 5.4 | SNMPv3 authentication uses SHA-256 or SHA-512 (not MD5) | **MEDIUM** | All |
| 5.5 | SNMPv3 privacy (encryption) uses AES-128 or AES-256 (not DES) | **MEDIUM** | All |
| 5.6 | SNMP write access is disabled unless explicitly required with documented justification | **HIGH** | All |
| 5.7 | SNMP is bound to management network interface only | **MEDIUM** | All |
| 5.8 | If SNMPv2c is required for legacy monitoring, community strings are non-default and ACL-restricted | **MEDIUM** | All |

### Common Mistakes

**Mistake**: ESXi SNMP configured with default `public` community string.

```bash
# BAD - default community string
esxcli system snmp set --communities=public --enable true

# GOOD - SNMPv3 with authPriv
esxcli system snmp set --enable true
esxcli system snmp set --authentication SHA256
esxcli system snmp set --privacy AES128
esxcli system snmp hash --auth-hash <auth_secret> --priv-hash <priv_secret> --raw-secret
esxcli system snmp set --users <username>/SHA256/<auth_hash>/AES128/<priv_hash>/priv
```

---

## 6. Lockdown Mode and Access Control

**Category**: Access Restriction

Lockdown mode restricts direct access to ESXi hosts, forcing all management through vCenter. This prevents unauthorized direct host manipulation, ensures audit logging through vCenter, and reduces the attack surface. Proxmox and KVM environments achieve similar goals through PAM, role-based access, and directory services integration.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 6.1 | ESXi lockdown mode is enabled (Normal or Strict) on all production hosts | **HIGH** | vSphere |
| 6.2 | ESXi Exception Users list is minimized and documented | **MEDIUM** | vSphere |
| 6.3 | DCUI (Direct Console User Interface) access is restricted to authorized personnel | **MEDIUM** | vSphere |
| 6.4 | vCenter SSO is integrated with enterprise directory service (Active Directory, LDAP) | **HIGH** | vSphere |
| 6.5 | Local `administrator@vsphere.local` password is stored in a sealed break-glass envelope or PAM vault | **HIGH** | vSphere |
| 6.6 | Proxmox PAM and PVE realms are integrated with LDAP/AD for centralized authentication | **HIGH** | Proxmox |
| 6.7 | Proxmox web UI access is restricted to management network via firewall or reverse proxy | **HIGH** | Proxmox |
| 6.8 | Proxmox two-factor authentication (TOTP, YubiKey, WebAuthn) is enabled for administrative accounts | **MEDIUM** | Proxmox |
| 6.9 | KVM/libvirt uses polkit policies to restrict management access to authorized users/groups | **HIGH** | KVM |
| 6.10 | Default local accounts (root, admin) have strong, unique passwords per host | **HIGH** | All |
| 6.11 | Service accounts used by monitoring/backup tools have minimum necessary privileges | **MEDIUM** | All |
| 6.12 | Role-based access control is implemented with granular roles (not everyone is Administrator) | **HIGH** | All |

### Common Mistakes

**Mistake**: ESXi lockdown mode disabled, allowing direct host management that bypasses vCenter RBAC and audit.

```
# vSphere PowerCLI
# BAD - lockdown mode not enabled
Get-VMHost | Where-Object { $_.ExtensionData.Config.LockdownMode -eq "lockdownDisabled" }

# GOOD - enable normal lockdown mode
Get-VMHost | ForEach-Object {
    ($_ | Get-View).EnterLockdownMode()
}
```

**Mistake**: Proxmox web UI exposed to all VLANs without restriction.

```ini
# /etc/default/pveproxy
# GOOD - restrict web UI to management network
ALLOW_FROM="10.0.1.0/24,10.0.2.0/24"
DENY_FROM="all"
POLICY="allow"
```

**Mistake**: All administrators using the built-in `administrator@vsphere.local` account.

Why it is a problem: Shared accounts eliminate individual accountability and make audit trails meaningless. If the password is compromised, there is no way to revoke access for a single person without changing it for everyone. Integrate with directory services and assign named accounts with appropriate roles.

---

## 7. VMkernel Network Separation (vSphere)

**Category**: Network Isolation

VMkernel interfaces carry sensitive traffic types: management, vMotion, vSAN, IP storage (iSCSI/NFS), Fault Tolerance logging, and replication. Mixing these traffic types on shared networks creates security and performance risks. Each traffic type should use a dedicated VMkernel adapter, VLAN, and subnet.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 7.1 | Management traffic uses a dedicated VMkernel adapter and VLAN | **HIGH** | vSphere |
| 7.2 | vMotion traffic is on a dedicated VMkernel adapter and VLAN, isolated from production networks | **HIGH** | vSphere |
| 7.3 | vSAN traffic uses a dedicated VMkernel adapter and VLAN (10 GbE minimum recommended) | **HIGH** | vSphere |
| 7.4 | IP storage (iSCSI/NFS) traffic uses a dedicated VMkernel adapter and VLAN | **HIGH** | vSphere |
| 7.5 | Fault Tolerance logging uses a dedicated VMkernel adapter and VLAN | **MEDIUM** | vSphere |
| 7.6 | VMkernel adapters for vMotion, vSAN, and IP storage do not have default gateway assigned (routing isolation) | **MEDIUM** | vSphere |
| 7.7 | Jumbo frames (MTU 9000) are consistently configured end-to-end for vMotion, vSAN, and IP storage paths | **MEDIUM** | vSphere |
| 7.8 | vMotion traffic is encrypted for clusters spanning L3 boundaries or untrusted networks | **HIGH** | vSphere |
| 7.9 | Proxmox corosync traffic uses a dedicated network interface separate from management and VM traffic | **HIGH** | Proxmox |
| 7.10 | Proxmox Ceph public and cluster networks are separated from management and VM traffic | **HIGH** | Proxmox |
| 7.11 | KVM/libvirt bridge interfaces for VM traffic are separate from host management interfaces | **MEDIUM** | KVM |

### Common Mistakes

**Mistake**: vMotion and management traffic sharing the same VMkernel adapter and VLAN.

Why it is a problem: vMotion transfers VM memory contents in cleartext (unless encrypted). An attacker on the management VLAN can sniff vMotion traffic to capture VM memory, which may contain credentials, encryption keys, and sensitive data. Additionally, vMotion bursts can saturate the management network, causing vCenter connectivity issues.

```powershell
# PowerCLI - Check VMkernel adapter separation
Get-VMHostNetworkAdapter -VMKernel | Select-Object VMHost, Name, IP, SubnetMask, `
    @{N="Services";E={($_ | Get-VMHostNetworkAdapter | Get-VMHostNetworkAdapterVMKernelServiceInfo).Service}}
```

**Mistake**: Jumbo frames configured on VMkernel adapter but not on physical switch or distributed switch uplinks.

Why it is a problem: MTU mismatch causes packet fragmentation or silent drops. vSAN and iSCSI performance degrades dramatically with MTU mismatches. Test end-to-end with `vmkping -s 8972 -d <target>` to verify jumbo frame path.

---

## 8. Certificate Management

**Category**: Cryptographic Hygiene

Hypervisor environments use TLS certificates for management interfaces, API endpoints, vCenter-to-host communication, and web UIs. Default self-signed certificates are acceptable for initial deployment but must be replaced with certificates from an enterprise CA for production. Expired or untrusted certificates cause service disruptions and security warnings that train administrators to ignore certificate errors.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 8.1 | ESXi host certificates are issued by VMCA or an enterprise subordinate CA, not default self-signed | **MEDIUM** | vSphere |
| 8.2 | vCenter Server certificate is issued by an enterprise CA trusted by administrator workstations | **MEDIUM** | vSphere |
| 8.3 | VMCA is configured as a subordinate CA to the enterprise PKI (hybrid or full custom mode) | **MEDIUM** | vSphere |
| 8.4 | Certificate expiration monitoring is in place with alerting at 30, 14, and 7 days before expiry | **HIGH** | All |
| 8.5 | TLS 1.0 and TLS 1.1 are disabled on all management interfaces | **HIGH** | All |
| 8.6 | Weak cipher suites (RC4, DES, 3DES, NULL, export ciphers) are disabled | **HIGH** | All |
| 8.7 | Proxmox web UI certificate is replaced with an enterprise CA-signed certificate | **MEDIUM** | Proxmox |
| 8.8 | Proxmox SPICE/noVNC console connections use valid TLS certificates | **MEDIUM** | Proxmox |
| 8.9 | KVM/libvirt TLS is configured for remote management (libvirtd certificate and CA) | **MEDIUM** | KVM |
| 8.10 | Certificate private keys are stored securely with appropriate file permissions (0600) | **HIGH** | All |
| 8.11 | Certificate renewal is automated where possible (ACME/Let's Encrypt for non-production, enterprise CA auto-enrollment for production) | **LOW** | All |

### Common Mistakes

**Mistake**: ESXi hosts using default VMCA certificates that have expired because VMCA was not monitored.

Why it is a problem: Expired certificates cause vCenter to lose management connectivity to ESXi hosts, breaking HA, DRS, and monitoring. In VMCA-managed environments, certificate lifecycle is automated but the VMCA root certificate itself must be tracked.

**Mistake**: Disabling TLS certificate verification to work around self-signed certificates.

```python
# BAD - disabling TLS verification in automation scripts
from pyVim.connect import SmartConnect
si = SmartConnect(host="vcenter.example.com", user="admin", pwd="pass",
                  disableSslCertValidation=True)

# GOOD - trust the CA certificate
import ssl
context = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
context.load_verify_locations("/etc/pki/tls/certs/enterprise-ca.pem")
si = SmartConnect(host="vcenter.example.com", user="admin", pwd="pass",
                  sslContext=context)
```

---

## 9. Proxmox VE Specific Configuration

**Category**: Proxmox Hardening

Proxmox VE has platform-specific considerations including corosync cluster communication, Ceph storage integration, the web management UI, and storage backend configuration. These components require explicit hardening beyond general Linux server practices.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 9.1 | Corosync uses a dedicated network link separate from management and VM traffic | **HIGH** | Proxmox |
| 9.2 | Corosync is configured with redundant links (link0 and link1) for cluster communication resilience | **MEDIUM** | Proxmox |
| 9.3 | Corosync encryption and authentication are enabled (`crypto_cipher` and `crypto_hash` in corosync.conf) | **HIGH** | Proxmox |
| 9.4 | Ceph monitors and OSDs are bound to dedicated storage network interfaces | **HIGH** | Proxmox |
| 9.5 | Ceph public and cluster networks are separated (`public_network` and `cluster_network` in ceph.conf) | **HIGH** | Proxmox |
| 9.6 | Ceph authentication is enabled (`auth_cluster_required = cephx`, `auth_service_required = cephx`, `auth_client_required = cephx`) | **CRITICAL** | Proxmox |
| 9.7 | Ceph dashboard (if enabled) is restricted to management network and uses HTTPS | **HIGH** | Proxmox |
| 9.8 | Proxmox web UI listens on management network only (configured via `/etc/default/pveproxy`) | **HIGH** | Proxmox |
| 9.9 | Proxmox API tokens use minimum required privileges and are scoped with token-specific permissions | **MEDIUM** | Proxmox |
| 9.10 | Proxmox storage backends (ZFS, LVM, LVM-thin, Ceph, NFS, iSCSI) are configured with appropriate redundancy | **MEDIUM** | Proxmox |
| 9.11 | Proxmox subscription or no-subscription repository is correctly configured (no mixing) | **LOW** | Proxmox |
| 9.12 | Proxmox node kernel and packages are regularly updated with security patches | **HIGH** | Proxmox |

### Example Correct Configuration

**Corosync with encryption and redundant links**:

```
# /etc/pve/corosync.conf
totem {
    version: 2
    cluster_name: prod-cluster
    transport: knet
    crypto_cipher: aes256
    crypto_hash: sha256

    interface {
        linknumber: 0
    }
    interface {
        linknumber: 1
    }
}

nodelist {
    node {
        name: pve1
        nodeid: 1
        ring0_addr: 10.0.10.1    # Dedicated corosync link 0
        ring1_addr: 10.0.11.1    # Dedicated corosync link 1
    }
    node {
        name: pve2
        nodeid: 2
        ring0_addr: 10.0.10.2
        ring1_addr: 10.0.11.2
    }
    node {
        name: pve3
        nodeid: 3
        ring0_addr: 10.0.10.3
        ring1_addr: 10.0.11.3
    }
}
```

**Ceph network separation**:

```ini
# /etc/ceph/ceph.conf
[global]
public_network = 10.0.20.0/24     # Client-facing (VM access to Ceph)
cluster_network = 10.0.21.0/24    # OSD replication traffic (separate from public)

auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

# Performance and safety settings
osd_pool_default_size = 3
osd_pool_default_min_size = 2
mon_osd_full_ratio = 0.95
mon_osd_nearfull_ratio = 0.85
```

---

## 10. KVM/libvirt Specific Configuration

**Category**: KVM Hardening

KVM/libvirt environments require attention to virtio device configuration, SELinux/sVirt mandatory access control, CPU topology (pinning, NUMA awareness), memory management (hugepages), and storage pool security. These settings directly impact performance, isolation, and security.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 10.1 | VMs use virtio drivers for disk, network, and memory ballooning | **MEDIUM** | KVM |
| 10.2 | SELinux is in enforcing mode on KVM hosts | **HIGH** | KVM (RHEL/CentOS) |
| 10.3 | sVirt (SELinux-based VM isolation) is active and VMs run with unique security labels | **HIGH** | KVM (RHEL/CentOS) |
| 10.4 | AppArmor profiles are enforced for libvirt/QEMU processes on Ubuntu/Debian hosts | **HIGH** | KVM (Ubuntu/Debian) |
| 10.5 | CPU pinning is configured for latency-sensitive workloads with documented rationale | **LOW** | KVM |
| 10.6 | NUMA topology is respected in VM placement (`numatune` in libvirt XML) | **MEDIUM** | KVM |
| 10.7 | Hugepages are configured for VMs requiring deterministic memory access latency | **LOW** | KVM |
| 10.8 | Host CPU is not overcommitted beyond documented ratios for the workload type | **MEDIUM** | KVM |
| 10.9 | Storage pools use appropriate drivers (dir, lvm, rbd, iscsi) with access controls | **MEDIUM** | KVM |
| 10.10 | Libvirt storage pool permissions restrict access to the libvirt process and authorized users | **HIGH** | KVM |
| 10.11 | QEMU is configured to run as a non-root user (`user` and `group` in qemu.conf) | **HIGH** | KVM |
| 10.12 | VM disk images use qcow2 with encryption for sensitive workloads (LUKS backing) | **MEDIUM** | KVM |
| 10.13 | Libvirt remote management uses TLS with certificate-based authentication | **HIGH** | KVM |
| 10.14 | KVM kernel modules are loaded with security-relevant parameters (`kvm.ignore_msrs=0`) | **LOW** | KVM |
| 10.15 | Memory ballooning is configured with minimum memory guarantees to prevent guest starvation | **MEDIUM** | KVM |

### Common Mistakes

**Mistake**: SELinux disabled or in permissive mode on KVM hosts.

```bash
# BAD - SELinux disabled
SELINUX=disabled    # in /etc/selinux/config

# GOOD - SELinux enforcing with sVirt
SELINUX=enforcing
SELINUXTYPE=targeted

# Verify sVirt labeling
ps -eZ | grep qemu
# Should show unique labels like:
# system_u:system_r:svirt_t:s0:c123,c456  /usr/libexec/qemu-kvm
```

**Mistake**: QEMU processes running as root.

```bash
# /etc/libvirt/qemu.conf
# BAD - QEMU runs as root
user = "root"
group = "root"

# GOOD - QEMU runs as dedicated non-root user
user = "qemu"
group = "qemu"
```

**Mistake**: NUMA topology ignored for large VMs spanning NUMA nodes.

```xml
<!-- BAD - no NUMA awareness, VM memory accesses cross NUMA boundaries -->
<vcpu placement="static">16</vcpu>
<memory unit="GiB">64</memory>

<!-- GOOD - NUMA-aware placement -->
<vcpu placement="static">16</vcpu>
<memory unit="GiB">64</memory>
<numatune>
  <memory mode="strict" nodeset="0-1"/>
</numatune>
<cpu mode="host-passthrough">
  <topology sockets="2" cores="8" threads="1"/>
  <numa>
    <cell id="0" cpus="0-7" memory="32" unit="GiB"/>
    <cell id="1" cpus="8-15" memory="32" unit="GiB"/>
  </numa>
</cpu>
```

### Example Correct Configuration

**Hugepages configuration for KVM**:

```bash
# /etc/sysctl.d/hugepages.conf
# Allocate 32 x 1GB hugepages (32 GB for VM use)
vm.nr_hugepages = 0
# Use 1GB hugepages via kernel boot parameter instead:
# hugepagesz=1G hugepages=32

# Or 2MB hugepages via sysctl
vm.nr_hugepages = 16384    # 16384 x 2MB = 32GB
```

```xml
<!-- libvirt VM configuration with hugepages -->
<memoryBacking>
  <hugepages>
    <page size="1" unit="GiB"/>
  </hugepages>
  <locked/>
</memoryBacking>
```

**Libvirt storage pool with restricted permissions**:

```xml
<pool type="dir">
  <name>vm-images</name>
  <target>
    <path>/var/lib/libvirt/images</path>
    <permissions>
      <mode>0711</mode>
      <owner>107</owner>   <!-- qemu user UID -->
      <group>107</group>   <!-- qemu group GID -->
    </permissions>
  </target>
</pool>
```

---

## 11. General Hardening

**Category**: Platform Hardening

Cross-platform hardening practices apply to all hypervisor environments regardless of vendor. These cover management network isolation, directory services integration, local emergency accounts, and firmware posture.

### Checkpoints

| # | Checkpoint | Severity | Platform |
|---|---|---|---|
| 11.1 | Management network is isolated from VM production traffic (dedicated VLAN/subnet) | **HIGH** | All |
| 11.2 | Management network is not routable from untrusted networks (DMZ, guest, internet) | **CRITICAL** | All |
| 11.3 | Out-of-band management (iLO, iDRAC, IPMI) is on a dedicated, isolated management network | **CRITICAL** | All |
| 11.4 | Directory services integration is configured for centralized authentication and authorization | **HIGH** | All |
| 11.5 | Local emergency accounts exist on every host for break-glass access when directory services are unavailable | **HIGH** | All |
| 11.6 | Local emergency account passwords are unique per host, stored in a PAM vault, and rotated on a schedule | **HIGH** | All |
| 11.7 | Hypervisor firmware (BIOS/UEFI) is current and matches vendor security advisories | **MEDIUM** | All |
| 11.8 | Server hardware firmware (BMC, NIC, storage controller, GPU) is current | **MEDIUM** | All |
| 11.9 | UEFI Secure Boot is enabled on hypervisor hosts | **MEDIUM** | All |
| 11.10 | TPM 2.0 is configured and used for measured boot or key storage where supported | **LOW** | All |
| 11.11 | Host power management policies do not compromise VM performance in production (no aggressive C-states) | **LOW** | All |
| 11.12 | Hypervisor host is dedicated to virtualization (no additional services, agents, or applications installed) | **MEDIUM** | All |
| 11.13 | Vendor security advisories (VMSAs, Proxmox SAs, QEMU CVEs) are tracked and patched within SLA | **HIGH** | All |
| 11.14 | Core dumps are configured to not write to shared storage or insecure locations | **MEDIUM** | All |

### Common Mistakes

**Mistake**: Management network routable from the guest VM network.

Why it is a problem: If a VM is compromised, the attacker can reach the hypervisor management interface from inside the guest network. This turns a VM compromise into a hypervisor compromise, affecting every VM on the host. Management networks must be completely isolated with no routing path from VM traffic networks.

**Mistake**: No local emergency accounts, relying entirely on directory services.

Why it is a problem: When Active Directory or LDAP is unavailable (network partition, AD outage, DNS failure), administrators are locked out of the hypervisor infrastructure at the exact moment they most need access. Maintain documented break-glass local accounts on every host.

**Mistake**: Hypervisor firmware not updated, leaving known vulnerabilities (speculative execution, BMC exploits) unpatched.

```bash
# Check ESXi host firmware/patch level
esxcli software vib list | head -20
esxcli software profile get

# Check Proxmox kernel and package versions
uname -r
pveversion -v

# Check KVM host for CPU vulnerabilities
grep -r . /sys/devices/system/cpu/vulnerabilities/
```

---

## Review Procedure Summary

When evaluating hypervisor configuration:

1. **Identify the platform(s)**: Determine whether the environment uses vSphere/ESXi, Proxmox VE, KVM/libvirt, or a combination. Apply platform-specific checks accordingly.
2. **Assess remote access**: Verify SSH is disabled or restricted, lockdown mode is enabled (vSphere), and management access uses centralized authentication.
3. **Verify network separation**: Confirm VMkernel/management traffic types are on dedicated adapters, VLANs, and subnets. Check that management networks are isolated from VM traffic.
4. **Check time and logging**: Verify NTP is configured with internal sources and syslog forwarding is active to a central system.
5. **Evaluate monitoring security**: Confirm SNMP uses v3 with authPriv, no default community strings, and monitoring is bound to management interfaces.
6. **Review certificate hygiene**: Check for enterprise CA certificates, TLS version enforcement, and expiration monitoring.
7. **Inspect platform-specific hardening**: For Proxmox, check corosync encryption and Ceph authentication. For KVM, check SELinux/sVirt, QEMU user, and NUMA configuration.
8. **Validate emergency access**: Confirm local break-glass accounts exist with unique, vaulted passwords.
9. **Assess firmware posture**: Check hypervisor, BIOS/UEFI, and BMC firmware versions against vendor security advisories.
10. **Classify each finding** using the severity levels in each section's checkpoint table.

---

## Quick Reference: Severity Summary

| Severity | Hypervisor Configuration Findings |
|----------|----------------------------------|
| CRITICAL | Default SNMP community strings on production hosts; management network routable from untrusted networks; out-of-band management (iLO/iDRAC/IPMI) on production network; Ceph authentication disabled |
| HIGH | SSH enabled persistently on ESXi; firewall disabled or permissive (`allowedAll`); no NTP configured; no syslog forwarding; SNMP v1/v2c in use; lockdown mode disabled; no directory services integration; TLS 1.0/1.1 enabled; SELinux disabled on KVM hosts; QEMU running as root; no certificate expiration monitoring; vendor security advisories not tracked; vMotion/management on shared network; emergency accounts missing; root SSH login permitted |
| MEDIUM | No SSH timeout configured; syslog using UDP only; weak SNMP hash algorithms; VMkernel jumbo frame mismatch; self-signed certificates in production; no corosync redundant links; NUMA topology ignored; host CPU overcommitted without documentation; no VM disk encryption for sensitive workloads; firmware not current; host not dedicated to virtualization; missing log rotation |
| LOW | No SSH login banner; NTP authentication not enabled; no time drift monitoring; SSH idle timeout not set; certificate renewal not automated; TPM not configured; aggressive power management states; hugepages not configured; CPU pinning not documented |
| INFO | Well-separated VMkernel networks with documented purpose; enterprise CA-managed certificates with automated renewal; corosync with encrypted redundant links and dedicated network; SELinux enforcing with verified sVirt labels; NUMA-aware VM placement with performance validation |

---

## References

- CIS VMware ESXi 8.0 Benchmark, v1.1.0
- CIS VMware ESXi 7.0 Benchmark, v1.4.0
- VMware vSphere 8 Security Configuration & Hardening Guide
- VMware Security Advisories (VMSAs): https://www.vmware.com/security/advisories.html
- Proxmox VE Administration Guide: Cluster Manager, Firewall, Ceph
- Proxmox VE Security Best Practices (Proxmox Wiki)
- Red Hat Enterprise Linux - Virtualization Security Guide
- CIS Red Hat Enterprise Linux 9 Benchmark (KVM host hardening)
- NIST SP 800-125: Guide to Security for Full Virtualization Technologies
- NIST SP 800-125A: Security Recommendations for Server-based Hypervisor Platforms
- DISA STIG for VMware vSphere ESXi 7.0 / 8.0
- libvirt documentation: TLS setup, SELinux/sVirt integration
