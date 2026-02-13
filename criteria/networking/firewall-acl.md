# Firewall and ACL Reference Checklist

Canonical reference for evaluating access control lists, zone-based firewall configurations,
control plane protection, and Layer 2 security mechanisms. Complements the inlined criteria
in `review-networking.md` with CCIE-equivalent depth covering ACL design principles,
anti-spoofing, and common dangerous permit patterns.

Covers: ACL design, object groups, zone-based firewall, CoPP, management plane ACLs,
uRPF, IP Source Guard, DHCP snooping, Dynamic ARP Inspection, and reflexive ACLs.

---

## 1. ACL Design Principles

### 1.1 Least Privilege and Ordering

| # | Checkpoint | Severity |
|---|---|---|
| 1.1.1 | ACLs follow the principle of least privilege: deny by default, permit only what is required | **HIGH** |
| 1.1.2 | Most specific entries precede more general entries (proper ordering prevents shadowed rules) | **HIGH** |
| 1.1.3 | Every ACL ends with an explicit deny-all with logging for visibility | **MEDIUM** |
| 1.1.4 | ACLs are applied in the correct direction (inbound vs. outbound) relative to traffic flow | **HIGH** |
| 1.1.5 | ACL hit counts are reviewed periodically to identify unused or shadowed rules | **LOW** |
| 1.1.6 | ACLs include remarks/descriptions explaining the purpose of each entry or group of entries | **LOW** |
| 1.1.7 | ACLs are version-controlled and change-managed (not edited ad hoc on devices) | **MEDIUM** |
| 1.1.8 | Duplicate or redundant ACL entries are removed to reduce processing overhead | **LOW** |

### What to Look For

```regex
# Permit-any-any rules (overly permissive)
(?i)permit\s+(ip|tcp|udp)\s+(any|0\.0\.0\.0\s+255\.255\.255\.255)\s+(any|0\.0\.0\.0\s+255\.255\.255\.255)

# Missing explicit deny at end
(?i)access-list\s+\d+
# Verify: ends with deny any any (or implicit deny is acknowledged)

# ACL without remarks
(?i)ip\s+access-list\s+extended\s+\S+\s*\n\s*(permit|deny)
# Check: remark lines before permit/deny entries
```

### 1.2 Named vs. Numbered ACLs

| # | Checkpoint | Severity |
|---|---|---|
| 1.2.1 | Named ACLs are used instead of numbered ACLs for readability and maintainability | **LOW** |
| 1.2.2 | ACL names follow a consistent naming convention indicating direction and purpose | **LOW** |
| 1.2.3 | Extended ACLs are used where source and destination filtering is needed (not standard ACLs) | **MEDIUM** |
| 1.2.4 | Sequence numbers are used in named ACLs to allow insertion of rules without rewriting | **LOW** |

### Naming Convention Examples

```
! Good naming convention:
ip access-list extended ACL-WAN-INBOUND
ip access-list extended ACL-DMZ-TO-INTERNAL
ip access-list extended ACL-MGMT-ACCESS

! Poor naming (hard to understand purpose):
ip access-list extended ACL1
access-list 101 permit ...
```

### 1.3 Object Groups

| # | Checkpoint | Severity |
|---|---|---|
| 1.3.1 | Object groups are used for managing large ACLs with multiple similar entries | **LOW** |
| 1.3.2 | Network and service object groups are named descriptively | **LOW** |
| 1.3.3 | Object groups are reused across ACLs to ensure consistency | **MEDIUM** |
| 1.3.4 | Object groups are updated centrally when network/service definitions change | **MEDIUM** |

### Vendor Examples

**IOS/IOS-XE object groups**:
```
object-group network INTERNAL-SERVERS
 host 10.1.1.10
 host 10.1.1.11
 10.1.2.0 255.255.255.0

object-group service WEB-SERVICES tcp
 eq 80
 eq 443
 eq 8443

ip access-list extended ACL-INBOUND
 permit object-group WEB-SERVICES any object-group INTERNAL-SERVERS
 deny ip any any log
```

**NX-OS object groups**:
```
object-group ip address INTERNAL-SERVERS
  host 10.1.1.10
  host 10.1.1.11
  10.1.2.0/24

object-group ip port WEB-PORTS
  eq 80
  eq 443

ip access-list ACL-INBOUND
  permit tcp any addrgroup INTERNAL-SERVERS portgroup WEB-PORTS
  deny ip any any log
```

---

## 2. Common Dangerous Permits

### 2.1 Rules That Should Always Be Flagged

| # | Pattern | Severity | Risk |
|---|---------|----------|------|
| 2.1.1 | `permit ip any any` | **CRITICAL** | Completely bypasses access control |
| 2.1.2 | `permit tcp any any` or `permit udp any any` | **CRITICAL** | Protocol-scoped but still permits all traffic |
| 2.1.3 | `permit tcp any any eq 22` from untrusted zone | **HIGH** | SSH open to untrusted sources |
| 2.1.4 | `permit tcp any any eq 23` (Telnet) anywhere | **CRITICAL** | Cleartext management protocol |
| 2.1.5 | `permit tcp any any eq 3389` from untrusted zone | **HIGH** | RDP open to untrusted sources |
| 2.1.6 | `permit udp any any eq 161` from untrusted zone | **HIGH** | SNMP open to untrusted sources |
| 2.1.7 | `permit udp any any eq 69` (TFTP) | **HIGH** | Unauthenticated file transfer |
| 2.1.8 | `permit tcp any any eq 21` (FTP) | **HIGH** | Cleartext file transfer with credentials |
| 2.1.9 | `permit ip any any` on a VTY line ACL | **CRITICAL** | Management access unrestricted |
| 2.1.10 | `permit tcp any any range 1-65535` | **CRITICAL** | Equivalent to permit all TCP |
| 2.1.11 | `permit icmp any any` without type restriction | **MEDIUM** | Allows all ICMP including redirects and unreachables |
| 2.1.12 | Source `0.0.0.0/0` or `any` toward management interfaces | **HIGH** | Management plane exposed to all sources |

### Detection Regex Patterns

```regex
# Permit any-any (all protocols)
(?i)permit\s+ip\s+any\s+any
(?i)permit\s+ip\s+any\s+0\.0\.0\.0\s+255\.255\.255\.255
(?i)permit\s+ip\s+0\.0\.0\.0\s+255\.255\.255\.255\s+any

# Permit all TCP or UDP
(?i)permit\s+(tcp|udp)\s+any\s+any\s*$
(?i)permit\s+(tcp|udp)\s+any\s+any\s+range\s+1\s+65535

# Dangerous services from any source
(?i)permit\s+tcp\s+any\s+\S+\s+eq\s+(22|23|3389|3306|5432|1433|1521|6379|27017)
(?i)permit\s+udp\s+any\s+\S+\s+eq\s+(161|162|69|514)

# Telnet permitted anywhere
(?i)permit\s+tcp\s+\S+\s+\S+\s+eq\s+23

# FTP permitted
(?i)permit\s+tcp\s+\S+\s+\S+\s+eq\s+(20|21)\b

# SNMP from any
(?i)permit\s+udp\s+any\s+\S+\s+eq\s+161

# Unrestricted VTY access
(?i)line\s+vty\s+\d+\s+\d+\s*\n(?:[^!]*\n)*?\s*access-class\s+\d+\s+in
# Then check: access-list <num> permit ip any any

# Source any toward management networks
(?i)permit\s+(ip|tcp|udp)\s+any\s+(10\.\d+\.\d+\.\d+|172\.(1[6-9]|2\d|3[01])\.\d+\.\d+|192\.168\.\d+\.\d+)\s+
```

### 2.2 Cleartext Protocol Detection

| # | Checkpoint | Severity |
|---|---|---|
| 2.2.1 | Telnet (TCP/23) is not permitted in any ACL; SSH should be used instead | **CRITICAL** |
| 2.2.2 | FTP (TCP/20-21) is not permitted; SFTP or SCP should be used | **HIGH** |
| 2.2.3 | HTTP (TCP/80) to management interfaces is replaced by HTTPS (TCP/443) | **HIGH** |
| 2.2.4 | SNMP v1/v2c (UDP/161) is not permitted from untrusted zones; SNMPv3 is preferred | **HIGH** |
| 2.2.5 | TFTP (UDP/69) is not used for device configuration transfer in production | **HIGH** |
| 2.2.6 | Syslog (UDP/514) is not sent across untrusted network segments without TLS | **MEDIUM** |
| 2.2.7 | rlogin/rsh/rexec (TCP/512-514) are never permitted | **CRITICAL** |

---

## 3. Zone-Based Firewall (ZBF)

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 3.1 | Zone-based firewall is used instead of interface-based ACLs on router-firewalls | **MEDIUM** |
| 3.2 | All interfaces are assigned to a zone (unzoned interfaces can bypass policy) | **CRITICAL** |
| 3.3 | Default inter-zone policy is deny (traffic between zones is blocked unless explicitly permitted) | **HIGH** |
| 3.4 | Self zone policies protect the router's control plane (SSH, SNMP, routing protocols) | **HIGH** |
| 3.5 | Zone-pair policies use class maps and policy maps for granular inspection | **MEDIUM** |
| 3.6 | Stateful inspection (`inspect`) is enabled for permitted inter-zone traffic | **HIGH** |
| 3.7 | Zone-pair policies are documented with traffic flow justification | **LOW** |
| 3.8 | Application-layer inspection (HTTP, SMTP, DNS) is enabled where appropriate | **MEDIUM** |
| 3.9 | Zone-based firewall logs denied traffic for security monitoring | **MEDIUM** |

### What to Look For

```regex
# Interface without zone membership
(?i)interface\s+\S+\s*\n(?:(?!\s*zone-member)[^!])*!
# Verify: zone-member security <zone> present

# Missing self-zone policy
(?i)zone-pair\s+security
# Verify: zone-pair with self zone as source or destination exists

# Pass action without inspect (stateless)
(?i)service-policy\s+type\s+inspect\s+\S+\s*\n[^!]*policy-map[^!]*class[^!]*pass\b
# Should use: inspect instead of pass for stateful tracking
```

### Vendor Example

**IOS/IOS-XE zone-based firewall**:
```
zone security INSIDE
zone security OUTSIDE
zone security DMZ

zone-pair security IN-TO-OUT source INSIDE destination OUTSIDE
 service-policy type inspect IN-TO-OUT-POLICY

zone-pair security OUT-TO-DMZ source OUTSIDE destination DMZ
 service-policy type inspect OUT-TO-DMZ-POLICY

zone-pair security IN-TO-SELF source INSIDE destination self
 service-policy type inspect MGMT-POLICY

! All interfaces assigned to zones:
interface GigabitEthernet0/0
 zone-member security INSIDE
interface GigabitEthernet0/1
 zone-member security OUTSIDE
interface GigabitEthernet0/2
 zone-member security DMZ

! Policy map with inspect:
policy-map type inspect IN-TO-OUT-POLICY
 class type inspect PERMITTED-TRAFFIC
  inspect
 class class-default
  drop log
```

---

## 4. Control Plane Policing (CoPP)

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 4.1 | CoPP is configured to protect the route processor from DoS attacks | **HIGH** |
| 4.2 | CoPP classifies traffic into categories: routing protocols, management, monitoring, undesirable | **MEDIUM** |
| 4.3 | Routing protocol traffic (BGP, OSPF, IS-IS, EIGRP) is permitted at appropriate rates | **HIGH** |
| 4.4 | Management traffic (SSH, SNMP, NTP) is rate-limited to prevent abuse | **MEDIUM** |
| 4.5 | Undesirable traffic (fragments, TTL-expired, options) is rate-limited or dropped | **MEDIUM** |
| 4.6 | ICMP to the control plane is rate-limited (not blocked entirely -- needed for PMTUD) | **MEDIUM** |
| 4.7 | CoPP counters are monitored for drops indicating attack or misconfiguration | **LOW** |
| 4.8 | ARP and broadcast traffic is rate-limited to the control plane | **MEDIUM** |

### What to Look For

```regex
# Missing CoPP
(?i)control-plane
# Verify: service-policy input exists under control-plane

# CoPP policy map
(?i)policy-map\s+\S*(copp|control|cp)\S*
# Verify: class maps and police actions defined

# NX-OS CoPP
(?i)copp\s+profile\s+(strict|moderate|lenient)
# strict is recommended for production
```

### Vendor Examples

**IOS/IOS-XE CoPP**:
```
ip access-list extended ACL-COPP-ROUTING
 permit tcp any any eq bgp
 permit tcp any eq bgp any
 permit ospf any any
 permit eigrp any any

ip access-list extended ACL-COPP-MANAGEMENT
 permit tcp 10.0.0.0 0.0.0.255 any eq 22
 permit udp 10.0.0.0 0.0.0.255 any eq 161

ip access-list extended ACL-COPP-UNDESIRABLE
 permit icmp any any fragments
 permit ip any any option any-options
 permit udp any any eq 19
 permit tcp any any eq 19

class-map match-all CM-COPP-ROUTING
 match access-group name ACL-COPP-ROUTING

class-map match-all CM-COPP-MANAGEMENT
 match access-group name ACL-COPP-MANAGEMENT

class-map match-all CM-COPP-UNDESIRABLE
 match access-group name ACL-COPP-UNDESIRABLE

policy-map PM-COPP
 class CM-COPP-ROUTING
  police 5000000 conform-action transmit exceed-action transmit
 class CM-COPP-MANAGEMENT
  police 500000 conform-action transmit exceed-action drop
 class CM-COPP-UNDESIRABLE
  police 32000 conform-action drop exceed-action drop
 class class-default
  police 250000 conform-action transmit exceed-action drop

control-plane
 service-policy input PM-COPP
```

**NX-OS CoPP**:
```
copp profile strict
! NX-OS ships with predefined CoPP profiles; strict is recommended
! Verify with: show copp profile
```

---

## 5. Management Plane ACLs

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 5.1 | VTY lines have an ACL restricting SSH access to management networks only | **CRITICAL** |
| 5.2 | HTTP/HTTPS server access is restricted to management networks | **HIGH** |
| 5.3 | SNMP access is restricted to NMS servers via ACL | **HIGH** |
| 5.4 | NTP access is restricted to authorized time sources | **MEDIUM** |
| 5.5 | Console port has a timeout configured (exec-timeout) | **MEDIUM** |
| 5.6 | VTY transport is restricted to SSH only (no Telnet) | **CRITICAL** |
| 5.7 | Auxiliary port is disabled if not used | **MEDIUM** |
| 5.8 | Management plane protection (MPP) is configured where supported | **MEDIUM** |

### What to Look For

```regex
# VTY without access-class
(?i)line\s+vty\s+\d+\s+\d+\s*\n(?:(?!\s*access-class)[^!])*!

# Telnet permitted on VTY
(?i)line\s+vty\s+\d+\s+\d+\s*\n[^!]*transport\s+input\s+(all|telnet)

# Missing exec-timeout on console
(?i)line\s+con\s+0\s*\n(?:(?!\s*exec-timeout)[^!])*!
(?i)exec-timeout\s+0\s+0
# exec-timeout 0 0 disables timeout (dangerous)

# SNMP without ACL
(?i)snmp-server\s+community\s+\S+\s+(RO|RW)\s*$
# Should be: snmp-server community <string> RO <acl-number-or-name>

# HTTP server without ACL
(?i)ip\s+http\s+server
# Verify: ip http access-class <acl> exists
```

### Vendor Examples

**IOS/IOS-XE management lockdown**:
```
! VTY restricted to SSH from management network:
ip access-list standard ACL-VTY-ACCESS
 permit 10.0.0.0 0.0.0.255
 deny any log

line vty 0 15
 access-class ACL-VTY-ACCESS in
 transport input ssh
 exec-timeout 10 0
 login local

! Console timeout:
line con 0
 exec-timeout 5 0
 login local

! Auxiliary port disabled:
line aux 0
 no exec
 transport input none
 transport output none

! SNMP restricted:
snmp-server community <string> RO ACL-SNMP-ACCESS
ip access-list standard ACL-SNMP-ACCESS
 permit 10.0.0.10
 permit 10.0.0.11
 deny any log

! HTTP server restricted or disabled:
no ip http server
ip http secure-server
ip http access-class 10
```

**JunOS management lockdown**:
```
system {
    login {
        class ADMIN {
            permissions all;
        }
        user admin {
            class ADMIN;
            authentication {
                encrypted-password "$6$...";
            }
        }
    }
    services {
        ssh {
            root-login deny;
            protocol-version v2;
        }
        netconf {
            ssh;
        }
    }
}

firewall {
    filter MGMT-FILTER {
        term ALLOW-SSH {
            from {
                source-address 10.0.0.0/24;
                protocol tcp;
                destination-port ssh;
            }
            then accept;
        }
        term ALLOW-SNMP {
            from {
                source-address 10.0.0.0/24;
                protocol udp;
                destination-port snmp;
            }
            then accept;
        }
        term DENY-ALL {
            then {
                discard;
                log;
            }
        }
    }
}
```

---

## 6. Unicast Reverse Path Forwarding (uRPF)

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 6.1 | uRPF is enabled on external-facing (untrusted) interfaces for anti-spoofing | **HIGH** |
| 6.2 | uRPF mode is appropriate: strict on single-homed, loose (or feasible-path) on multi-homed | **HIGH** |
| 6.3 | uRPF is NOT enabled on interfaces where asymmetric routing is expected (or use loose/feasible mode) | **HIGH** |
| 6.4 | uRPF ACL is configured to allow known asymmetric sources where needed | **MEDIUM** |
| 6.5 | uRPF is combined with other anti-spoofing measures at the edge | **MEDIUM** |

### What to Look For

```regex
# Missing uRPF on external interfaces
(?i)interface\s+\S+\s*\n[^!]*description\s+.*(?:WAN|INTERNET|UNTRUSTED|EXTERNAL|UPLINK)
# Verify: ip verify unicast source reachable-via exists

# uRPF strict on multi-homed interface (may drop legitimate traffic)
(?i)ip\s+verify\s+unicast\s+source\s+reachable-via\s+rx\b
# On multi-homed: should be "any" (loose) or have allow-self-ping / ACL

# uRPF configuration
(?i)ip\s+verify\s+unicast\s+source\s+reachable-via\s+(rx|any)
```

### Vendor Examples

**IOS/IOS-XE uRPF strict (single-homed edge)**:
```
interface GigabitEthernet0/0
 description WAN-UPLINK
 ip verify unicast source reachable-via rx
```

**IOS/IOS-XE uRPF loose (multi-homed)**:
```
interface GigabitEthernet0/0
 description TRANSIT-PROVIDER-A
 ip verify unicast source reachable-via any
```

**JunOS uRPF**:
```
interfaces {
    ge-0/0/0 {
        unit 0 {
            family inet {
                rpf-check;
            }
        }
    }
}
```

**NX-OS uRPF**:
```
interface Ethernet1/1
  ip verify unicast source reachable-via rx
```

---

## 7. DHCP Snooping

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 7.1 | DHCP snooping is enabled globally and on all access VLANs | **HIGH** |
| 7.2 | Only uplinks to DHCP servers or trusted infrastructure are marked as trusted ports | **HIGH** |
| 7.3 | Access ports (end-user facing) are untrusted (default) | **HIGH** |
| 7.4 | DHCP snooping rate limiting is configured on untrusted ports | **MEDIUM** |
| 7.5 | DHCP snooping database (binding table) is persisted to survive reboots | **MEDIUM** |
| 7.6 | Option 82 insertion is configured where required (relay scenarios) | **LOW** |
| 7.7 | DHCP snooping is enabled before IP Source Guard and DAI (prerequisite) | **HIGH** |

### What to Look For

```regex
# Missing DHCP snooping
(?i)ip\s+dhcp\s+snooping\s*$
(?i)ip\s+dhcp\s+snooping\s+vlan\s+

# Trusted port on access interface (should only be on uplinks)
(?i)interface\s+\S+\s*\n[^!]*switchport\s+mode\s+access\s*\n[^!]*ip\s+dhcp\s+snooping\s+trust

# Missing rate limit on untrusted ports
(?i)ip\s+dhcp\s+snooping\s+limit\s+rate
```

### Vendor Examples

**IOS/IOS-XE DHCP snooping**:
```
ip dhcp snooping
ip dhcp snooping vlan 10,20,30
ip dhcp snooping database flash:dhcp-snooping.db

! Trusted uplink:
interface GigabitEthernet0/1
 ip dhcp snooping trust

! Untrusted access port (default, but explicit rate-limit):
interface GigabitEthernet0/2
 ip dhcp snooping limit rate 15
```

**NX-OS DHCP snooping**:
```
feature dhcp
ip dhcp snooping
ip dhcp snooping vlan 10,20,30

interface Ethernet1/1
  ip dhcp snooping trust

interface Ethernet1/2
  ip dhcp snooping limit rate 15
```

---

## 8. Dynamic ARP Inspection (DAI)

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 8.1 | DAI is enabled on all access VLANs (requires DHCP snooping as prerequisite) | **HIGH** |
| 8.2 | Trusted ports are limited to uplinks and infrastructure ports | **HIGH** |
| 8.3 | DAI rate limiting is configured on untrusted ports to prevent ARP flooding | **MEDIUM** |
| 8.4 | ARP ACLs are configured for static IP hosts that do not use DHCP | **MEDIUM** |
| 8.5 | DAI validation checks include source MAC, destination MAC, and IP addresses | **MEDIUM** |
| 8.6 | DAI err-disable recovery is configured with a reasonable interval | **MEDIUM** |

### What to Look For

```regex
# Missing DAI
(?i)ip\s+arp\s+inspection\s+vlan\s+

# DAI validation checks
(?i)ip\s+arp\s+inspection\s+validate\s+(src-mac|dst-mac|ip)

# Missing err-disable recovery for DAI
(?i)errdisable\s+recovery\s+cause\s+arp-inspection
```

### Vendor Examples

**IOS/IOS-XE DAI**:
```
ip arp inspection vlan 10,20,30
ip arp inspection validate src-mac dst-mac ip

! Trusted uplink:
interface GigabitEthernet0/1
 ip arp inspection trust

! Untrusted access port with rate limit:
interface GigabitEthernet0/2
 ip arp inspection limit rate 15

! Err-disable recovery:
errdisable recovery cause arp-inspection
errdisable recovery interval 300

! Static host ARP ACL (for non-DHCP hosts):
arp access-list STATIC-ARP
 permit ip host 10.1.1.100 mac host 0000.1111.2222
ip arp inspection filter STATIC-ARP vlan 10
```

---

## 9. IP Source Guard

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 9.1 | IP Source Guard is enabled on all untrusted access ports | **HIGH** |
| 9.2 | IP Source Guard uses the DHCP snooping binding table for validation | **HIGH** |
| 9.3 | Static IP source bindings are configured for hosts with static IP addresses | **MEDIUM** |
| 9.4 | IP Source Guard is enabled after DHCP snooping is verified working | **HIGH** |
| 9.5 | IP Source Guard filters both IP and MAC where supported (ip verify source port-security) | **MEDIUM** |

### What to Look For

```regex
# IP Source Guard
(?i)ip\s+verify\s+source
(?i)ip\s+verify\s+source\s+port-security

# Static binding for non-DHCP hosts
(?i)ip\s+source\s+binding\s+\S+\s+vlan\s+\d+\s+\S+\s+interface
```

### Vendor Examples

**IOS/IOS-XE IP Source Guard**:
```
! On access ports (requires DHCP snooping):
interface GigabitEthernet0/2
 ip verify source

! With port-security (validates both IP and MAC):
interface GigabitEthernet0/3
 ip verify source port-security

! Static binding for server with static IP:
ip source binding 0000.1111.2222 vlan 10 10.1.1.100 interface GigabitEthernet0/4
```

---

## 10. Reflexive ACLs and Stateful Filtering

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 10.1 | Reflexive ACLs or CBAC are used for stateful return traffic on router-based firewalls | **HIGH** |
| 10.2 | Return traffic is not permitted via broad static ACL entries (e.g., permit established) | **HIGH** |
| 10.3 | Reflexive ACL timeout is configured to reasonable values | **MEDIUM** |
| 10.4 | `established` keyword is not used as a substitute for stateful inspection (TCP flags only, easily spoofed) | **HIGH** |

### What to Look For

```regex
# permit established (not truly stateful)
(?i)permit\s+tcp\s+any\s+any\s+established
# This only checks the ACK/RST flags; it does not track connections

# Reflexive ACL usage (positive indicator)
(?i)evaluate\s+\S+
(?i)reflect\s+\S+
```

### Vendor Examples

**IOS reflexive ACL (legacy)**:
```
ip access-list extended OUTBOUND
 permit tcp any any reflect TCP-SESSIONS
 permit udp any any reflect UDP-SESSIONS
 permit icmp any any reflect ICMP-SESSIONS

ip access-list extended INBOUND
 evaluate TCP-SESSIONS
 evaluate UDP-SESSIONS
 evaluate ICMP-SESSIONS
 deny ip any any log

interface GigabitEthernet0/0
 ip access-group INBOUND in
 ip access-group OUTBOUND out
```

**Preferred approach**: Use zone-based firewall (Section 3) instead of reflexive ACLs for
proper stateful inspection with application-layer awareness.

---

## 11. Platform-Specific Firewall Considerations

### 11.1 NX-OS (Nexus)

| # | Checkpoint | Severity |
|---|---|---|
| 11.1.1 | PACL (Port ACL), VACL (VLAN ACL), and RACL (Routed ACL) are applied at appropriate layers | **MEDIUM** |
| 11.1.2 | VACL redirect or capture actions are documented and justified | **MEDIUM** |
| 11.1.3 | ACL TCAM utilization is monitored (NX-OS has finite TCAM for ACLs) | **MEDIUM** |
| 11.1.4 | Statistics are enabled on ACLs for hit-count monitoring | **LOW** |

### 11.2 JunOS Firewall Filters

| # | Checkpoint | Severity |
|---|---|---|
| 11.2.1 | Firewall filters use `then accept` and `then discard` (not `reject` on transit traffic) | **MEDIUM** |
| 11.2.2 | Firewall filters applied to `lo0` protect the RE (Routing Engine) | **HIGH** |
| 11.2.3 | Policers are applied to protect against traffic storms | **MEDIUM** |
| 11.2.4 | Filter terms are ordered correctly (most specific first, default deny last) | **HIGH** |

### JunOS lo0 Protection Example

```
firewall {
    filter PROTECT-RE {
        term ALLOW-BGP {
            from {
                source-address 10.0.0.0/24;
                protocol tcp;
                destination-port bgp;
            }
            then accept;
        }
        term ALLOW-OSPF {
            from {
                protocol ospf;
            }
            then accept;
        }
        term ALLOW-SSH {
            from {
                source-address 10.0.0.0/24;
                protocol tcp;
                destination-port ssh;
            }
            then accept;
        }
        term ALLOW-ICMP {
            from {
                protocol icmp;
                icmp-type [ echo-reply echo-request time-exceeded unreachable ];
            }
            then {
                policer ICMP-POLICER;
                accept;
            }
        }
        term ALLOW-NTP {
            from {
                source-address 10.0.0.0/24;
                protocol udp;
                destination-port ntp;
            }
            then accept;
        }
        term DENY-ALL {
            then {
                discard;
                count DENIED-TO-RE;
                log;
            }
        }
    }
}

interfaces {
    lo0 {
        unit 0 {
            family inet {
                filter {
                    input PROTECT-RE;
                }
            }
        }
    }
}
```

### 11.3 EOS (Arista)

| # | Checkpoint | Severity |
|---|---|---|
| 11.3.1 | ACLs use counters for visibility (`ip access-list ... counters per-entry`) | **LOW** |
| 11.3.2 | Management ACL (`management api http-commands`, `management ssh`) restricts access appropriately | **HIGH** |
| 11.3.3 | Control-plane ACL is applied to protect the switch CPU | **HIGH** |

### EOS Management ACL Example

```
management ssh
   ip access-group MGMT-ACCESS in

ip access-list MGMT-ACCESS
   10 permit tcp 10.0.0.0/24 any eq ssh
   20 deny ip any any log
```

---

## Review Procedure Summary

When evaluating firewall and ACL configurations:

1. **Inventory all ACLs**: List every ACL and where it is applied (interface, VTY, SNMP, etc.).
2. **Check for dangerous permits**: Scan for `permit any any`, cleartext protocols, management
   access from untrusted sources using the detection regex patterns in Section 2.
3. **Validate direction and placement**: Confirm ACLs are applied inbound where filtering
   is most effective (close to the source).
4. **Assess management plane**: Verify VTY access, SNMP, HTTP, and NTP are restricted to
   management networks only.
5. **Check CoPP**: Confirm control plane policing protects the route processor.
6. **Evaluate L2 security**: Verify DHCP snooping, DAI, and IP Source Guard are enabled
   on access VLANs and ports.
7. **Verify anti-spoofing**: Confirm uRPF is enabled on external-facing interfaces.
8. **Zone-based firewall**: If router-based firewalling is in use, verify all interfaces
   are zoned and stateful inspection is enabled.
9. **Classify each finding** using the severity levels in each section.
10. **Prioritize remediation**: CRITICAL findings (unrestricted management access, permit
    any any, Telnet enabled) first, then HIGH, MEDIUM, LOW.

---

## Quick Reference: Severity Summary

| Severity | Firewall & ACL Findings |
|----------|-------------------------|
| CRITICAL | `permit ip any any` in production ACL; Telnet (TCP/23) or rlogin/rsh (TCP/512-514) permitted; unrestricted VTY access; unzoned interface in ZBF deployment; management access from any source |
| HIGH | Missing ACL on VTY lines; SNMP unrestricted; SSH/RDP from untrusted zones; FTP/TFTP permitted; no CoPP; no DHCP snooping; no DAI; no IP Source Guard; no uRPF on external interfaces; `established` used as stateful substitute; BPDU filtering on access ports |
| MEDIUM | Missing explicit deny-all with logging; ACLs not version-controlled; ICMP unrestricted; no DHCP snooping rate limit; no DAI validation checks; uRPF ACL not configured for asymmetric paths; CoPP undesirable class not policed; syslog over untrusted segments without TLS; NTP access unrestricted |
| LOW | Numbered ACLs instead of named; missing ACL remarks; no ACL hit-count review; inconsistent naming conventions; missing object groups; no CoPP counter monitoring |
| INFO | Well-structured zone-based firewall; proper use of object groups; comprehensive CoPP with all traffic classes; DHCP snooping + DAI + IP Source Guard all deployed together |

---

## Standards References

- **RFC 2827 / BCP 38** - Network Ingress Filtering: Defeating Denial of Service Attacks which employ IP Source Address Spoofing
- **RFC 3704 / BCP 84** - Ingress Filtering for Multihomed Networks
- **NIST SP 800-41 Rev. 1** - Guidelines on Firewalls and Firewall Policy
- **NIST SP 800-123** - Guide to General Server Security
- **CIS Cisco IOS Benchmark** - Sections on ACL, management plane, and CoPP
- **CIS Cisco NX-OS Benchmark** - Sections on PACL, VACL, CoPP
- **NSA Network Infrastructure Security Guide** - ACL and anti-spoofing recommendations
- **PCI-DSS v4.0 Requirement 1** - Install and Maintain Network Security Controls
- **MANRS** - Anti-spoofing (BCP 38/84) implementation
