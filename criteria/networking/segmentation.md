# Network Segmentation Reference Checklist

Canonical reference for evaluating network segmentation design, VLAN architecture, inter-VLAN
routing controls, micro-segmentation, and compliance-driven isolation requirements.
Complements the inlined criteria in `review-networking.md` with CCIE-equivalent depth covering
enterprise VLAN design patterns, private VLANs, TrustSec/SGT, and regulatory segmentation
mandates.

Covers: VLAN design by function, inter-VLAN routing control, native VLAN security, private
VLANs, micro-segmentation (SGT/TrustSec, NSX), out-of-band management, PCI/HIPAA
segmentation, east-west filtering, and trust zone isolation.

---

## 1. VLAN Design Principles

### 1.1 Functional Segmentation

| # | Checkpoint | Severity |
|---|---|---|
| 1.1.1 | VLANs are assigned by function, not by physical location (role-based segmentation) | **HIGH** |
| 1.1.2 | Management traffic uses a dedicated VLAN separate from user data VLANs | **CRITICAL** |
| 1.1.3 | Voice traffic uses a dedicated voice VLAN with QoS marking | **HIGH** |
| 1.1.4 | Guest/visitor traffic uses an isolated VLAN with internet-only access | **HIGH** |
| 1.1.5 | IoT and building automation devices are on isolated VLANs with restricted access | **HIGH** |
| 1.1.6 | Storage traffic (iSCSI, NFS, FCoE) uses dedicated VLANs not accessible from user segments | **CRITICAL** |
| 1.1.7 | DMZ servers use dedicated VLANs with strict inter-zone firewall rules | **HIGH** |
| 1.1.8 | Server/application tiers use separate VLANs (web, app, database) | **HIGH** |
| 1.1.9 | VLAN 1 is not used for any production traffic | **HIGH** |
| 1.1.10 | VLAN IDs follow a documented, consistent numbering scheme | **LOW** |
| 1.1.11 | VLAN design is documented in a network diagram or IPAM system | **MEDIUM** |
| 1.1.12 | Broadcast domain size is controlled (< 250-500 hosts per VLAN for performance) | **MEDIUM** |

### Recommended VLAN Allocation Scheme

| VLAN Range | Purpose | Trust Level | Notes |
|------------|---------|-------------|-------|
| 1 | Default (unused) | N/A | Never use for production traffic |
| 2-99 | Infrastructure | High | Management, routing, monitoring |
| 100-199 | Server/Data Center | High | Web, app, database tiers |
| 200-299 | Storage | Critical | iSCSI, NFS, vMotion, backup |
| 300-499 | User/Workstation | Medium | Segmented by department or role |
| 500-599 | Voice | Medium-High | Dedicated QoS, 802.1p marking |
| 600-699 | Wireless | Medium | Corporate SSID traffic |
| 700-799 | Guest/Visitor | Low | Internet-only, no internal access |
| 800-899 | IoT/Building | Low | Isolated, restricted access |
| 900-998 | DMZ/External | Low | Public-facing services |
| 999 | Dead-end/Parking | None | Unused port assignment |
| 1000-4094 | Extended | Varies | Overflow, special purpose |

### What to Look For

```regex
# VLAN 1 in use for production
(?i)switchport\s+access\s+vlan\s+1\b
(?i)interface\s+Vlan1\s*\n[^!]*ip\s+address

# No dedicated management VLAN
(?i)interface\s+Vlan1\s*\n[^!]*ip\s+address\s+\d+\.\d+\.\d+\.\d+
# Management IP on VLAN 1 indicates no dedicated management VLAN

# Voice VLAN configured (positive indicator)
(?i)switchport\s+voice\s+vlan\s+\d+

# Missing voice VLAN on phone ports
(?i)interface\s+\S+\s*\n[^!]*description\s+.*(?:phone|voip|ip.?phone)
# Verify: switchport voice vlan <id> present

# Unused port without dead-end VLAN
(?i)interface\s+\S+\s*\n\s*!
# Verify: assigned to parking VLAN and shut down
```

### Vendor Examples

**IOS/IOS-XE VLAN structure**:
```
! Management VLAN
vlan 10
 name MGMT-NETWORK

! Server VLANs
vlan 100
 name SERVERS-WEB
vlan 101
 name SERVERS-APP
vlan 102
 name SERVERS-DB

! Storage VLANs
vlan 200
 name STORAGE-ISCSI
vlan 201
 name STORAGE-NFS

! User VLANs
vlan 300
 name USERS-ENGINEERING
vlan 301
 name USERS-FINANCE
vlan 302
 name USERS-HR

! Voice VLAN
vlan 500
 name VOICE

! Guest VLAN
vlan 700
 name GUEST-INTERNET

! IoT VLAN
vlan 800
 name IOT-BUILDING

! DMZ VLAN
vlan 900
 name DMZ-PUBLIC

! Dead-end VLAN
vlan 999
 name PARKING-UNUSED
```

**Access port with voice VLAN**:
```
interface GigabitEthernet0/1
 description USER-PORT-ENG
 switchport mode access
 switchport access vlan 300
 switchport voice vlan 500
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
 ip dhcp snooping limit rate 15
```

---

## 2. Inter-VLAN Routing Control

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 2.1 | Inter-VLAN routing passes through a firewall or ACL-enforcing L3 device | **HIGH** |
| 2.2 | Not all VLANs can route to all other VLANs (segmentation is enforced, not just logical) | **HIGH** |
| 2.3 | Guest VLAN has no route to internal VLANs (internet-only egress) | **CRITICAL** |
| 2.4 | IoT VLAN has no route to user or server VLANs unless explicitly required | **HIGH** |
| 2.5 | Storage VLANs are not routable from user workstation VLANs | **CRITICAL** |
| 2.6 | Management VLAN is accessible only from authorized jump hosts or VPN | **HIGH** |
| 2.7 | DMZ-to-internal routing is restricted to specific ports and destinations | **HIGH** |
| 2.8 | Database VLAN is accessible only from application tier VLANs on specific ports | **HIGH** |
| 2.9 | Inter-VLAN ACLs are applied on the SVI (VLAN interface) for consistent enforcement | **MEDIUM** |
| 2.10 | Default route for guest/IoT segments points only to internet gateway, not internal core | **HIGH** |

### What to Look For

```regex
# Inter-VLAN routing without ACL
(?i)interface\s+Vlan\d+\s*\n[^!]*ip\s+address\s+\d+
# Verify: ip access-group <name> in/out exists on SVI

# Guest VLAN with route to internal
(?i)interface\s+Vlan(7\d\d)\s*\n[^!]*ip\s+address
# Verify: ACL blocks all internal destinations, only permits internet

# Storage VLAN routable from user segments
(?i)interface\s+Vlan(2\d\d)\s*\n[^!]*ip\s+address
# Verify: ACL restricts access to storage initiators only

# SVIs without ACLs
(?i)interface\s+Vlan\d+\s*\n(?:(?!\s*ip\s+access-group)[^!])*!
```

### Vendor Examples

**IOS/IOS-XE inter-VLAN ACL on SVI**:
```
! Guest VLAN - internet only, no internal access
ip access-list extended ACL-GUEST-IN
 remark Allow DHCP
 permit udp any any eq 67
 permit udp any any eq 68
 remark Allow DNS
 permit udp any host 10.0.0.53 eq 53
 remark Block RFC1918 destinations
 deny ip any 10.0.0.0 0.255.255.255 log
 deny ip any 172.16.0.0 0.15.255.255 log
 deny ip any 192.168.0.0 0.0.255.255 log
 remark Allow internet
 permit ip any any

interface Vlan700
 description GUEST-INTERNET
 ip address 10.7.0.1 255.255.255.0
 ip access-group ACL-GUEST-IN in
 ip helper-address 10.0.0.53
```

**Database VLAN restricted access**:
```
ip access-list extended ACL-DB-IN
 remark Allow app servers on MySQL port
 permit tcp 10.1.1.0 0.0.0.255 any eq 3306
 remark Allow monitoring
 permit tcp 10.0.0.0 0.0.0.255 any eq 9100
 remark Block everything else
 deny ip any any log

interface Vlan102
 description SERVERS-DB
 ip address 10.1.2.1 255.255.255.0
 ip access-group ACL-DB-IN in
```

**JunOS inter-VLAN firewall filter**:
```
firewall {
    family inet {
        filter GUEST-RESTRICT {
            term ALLOW-DHCP {
                from {
                    protocol udp;
                    destination-port [67 68];
                }
                then accept;
            }
            term ALLOW-DNS {
                from {
                    protocol udp;
                    destination-address 10.0.0.53/32;
                    destination-port domain;
                }
                then accept;
            }
            term BLOCK-INTERNAL {
                from {
                    destination-address {
                        10.0.0.0/8;
                        172.16.0.0/12;
                        192.168.0.0/16;
                    }
                }
                then {
                    discard;
                    log;
                    count blocked-to-internal;
                }
            }
            term ALLOW-INTERNET {
                then accept;
            }
        }
    }
}

interfaces {
    irb {
        unit 700 {
            description "GUEST-INTERNET";
            family inet {
                filter {
                    input GUEST-RESTRICT;
                }
                address 10.7.0.1/24;
            }
        }
    }
}
```

---

## 3. Native VLAN Security

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 3.1 | Native VLAN is changed from VLAN 1 to a dedicated, unused VLAN on all trunks | **HIGH** |
| 3.2 | Native VLAN is consistent on both ends of every trunk link | **HIGH** |
| 3.3 | Native VLAN tagging is enabled (802.1Q tag native VLAN frames) | **MEDIUM** |
| 3.4 | VLAN 1 has no SVI (no IP address assigned) in production | **HIGH** |
| 3.5 | Native VLAN is not used for any user or server traffic | **HIGH** |
| 3.6 | VLAN hopping via double-tagging is mitigated by native VLAN separation | **HIGH** |
| 3.7 | DTP is disabled to prevent VLAN hopping via trunk negotiation | **HIGH** |

### What to Look For

```regex
# Default native VLAN (VLAN 1)
(?i)interface\s+\S+\s*\n[^!]*switchport\s+mode\s+trunk\s*\n(?:(?!native\s+vlan)[^!])*!
# Missing: switchport trunk native vlan <non-1>

# Native VLAN 1 explicitly set
(?i)switchport\s+trunk\s+native\s+vlan\s+1\b

# VLAN 1 with IP address
(?i)interface\s+Vlan1\s*\n[^!]*ip\s+address\s+\d+

# Missing native VLAN tagging
(?i)vlan\s+dot1q\s+tag\s+native
# Should be present globally for 802.1Q native tagging
```

### Vendor Examples

**IOS/IOS-XE native VLAN hardening**:
```
! Global native VLAN tagging:
vlan dot1q tag native

! Per-trunk native VLAN:
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 999
 switchport nonegotiate

! Remove IP from VLAN 1:
interface Vlan1
 no ip address
 shutdown
```

**NX-OS native VLAN hardening**:
```
vlan dot1q tag native

interface Ethernet1/1
  switchport mode trunk
  switchport trunk native vlan 999
```

---

## 4. Private VLANs (PVLANs)

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 4.1 | Private VLANs are used in DMZ and multi-tenant environments to prevent lateral movement | **HIGH** |
| 4.2 | Isolated ports cannot communicate with other isolated or community ports | **HIGH** |
| 4.3 | Community ports can only communicate within their community and with promiscuous ports | **MEDIUM** |
| 4.4 | Promiscuous port is the gateway/firewall interface (upstream) | **MEDIUM** |
| 4.5 | PVLAN mappings are documented and match the intended isolation model | **MEDIUM** |
| 4.6 | PVLAN edge (protected ports) is used on access switches where full PVLAN is unsupported | **MEDIUM** |
| 4.7 | Proxy ARP is disabled on routed PVLAN interfaces to prevent information leakage | **HIGH** |

### What to Look For

```regex
# Private VLAN configuration (positive indicator)
(?i)private-vlan\s+(primary|isolated|community)

# PVLAN association
(?i)private-vlan\s+association\s+\d+

# PVLAN mapping on SVI
(?i)private-vlan\s+mapping\s+

# Missing proxy ARP disable on PVLAN SVIs
(?i)interface\s+Vlan\d+\s*\n[^!]*private-vlan\s+mapping
# Verify: no ip proxy-arp exists

# Protected port (simple peer isolation)
(?i)switchport\s+protected
```

### Vendor Examples

**IOS/IOS-XE Private VLAN (DMZ isolation)**:
```
! Primary VLAN for DMZ
vlan 900
 private-vlan primary
 private-vlan association 901,902

! Isolated VLAN (servers cannot talk to each other)
vlan 901
 private-vlan isolated

! Community VLAN (group of related servers can talk)
vlan 902
 private-vlan community

! Promiscuous port (uplink to firewall)
interface GigabitEthernet0/1
 switchport mode private-vlan promiscuous
 switchport private-vlan mapping 900 901,902

! Isolated port (individual DMZ server)
interface GigabitEthernet0/2
 switchport mode private-vlan host
 switchport private-vlan host-association 900 901

! Community port (clustered DMZ servers)
interface GigabitEthernet0/3
 switchport mode private-vlan host
 switchport private-vlan host-association 900 902

! SVI with PVLAN mapping
interface Vlan900
 ip address 10.9.0.1 255.255.255.0
 private-vlan mapping 901,902
 no ip proxy-arp
```

**NX-OS Private VLAN**:
```
feature private-vlan

vlan 900
  private-vlan primary
  private-vlan association 901

vlan 901
  private-vlan isolated

interface Ethernet1/1
  switchport mode private-vlan promiscuous
  switchport private-vlan mapping 900 901

interface Ethernet1/2
  switchport mode private-vlan host
  switchport private-vlan host-association 900 901
```

**Simple port isolation (protected ports)**:
```
! IOS/IOS-XE - ports in the same VLAN cannot communicate
interface range GigabitEthernet0/2-10
 switchport mode access
 switchport access vlan 900
 switchport protected
```

---

## 5. Micro-Segmentation

### 5.1 Cisco TrustSec / Security Group Tags (SGT)

| # | Checkpoint | Severity |
|---|---|---|
| 5.1.1 | SGT/TrustSec is used for identity-based segmentation beyond VLAN boundaries | **MEDIUM** |
| 5.1.2 | Security Group ACLs (SGACLs) enforce policy based on source and destination SGTs | **HIGH** |
| 5.1.3 | SGT classification is done at the access layer (closest to the endpoint) | **MEDIUM** |
| 5.1.4 | ISE (Identity Services Engine) is the authoritative source for SGT policy | **MEDIUM** |
| 5.1.5 | SGACL enforcement is verified on all transit devices in the data path | **HIGH** |
| 5.1.6 | SXP (SGT Exchange Protocol) is used where inline tagging is not supported | **MEDIUM** |
| 5.1.7 | SGT assignments are logged and auditable | **MEDIUM** |
| 5.1.8 | Default SGACL policy is deny (unknown SGTs cannot communicate freely) | **HIGH** |
| 5.1.9 | CTS (Cisco TrustSec) credentials are configured for device authentication | **HIGH** |
| 5.1.10 | IP-to-SGT mappings are consistent across the network | **HIGH** |

### What to Look For

```regex
# TrustSec / CTS configuration (positive indicator)
(?i)cts\s+(authorization|credentials|role-based)
(?i)cts\s+role-based\s+enforcement

# SGT assignment
(?i)cts\s+role-based\s+sgt-map\s+\S+\s+sgt\s+\d+

# SGACL policy
(?i)cts\s+role-based\s+permissions\s+

# SXP configuration
(?i)cts\s+sxp\s+

# Missing enforcement
(?i)cts\s+role-based\s+enforcement\s*$
# Verify: both vlan-list and interface enforcement configured
```

### Vendor Examples

**IOS/IOS-XE TrustSec**:
```
! Enable CTS
cts authorization list CTS-AUTH
cts role-based enforcement
cts role-based enforcement vlan-list all

! Static SGT mapping (for devices not doing 802.1X)
cts role-based sgt-map 10.1.1.0/24 sgt 10
cts role-based sgt-map 10.1.2.0/24 sgt 20

! SGACL definition
cts role-based permissions from 10 to 20 SGACL-USER-TO-SERVER

ip access-list role-based SGACL-USER-TO-SERVER
 permit tcp any any eq 443
 permit tcp any any eq 80
 permit icmp any any echo
 deny ip any any log

! SXP for devices without inline tagging
cts sxp enable
cts sxp default source-ip 10.0.0.1
cts sxp connection peer 10.0.0.2 password default mode local listener
```

### 5.2 VMware NSX Micro-Segmentation

| # | Checkpoint | Severity |
|---|---|---|
| 5.2.1 | NSX distributed firewall (DFW) rules enforce east-west segmentation at the hypervisor level | **HIGH** |
| 5.2.2 | DFW default rule is set to deny (not allow) | **CRITICAL** |
| 5.2.3 | Security groups are used for dynamic policy application based on VM attributes | **MEDIUM** |
| 5.2.4 | DFW rules are ordered by specificity (most specific first, default deny last) | **HIGH** |
| 5.2.5 | DFW applied-to field is scoped appropriately (not applied to all VMs unnecessarily) | **MEDIUM** |
| 5.2.6 | Emergency allow rule exists but is disabled for break-glass scenarios | **MEDIUM** |
| 5.2.7 | DFW rule logging is enabled for denied traffic | **MEDIUM** |
| 5.2.8 | Exclusion list is minimized and documented | **HIGH** |
| 5.2.9 | Micro-segmentation policies are version-controlled and change-managed | **MEDIUM** |
| 5.2.10 | Network flow monitoring (NSX Intelligence or equivalent) validates policy effectiveness | **MEDIUM** |

### What to Look For

```regex
# NSX DFW default rule allow (dangerous)
(?i)(default.?rule|section\s+default)\s*.*\b(allow|permit)\b
# Should be: deny/drop/reject

# NSX security group
(?i)(security.?group|ns.?group|nsgroup)

# DFW rule with action allow any-any
(?i)(source|src)\s*[:=]\s*(any|all)\s*.*(destination|dst)\s*[:=]\s*(any|all)\s*.*action\s*[:=]\s*(allow|permit)

# Exclusion list entries
(?i)exclusion.?list
# Verify: minimal entries with documented justification
```

### 5.3 Cloud-Native Micro-Segmentation

| # | Checkpoint | Severity |
|---|---|---|
| 5.3.1 | Security groups (AWS) or NSGs (Azure) enforce per-workload segmentation | **HIGH** |
| 5.3.2 | Security groups follow least privilege (no `0.0.0.0/0` on non-public ports) | **CRITICAL** |
| 5.3.3 | Security group rules reference other security groups (not CIDR blocks) where possible | **MEDIUM** |
| 5.3.4 | Network ACLs provide subnet-level defense in depth (in addition to security groups) | **MEDIUM** |
| 5.3.5 | VPC flow logs are enabled for all subnets | **MEDIUM** |
| 5.3.6 | Service mesh (Istio, Linkerd, Consul Connect) enforces mTLS and authorization policies | **MEDIUM** |
| 5.3.7 | Kubernetes Network Policies restrict pod-to-pod communication | **HIGH** |
| 5.3.8 | Kubernetes default-deny network policy is applied per namespace | **HIGH** |

### What to Look For

```regex
# AWS security group open to world on non-standard ports
(?i)ingress\s*\{[^}]*cidr_blocks\s*=\s*\["0\.0\.0\.0/0"\][^}]*from_port\s*=\s*(22|3389|3306|5432)
(?i)ingress\s*\{[^}]*from_port\s*=\s*(22|3389|3306|5432)[^}]*cidr_blocks\s*=\s*\["0\.0\.0\.0/0"\]

# Kubernetes missing network policy
(?i)kind:\s+Namespace
# Verify: corresponding NetworkPolicy with default-deny exists

# Kubernetes default-deny network policy (positive indicator)
(?i)kind:\s+NetworkPolicy[^-]*policyTypes:\s*\n\s*-\s*Ingress\s*\n\s*-\s*Egress
```

### Vendor Examples

**Kubernetes default-deny network policy**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: production
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

**Kubernetes allow specific traffic**:
```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-web-to-api
  namespace: production
spec:
  podSelector:
    matchLabels:
      app: api
  policyTypes:
    - Ingress
  ingress:
    - from:
        - podSelector:
            matchLabels:
              app: web
      ports:
        - protocol: TCP
          port: 8080
```

**AWS security group (Terraform) -- proper segmentation**:
```hcl
resource "aws_security_group" "app" {
  name_prefix = "app-"
  vpc_id      = var.vpc_id

  ingress {
    from_port       = 8080
    to_port         = 8080
    protocol        = "tcp"
    security_groups = [aws_security_group.alb.id]
    description     = "Allow traffic from ALB only"
  }

  egress {
    from_port       = 5432
    to_port         = 5432
    protocol        = "tcp"
    security_groups = [aws_security_group.database.id]
    description     = "Allow outbound to database only"
  }
}
```

---

## 6. Out-of-Band Management

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 6.1 | Management traffic uses a physically or logically separate network from production data | **HIGH** |
| 6.2 | Out-of-band (OOB) management network is not routable from user or server VLANs | **HIGH** |
| 6.3 | Console servers provide serial console access without relying on the data network | **MEDIUM** |
| 6.4 | IPMI/iDRAC/iLO/CIMC interfaces are on the management network, never on production VLANs | **CRITICAL** |
| 6.5 | Management network has its own default gateway and does not traverse production firewalls | **MEDIUM** |
| 6.6 | Jump hosts / bastion hosts are the only entry point to the management network | **HIGH** |
| 6.7 | Jump hosts require MFA and are hardened (no direct internet access, minimal software) | **HIGH** |
| 6.8 | Management VRF (VRF-lite) is used to isolate management routing from production | **MEDIUM** |
| 6.9 | SSH keys for management access are centrally managed and rotated | **MEDIUM** |
| 6.10 | Management network access is logged and auditable | **MEDIUM** |

### What to Look For

```regex
# IPMI/BMC on production VLAN
(?i)(ipmi|idrac|ilo|cimc|bmc)\s*.*\b(vlan\s+[1-9]\d{2,3}|production|user|server)
# BMC interfaces should be on management VLAN only

# Management VRF (positive indicator)
(?i)vrf\s+(definition\s+|forwarding\s+)?(MGMT|MANAGEMENT|OOB)

# Device management interface on production VRF
(?i)interface\s+\S+\s*\n[^!]*description\s+.*(?:mgmt|management)\s*\n(?:(?!vrf)[^!])*!
# Verify: vrf forwarding MGMT present

# Missing management ACL
(?i)interface\s+\S+\s*\n[^!]*description\s+.*(?:mgmt|management)
# Verify: ip access-group restricting management access
```

### Vendor Examples

**IOS/IOS-XE management VRF**:
```
vrf definition MGMT
 address-family ipv4
 exit-address-family

interface GigabitEthernet0/0
 description MGMT-NETWORK
 vrf forwarding MGMT
 ip address 10.0.0.1 255.255.255.0

! Management services bound to VRF:
ip route vrf MGMT 0.0.0.0 0.0.0.0 10.0.0.254

ip ssh source-interface GigabitEthernet0/0
snmp-server source-interface informs GigabitEthernet0/0

logging source-interface GigabitEthernet0/0 vrf MGMT
logging host 10.0.0.50 vrf MGMT

ntp source GigabitEthernet0/0
ntp server vrf MGMT 10.0.0.51
```

**NX-OS management VRF**:
```
! NX-OS uses VRF management by default for mgmt0
interface mgmt0
  vrf member management
  ip address 10.0.0.1/24

vrf context management
  ip route 0.0.0.0/0 10.0.0.254
```

---

## 7. PCI-DSS Segmentation Requirements

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 7.1 | Cardholder Data Environment (CDE) is isolated in a dedicated VLAN/subnet | **CRITICAL** |
| 7.2 | All traffic into and out of the CDE traverses a firewall with explicit rules | **CRITICAL** |
| 7.3 | Network segmentation reduces PCI scope (non-CDE systems cannot reach CDE) | **CRITICAL** |
| 7.4 | Segmentation controls are tested at least annually (PCI-DSS Req 11.3.4) | **HIGH** |
| 7.5 | Only necessary ports and protocols are permitted into the CDE | **HIGH** |
| 7.6 | Administrative access to CDE systems requires MFA (PCI-DSS Req 8.4.1) | **HIGH** |
| 7.7 | Wireless networks are segmented from the CDE or wireless access to CDE is prohibited | **HIGH** |
| 7.8 | Third-party/vendor access to CDE is through a controlled, monitored path | **HIGH** |
| 7.9 | Network diagrams document all connections into and out of the CDE | **HIGH** |
| 7.10 | CDE segmentation extends to cloud environments (VPC, security groups) | **HIGH** |
| 7.11 | PCI segment is monitored for unauthorized traffic flows (IDS/IPS or flow analysis) | **HIGH** |
| 7.12 | Connected-to systems (systems with a path to CDE) are identified and in scope | **HIGH** |

### What to Look For

```regex
# CDE VLAN or segment identification
(?i)(cde|cardholder|pci|payment)\s*(vlan|segment|subnet|network|zone)

# Firewall rules for CDE
(?i)(cde|cardholder|pci|payment)\s*
# Verify: firewall rules explicitly define permitted traffic

# Broad access to CDE segments
(?i)permit\s+(ip|tcp|udp)\s+(any|0\.0\.0\.0)\s+.*\b(cde|cardholder|pci|payment)\b
# Any-to-CDE should not exist

# Wireless-to-CDE path
(?i)(wireless|wifi|wlan|ssid)\s*.*\b(cde|cardholder|pci|payment)\b
# Wireless should be segmented from CDE
```

### Reference Architecture

```
                 Internet
                    |
              [Perimeter FW]
                    |
              [DMZ - VLAN 900]
              Web Servers (PCI)
                    |
              [Internal FW]
               /         \
     [CDE - VLAN 150]   [Non-CDE VLANs]
     App + DB Servers    User, Guest, IoT
     Payment Processing  (Cannot reach CDE)
              |
     [Mgmt FW / Jump Host]
              |
     [MGMT - VLAN 10]
     Admin Access (MFA)
```

### PCI-DSS v4.0 Relevant Requirements

| Requirement | Description | Segmentation Impact |
|-------------|-------------|---------------------|
| 1.2 | Network security controls configured and maintained | Firewall rules for CDE boundaries |
| 1.3 | Network access to/from CDE restricted | Inter-VLAN routing controls |
| 1.4 | Network connections between trusted and untrusted networks controlled | DMZ and CDE isolation |
| 2.2 | System components configured and managed securely | Segmentation device hardening |
| 8.4 | MFA for all access into the CDE | Management access controls |
| 11.3.4 | Segmentation controls tested at least annually | Penetration testing of boundaries |
| 11.4.1 | External and internal penetration testing | Validate segmentation effectiveness |

---

## 8. HIPAA Segmentation Requirements

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 8.1 | ePHI (electronic Protected Health Information) systems are in dedicated, isolated network segments | **CRITICAL** |
| 8.2 | Access to ePHI segments requires authentication and is limited to authorized personnel and systems | **CRITICAL** |
| 8.3 | Network segmentation supports the minimum necessary standard (164.502(b)) | **HIGH** |
| 8.4 | Audit logging captures all access attempts to ePHI network segments | **HIGH** |
| 8.5 | Medical device (IoMT) networks are isolated from ePHI and general user networks | **HIGH** |
| 8.6 | Guest and patient WiFi networks cannot reach ePHI or clinical systems | **CRITICAL** |
| 8.7 | Business Associate connections are through controlled, encrypted paths | **HIGH** |
| 8.8 | Emergency access procedures exist for ePHI segments (break-glass) | **HIGH** |
| 8.9 | Network segmentation design is documented in the HIPAA Security Rule risk assessment | **HIGH** |
| 8.10 | VPN or encrypted tunnel is used for any ePHI traversing untrusted networks | **CRITICAL** |

### What to Look For

```regex
# ePHI segment identification
(?i)(ephi|phi|hipaa|clinical|ehr|emr|health.?record|medical.?record)\s*(vlan|segment|subnet)

# Broad access to ePHI segments
(?i)permit\s+(ip|tcp|udp)\s+(any|0\.0\.0\.0)\s+.*\b(ephi|phi|clinical|ehr)\b

# Medical device VLAN
(?i)(medical|biomedical|iomt|clinical.?device)\s*(vlan|segment)
# Verify: isolated from ePHI data and user networks

# Guest WiFi with path to clinical
(?i)(guest|patient|visitor)\s*(wifi|wireless|ssid)
# Verify: cannot reach clinical or ePHI segments
```

### HIPAA Technical Safeguard References

| Safeguard | Section | Segmentation Requirement |
|-----------|---------|--------------------------|
| Access Control | 164.312(a)(1) | Unique user ID, emergency access, automatic logoff, encryption |
| Audit Controls | 164.312(b) | Record and examine activity in systems with ePHI |
| Integrity | 164.312(c)(1) | Protect ePHI from improper alteration or destruction |
| Transmission Security | 164.312(e)(1) | Encrypt ePHI in transit, integrity controls |

---

## 9. East-West Traffic Filtering

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 9.1 | East-west (lateral) traffic between server VLANs is filtered, not just north-south | **HIGH** |
| 9.2 | Firewall or ACL controls exist between application tiers (web-to-app, app-to-db) | **HIGH** |
| 9.3 | Micro-segmentation is deployed within the data center for workload isolation | **MEDIUM** |
| 9.4 | Network flow monitoring identifies unexpected east-west traffic patterns | **MEDIUM** |
| 9.5 | Lateral movement between compromised hosts is detectable and preventable | **HIGH** |
| 9.6 | Same-VLAN (intra-VLAN) traffic is controlled for high-security segments | **MEDIUM** |
| 9.7 | Service mesh or host-based firewalls supplement network-level controls | **MEDIUM** |
| 9.8 | East-west firewall rules follow least privilege (only required ports between tiers) | **HIGH** |
| 9.9 | DNS, NTP, and syslog traffic from servers is directed to authorized infrastructure only | **MEDIUM** |
| 9.10 | Servers cannot initiate outbound connections to the internet without justification | **HIGH** |

### What to Look For

```regex
# East-west firewall rules (positive indicator)
(?i)(east.?west|lateral|inter.?tier|server.?to.?server|tier.?to.?tier)

# Missing east-west filtering
(?i)interface\s+Vlan(1\d\d)\s*\n[^!]*ip\s+address
# Server VLANs (100-199 range) should have ACLs between tiers

# Servers with unrestricted internet access
(?i)permit\s+ip\s+(10\.\d+\.\d+\.\d+|172\.(1[6-9]|2\d|3[01])\.\d+\.\d+)\s+\S+\s+any
# Server subnets should not have blanket internet access

# Intra-VLAN filtering
(?i)(pvlan|private.?vlan|protected|peer.?isolation|micro.?segment)
# Positive indicators for intra-VLAN controls
```

### Vendor Examples

**East-west ACL between app tiers (IOS)**:
```
! Web tier to App tier
ip access-list extended ACL-WEB-TO-APP
 permit tcp 10.1.0.0 0.0.0.255 10.1.1.0 0.0.0.255 eq 8080
 permit tcp 10.1.0.0 0.0.0.255 10.1.1.0 0.0.0.255 eq 8443
 deny ip any any log

! App tier to Database tier
ip access-list extended ACL-APP-TO-DB
 permit tcp 10.1.1.0 0.0.0.255 10.1.2.0 0.0.0.255 eq 5432
 deny ip any any log

! Database tier - no initiation to other tiers
ip access-list extended ACL-DB-EGRESS
 permit tcp 10.1.2.0 0.0.0.255 10.0.0.0 0.0.0.255 eq 514
 permit udp 10.1.2.0 0.0.0.255 10.0.0.0 0.0.0.255 eq 123
 deny ip any any log
```

**NSX-T distributed firewall east-west rules (API)**:
```json
{
  "display_name": "Web-to-App-Only",
  "category": "Application",
  "rules": [
    {
      "display_name": "Allow-HTTPS-Web-to-App",
      "source_groups": ["/infra/domains/default/groups/Web-Servers"],
      "destination_groups": ["/infra/domains/default/groups/App-Servers"],
      "services": ["/infra/services/HTTPS"],
      "action": "ALLOW",
      "logged": true
    },
    {
      "display_name": "Deny-All-East-West",
      "source_groups": ["ANY"],
      "destination_groups": ["ANY"],
      "services": ["ANY"],
      "action": "DROP",
      "logged": true
    }
  ]
}
```

---

## 10. Trust Zone Architecture

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 10.1 | Network is divided into explicit trust zones with documented trust levels | **HIGH** |
| 10.2 | Traffic between trust zones traverses a stateful firewall | **HIGH** |
| 10.3 | Higher-trust zones can initiate connections to lower-trust zones (not the reverse) | **HIGH** |
| 10.4 | DMZ sits between untrusted (internet) and trusted (internal) zones | **HIGH** |
| 10.5 | Zero-trust principles are applied: verify explicitly, least privilege access, assume breach | **MEDIUM** |
| 10.6 | Trust zone policy is enforced at multiple layers (network, host, application) | **MEDIUM** |
| 10.7 | Trust zone boundaries are monitored with IDS/IPS or NDR (Network Detection and Response) | **MEDIUM** |
| 10.8 | Trust zone design accounts for insider threats (not just perimeter defense) | **HIGH** |
| 10.9 | Service accounts and application credentials follow trust zone boundaries | **HIGH** |
| 10.10 | Trust zone transitions are logged with source, destination, protocol, and action | **MEDIUM** |

### Trust Zone Reference Model

| Zone | Trust Level | Contains | Inbound From | Outbound To |
|------|-------------|----------|--------------|-------------|
| Internet | Untrusted | External users, attackers | N/A | DMZ only |
| DMZ | Low | Public web servers, API gateways, reverse proxies | Internet (HTTP/S) | Internal (specific APIs) |
| Internal/User | Medium | User workstations, printers, phones | VPN, wireless | Internet, DMZ, some servers |
| Server/App | Medium-High | Application servers, middleware | DMZ (reverse proxy), users | Database, storage, internet (limited) |
| Database | High | Database servers, cache | App servers only | Backup, replication |
| Storage | High | SAN, NAS, backup targets | Servers (iSCSI/NFS) | Replication targets |
| Management | High | Jump hosts, NMS, IPMI/BMC | VPN + MFA only | All zones (management protocols) |
| Restricted/PCI/PHI | Critical | CDE, ePHI systems | Authorized systems only | Minimal, monitored |

### What to Look For

```regex
# Zone definitions (positive indicators)
(?i)(zone|trust.?zone|security.?zone|zone.?pair)\s+

# Missing firewall between zones
(?i)interface\s+\S+\s*\n[^!]*description\s+.*(?:dmz|external|untrusted)
# Verify: traffic to/from this interface traverses a firewall

# Reverse connection from low-trust to high-trust
(?i)permit\s+tcp\s+.*\b(dmz|guest|iot|external)\b.*\b(internal|server|db|database|mgmt)\b
# Low-trust to high-trust should be minimized and explicitly justified

# Zero-trust indicators (positive)
(?i)(zero.?trust|never.?trust|always.?verify|mutual.?tls|mtls|identity.?aware)
```

---

## Review Procedure Summary

When evaluating network segmentation:

1. **Map VLANs to functions**: Verify each VLAN has a clear purpose (management, user, voice,
   guest, IoT, storage, DMZ, server tiers) and follows a documented scheme.
2. **Verify inter-VLAN controls**: Confirm ACLs or firewall rules restrict traffic between
   VLANs, especially between different trust levels.
3. **Check native VLAN security**: Verify native VLAN is not VLAN 1, is tagged, and is
   consistent across trunks.
4. **Assess isolation**: Confirm guest, IoT, and storage VLANs are properly isolated from
   internal resources and from each other.
5. **Evaluate micro-segmentation**: Check for TrustSec/SGT, NSX DFW, Kubernetes Network
   Policies, or equivalent workload-level controls.
6. **Verify OOB management**: Confirm management traffic is separated, BMC interfaces are
   isolated, and jump host access requires MFA.
7. **Check compliance segmentation**: For PCI and HIPAA environments, verify CDE/ePHI
   isolation meets regulatory requirements.
8. **Assess east-west filtering**: Verify server-to-server traffic is filtered, not just
   north-south perimeter traffic.
9. **Map trust zones**: Confirm explicit trust zone boundaries with firewalls between zones
   and appropriate directional controls.
10. **Classify each finding** using the severity levels in each section.
11. **Prioritize remediation**: CRITICAL findings (management on VLAN 1, no CDE isolation,
    BMC on production network, guest-to-ePHI path) first.

---

## Quick Reference: Severity Summary

| Severity | Segmentation Findings |
|----------|------------------------|
| CRITICAL | Management traffic on VLAN 1 or shared with production; storage VLANs routable from user segments; no CDE (PCI) or ePHI (HIPAA) network isolation; IPMI/iDRAC/iLO on production VLAN; guest WiFi can reach ePHI; NSX DFW default rule set to allow; cloud security group 0.0.0.0/0 on database ports |
| HIGH | No inter-VLAN ACLs between trust zones; guest VLAN can route to internal; IoT not isolated; management VLAN accessible without jump host; no east-west filtering between server tiers; private VLAN proxy-ARP not disabled; no Kubernetes network policy; SGACL enforcement missing; servers with unrestricted internet; DMZ-to-internal unrestricted; PCI segmentation untested; TrustSec default SGT policy allows all |
| MEDIUM | Missing native VLAN tagging; broadcast domain too large (> 500 hosts); no micro-segmentation in data center; no flow monitoring for east-west; management VRF not used; VLAN design undocumented; inter-VLAN ACLs on SVI inconsistent; NSX DFW logging not enabled; SXP not deployed where inline tagging unsupported; no IDS/IPS at zone boundaries |
| LOW | VLAN numbering scheme not documented; VLAN names not descriptive; missing descriptions on SVIs; PVLAN edge used instead of full PVLAN where supported |
| INFO | Well-designed trust zone architecture; comprehensive PVLAN deployment in DMZ; TrustSec/SGT with ISE integration; Kubernetes namespace isolation with default-deny; documented segmentation testing results |

---

## Standards References

- **NIST SP 800-125B** - Secure Virtual Network Configuration for Virtual Machine (VM) Protection
- **NIST SP 800-41 Rev. 1** - Guidelines on Firewalls and Firewall Policy
- **NIST SP 800-53 Rev. 5** - SC-7 (Boundary Protection), SC-32 (Information System Partitioning)
- **NIST SP 800-207** - Zero Trust Architecture
- **PCI-DSS v4.0** - Requirements 1.2, 1.3, 1.4, 11.3.4, 11.4.1
- **HIPAA Security Rule** - 164.312(a)(1), 164.312(e)(1)
- **CIS Controls v8** - Control 12 (Network Infrastructure Management), Control 13 (Network Monitoring and Defense)
- **CIS Cisco IOS Benchmark** - VLAN and trunking security
- **IEEE 802.1Q** - VLAN Tagging
- **IEEE 802.1X** - Port-Based Network Access Control
- **RFC 4364** - BGP/MPLS IP Virtual Private Networks (VPNs) -- VRF concepts
- **RFC 5765** - Security Issues with Private VLANs
- **Cisco TrustSec Design Guide** - SGT classification and enforcement
- **VMware NSX-T Security Design Guide** - Distributed firewall and micro-segmentation
- **NIST Cybersecurity Framework** - PR.AC (Access Control), DE.CM (Continuous Monitoring)
