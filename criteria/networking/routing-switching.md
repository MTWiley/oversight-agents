# Routing and Switching Reference Checklist

Canonical reference for evaluating routing protocol configuration, switching best practices,
and Layer 2/Layer 3 design across enterprise network infrastructure. Complements the inlined
criteria in `review-networking.md` with CCIE-equivalent depth, vendor-specific examples,
and standards references.

Covers: BGP, OSPF, IS-IS, EIGRP, static routing, STP, VTP, VLAN trunking, and
EtherChannel/port-channel configurations.

---

## 1. BGP (Border Gateway Protocol)

### 1.1 Peer Authentication

| # | Checkpoint | Severity |
|---|---|---|
| 1.1.1 | All eBGP sessions use TCP MD5 authentication (RFC 2385) or TCP-AO (RFC 5925) | **CRITICAL** |
| 1.1.2 | iBGP sessions use authentication in multi-tenant or shared infrastructure environments | **HIGH** |
| 1.1.3 | MD5 passwords are not stored in cleartext in configuration files | **HIGH** |
| 1.1.4 | BGP authentication keys are rotated on a defined schedule | **MEDIUM** |
| 1.1.5 | TCP-AO is preferred over MD5 where supported by both peers | **LOW** |

### What to Look For

```regex
# Missing BGP authentication (IOS/NX-OS)
(?i)neighbor\s+\S+\s+remote-as\s+\d+
# Verify: a corresponding "neighbor <ip> password" line exists

# Cleartext passwords in config
(?i)neighbor\s+\S+\s+password\s+0\s+
(?i)neighbor\s+\S+\s+password\s+[^7]\S+

# JunOS missing authentication
(?i)protocols\s*\{\s*bgp\s*\{[^}]*group\s+\S+\s*\{[^}]*neighbor\s+\S+
# Verify: authentication-key or authentication-algorithm present
```

### Vendor Examples

**IOS/IOS-XE** (correct):
```
router bgp 65001
 neighbor 10.0.0.1 remote-as 65002
 neighbor 10.0.0.1 password 7 <encrypted-string>
```

**NX-OS** (correct):
```
router bgp 65001
  neighbor 10.0.0.1
    remote-as 65002
    password 3 <encrypted-string>
```

**JunOS** (correct):
```
protocols {
    bgp {
        group EBGP-PEERS {
            neighbor 10.0.0.1 {
                peer-as 65002;
                authentication-key "$9$encrypted";
            }
        }
    }
}
```

**EOS** (correct):
```
router bgp 65001
   neighbor 10.0.0.1 remote-as 65002
   neighbor 10.0.0.1 password 7 <encrypted-string>
```

### 1.2 Max-Prefix Limits

| # | Checkpoint | Severity |
|---|---|---|
| 1.2.1 | All eBGP sessions have `maximum-prefix` configured | **HIGH** |
| 1.2.2 | Max-prefix threshold includes a warning percentage (typically 75-80%) | **MEDIUM** |
| 1.2.3 | Max-prefix action is set to `restart` with a reasonable interval, not indefinite teardown | **MEDIUM** |
| 1.2.4 | iBGP route reflector clients have max-prefix where route leaks could propagate | **MEDIUM** |
| 1.2.5 | Max-prefix values are appropriate for the peer type (transit, peer, customer) | **MEDIUM** |

### What to Look For

```regex
# Missing max-prefix (IOS/NX-OS)
(?i)neighbor\s+\S+\s+remote-as\s+\d+
# Verify: neighbor <ip> maximum-prefix <limit> exists

# JunOS missing prefix-limit
(?i)neighbor\s+\S+\s*\{[^}]*peer-as
# Verify: family inet unicast { prefix-limit { maximum } } exists
```

### Vendor Examples

**IOS/IOS-XE**:
```
neighbor 10.0.0.1 maximum-prefix 10000 80 restart 15
! 10000 max routes, warn at 80%, restart session after 15 minutes
```

**NX-OS**:
```
neighbor 10.0.0.1
  address-family ipv4 unicast
    maximum-prefix 10000 80 restart 15
```

**JunOS**:
```
family inet {
    unicast {
        prefix-limit {
            maximum 10000;
            teardown 80 idle-timeout 15;
        }
    }
}
```

### 1.3 Route Filtering

| # | Checkpoint | Severity |
|---|---|---|
| 1.3.1 | All eBGP sessions have explicit inbound prefix filters (prefix-lists or ORF) | **CRITICAL** |
| 1.3.2 | All eBGP sessions have explicit outbound prefix filters | **HIGH** |
| 1.3.3 | Default route is not accepted from peers unless explicitly intended (transit providers) | **HIGH** |
| 1.3.4 | Prefix filters are maintained and updated (IRR-based or RPKI validation) | **MEDIUM** |
| 1.3.5 | AS-path filters are applied to reject invalid or unexpected AS paths | **HIGH** |
| 1.3.6 | No eBGP session uses a bare `permit any` inbound without justification | **CRITICAL** |
| 1.3.7 | Route maps reference prefix-lists, not bare ACLs for prefix filtering | **LOW** |

### What to Look For

```regex
# eBGP neighbor without route-map or prefix-list inbound
(?i)neighbor\s+\S+\s+remote-as\s+(?!(?:\s*\d+\s*$))
# Missing: neighbor <ip> route-map <name> in
# Missing: neighbor <ip> prefix-list <name> in

# Overly permissive prefix-list
(?i)ip\s+prefix-list\s+\S+\s+seq\s+\d+\s+permit\s+0\.0\.0\.0/0\s+le\s+32

# Missing AS-path filter
(?i)neighbor\s+\S+\s+remote-as
# Verify: neighbor <ip> filter-list <num> in
```

### 1.4 BGP Communities

| # | Checkpoint | Severity |
|---|---|---|
| 1.4.1 | BGP communities are used for traffic engineering and policy application | **MEDIUM** |
| 1.4.2 | Well-known communities (NO_EXPORT, NO_ADVERTISE, NO_EXPORT_SUBCONFED) are used appropriately | **MEDIUM** |
| 1.4.3 | Community stripping is applied on eBGP ingress/egress where needed | **MEDIUM** |
| 1.4.4 | Extended communities or large communities (RFC 8092) are used for complex policies | **LOW** |
| 1.4.5 | `send-community` is enabled on neighbors where communities need to be propagated | **MEDIUM** |

### 1.5 Route Reflectors

| # | Checkpoint | Severity |
|---|---|---|
| 1.5.1 | Route reflector topology avoids single points of failure (redundant RRs per cluster) | **HIGH** |
| 1.5.2 | Cluster-ID is consistent across redundant route reflectors in the same cluster | **HIGH** |
| 1.5.3 | Route reflectors are not in the forwarding path to avoid suboptimal routing | **MEDIUM** |
| 1.5.4 | `next-hop-self` is configured where RR clients need reachable next-hops | **HIGH** |
| 1.5.5 | RR clients are configured with `route-reflector-client` on the RR, not on the client | **LOW** |

### 1.6 Graceful Restart and Graceful Shutdown

| # | Checkpoint | Severity |
|---|---|---|
| 1.6.1 | BGP graceful restart (RFC 4724) is enabled for planned maintenance scenarios | **MEDIUM** |
| 1.6.2 | Graceful restart timers are tuned (restart-time, stalepath-time) for the environment | **MEDIUM** |
| 1.6.3 | BGP graceful shutdown (RFC 8326) using GRACEFUL_SHUTDOWN community is implemented for maintenance | **MEDIUM** |
| 1.6.4 | Graceful restart helper mode is enabled on peers that support it | **LOW** |

### Vendor Examples

**IOS/IOS-XE graceful restart**:
```
router bgp 65001
 bgp graceful-restart
 bgp graceful-restart restart-time 120
 bgp graceful-restart stalepath-time 360
```

**JunOS graceful restart**:
```
routing-options {
    graceful-restart;
}
```

### 1.7 Bogon and Martian Filtering

| # | Checkpoint | Severity |
|---|---|---|
| 1.7.1 | Inbound eBGP filters reject RFC 1918 prefixes (10.0.0.0/8, 172.16.0.0/12, 192.168.0.0/16) | **CRITICAL** |
| 1.7.2 | Inbound eBGP filters reject RFC 6598 (100.64.0.0/10) shared address space | **HIGH** |
| 1.7.3 | Inbound eBGP filters reject documentation prefixes (192.0.2.0/24, 198.51.100.0/24, 203.0.113.0/24) | **HIGH** |
| 1.7.4 | Inbound eBGP filters reject multicast (224.0.0.0/4), loopback (127.0.0.0/8), link-local (169.254.0.0/16) | **HIGH** |
| 1.7.5 | Inbound eBGP filters reject too-specific prefixes (>/24 for IPv4, >/48 for IPv6) | **MEDIUM** |
| 1.7.6 | Inbound eBGP filters reject too-short prefixes (</8 for IPv4) | **MEDIUM** |
| 1.7.7 | Bogon prefix-lists are maintained and updated (Team Cymru or similar reference) | **MEDIUM** |
| 1.7.8 | RPKI Route Origin Validation (ROV) is enabled where supported | **MEDIUM** |

### Bogon Prefix-List Example (IOS)

```
ip prefix-list BOGONS seq 5 deny 0.0.0.0/8 le 32
ip prefix-list BOGONS seq 10 deny 10.0.0.0/8 le 32
ip prefix-list BOGONS seq 15 deny 100.64.0.0/10 le 32
ip prefix-list BOGONS seq 20 deny 127.0.0.0/8 le 32
ip prefix-list BOGONS seq 25 deny 169.254.0.0/16 le 32
ip prefix-list BOGONS seq 30 deny 172.16.0.0/12 le 32
ip prefix-list BOGONS seq 35 deny 192.0.0.0/24 le 32
ip prefix-list BOGONS seq 40 deny 192.0.2.0/24 le 32
ip prefix-list BOGONS seq 45 deny 192.168.0.0/16 le 32
ip prefix-list BOGONS seq 50 deny 198.18.0.0/15 le 32
ip prefix-list BOGONS seq 55 deny 198.51.100.0/24 le 32
ip prefix-list BOGONS seq 60 deny 203.0.113.0/24 le 32
ip prefix-list BOGONS seq 65 deny 224.0.0.0/3 le 32
ip prefix-list BOGONS seq 70 deny 240.0.0.0/4 le 32
ip prefix-list BOGONS seq 999 permit 0.0.0.0/0 le 24
```

### 1.8 BFD (Bidirectional Forwarding Detection)

| # | Checkpoint | Severity |
|---|---|---|
| 1.8.1 | BFD is enabled on BGP sessions for sub-second failure detection (RFC 5880, 5882) | **MEDIUM** |
| 1.8.2 | BFD timers are appropriate for the link type (point-to-point vs. multihop) | **MEDIUM** |
| 1.8.3 | BFD is enabled on IGP adjacencies (OSPF, IS-IS) for fast convergence | **MEDIUM** |
| 1.8.4 | BFD echo mode is disabled where it causes false positives (e.g., through firewalls) | **LOW** |

### Vendor Examples

**IOS/IOS-XE BFD for BGP**:
```
interface GigabitEthernet0/0
 bfd interval 300 min_rx 300 multiplier 3

router bgp 65001
 neighbor 10.0.0.1 fall-over bfd
```

**NX-OS BFD for BGP**:
```
feature bfd
router bgp 65001
  neighbor 10.0.0.1
    bfd
```

**JunOS BFD for BGP**:
```
protocols {
    bgp {
        group EBGP-PEERS {
            bfd-liveness-detection {
                minimum-interval 300;
                multiplier 3;
            }
        }
    }
}
```

---

## 2. OSPF (Open Shortest Path First)

### 2.1 Area Design

| # | Checkpoint | Severity |
|---|---|---|
| 2.1.1 | Area 0 backbone is contiguous (no partitioned backbone without virtual links) | **HIGH** |
| 2.1.2 | Stub, totally stub, or NSSA areas are used to reduce LSA flooding in non-backbone areas | **MEDIUM** |
| 2.1.3 | Number of routers per area is within design limits (typically < 50-80 routers per area) | **MEDIUM** |
| 2.1.4 | Inter-area route summarization is configured on ABRs | **MEDIUM** |
| 2.1.5 | Virtual links are documented and justified (temporary measure, not permanent design) | **MEDIUM** |
| 2.1.6 | OSPF router-id is explicitly set (not auto-selected from highest loopback) | **LOW** |
| 2.1.7 | Loopback interfaces used as router-IDs are assigned /32 masks | **LOW** |

### 2.2 OSPF Authentication

| # | Checkpoint | Severity |
|---|---|---|
| 2.2.1 | OSPF authentication is enabled on all interfaces in production areas | **HIGH** |
| 2.2.2 | SHA-HMAC (RFC 5709) or MD5 authentication is used (not cleartext, type 0) | **HIGH** |
| 2.2.3 | OSPF authentication key IDs support hitless key rotation (multiple keys with key chains) | **MEDIUM** |
| 2.2.4 | Virtual links use authentication matching the transit area | **HIGH** |
| 2.2.5 | Authentication is configured at the area level for consistency, not per-interface ad hoc | **LOW** |

### What to Look For

```regex
# OSPF without authentication (IOS)
(?i)ip\s+ospf\s+\d+\s+area\s+\d+
# Verify: ip ospf authentication or area <id> authentication exists

# Cleartext OSPF authentication
(?i)ip\s+ospf\s+authentication\s*$
# Missing: message-digest keyword

# OSPF authentication key in cleartext
(?i)ip\s+ospf\s+authentication-key\s+\S+
# Should use: ip ospf message-digest-key <id> md5 7 <encrypted>
```

### Vendor Examples

**IOS/IOS-XE OSPF MD5 authentication**:
```
interface GigabitEthernet0/0
 ip ospf authentication message-digest
 ip ospf message-digest-key 1 md5 7 <encrypted>
```

**NX-OS OSPF authentication**:
```
interface Ethernet1/1
  ip ospf authentication message-digest
  ip ospf message-digest-key 1 md5 3 <encrypted>
```

**JunOS OSPF authentication**:
```
protocols {
    ospf {
        area 0.0.0.0 {
            interface ge-0/0/0.0 {
                authentication {
                    md5 1 key "$9$encrypted";
                }
            }
        }
    }
}
```

### 2.3 Passive Interfaces

| # | Checkpoint | Severity |
|---|---|---|
| 2.3.1 | `passive-interface default` is configured with explicit `no passive-interface` on adjacency links | **HIGH** |
| 2.3.2 | All loopback, management, and end-user-facing interfaces are passive | **HIGH** |
| 2.3.3 | Only interfaces forming deliberate adjacencies are non-passive | **MEDIUM** |

### What to Look For

```regex
# Missing passive-interface default
(?i)router\s+ospf\s+\d+
# Verify: passive-interface default exists in the OSPF config block

# Active OSPF on user-facing interfaces
(?i)interface\s+(Vlan|vlan)\d+\s*\n[^!]*ip\s+ospf\s+\d+\s+area
# Check: is this a user VLAN that should be passive?
```

### Vendor Examples

**IOS/IOS-XE passive default**:
```
router ospf 1
 passive-interface default
 no passive-interface GigabitEthernet0/0
 no passive-interface GigabitEthernet0/1
```

**JunOS passive equivalent**:
```
protocols {
    ospf {
        area 0.0.0.0 {
            interface lo0.0 passive;
            interface ge-0/0/0.0;
            interface ge-0/0/1.0;
        }
    }
}
! Note: JunOS does not advertise interfaces not listed under OSPF
```

### 2.4 OSPF Timers and SPF Throttle

| # | Checkpoint | Severity |
|---|---|---|
| 2.4.1 | SPF throttle timers are configured (initial, hold, max-wait) to prevent CPU exhaustion during instability | **MEDIUM** |
| 2.4.2 | LSA throttle timers are configured to limit LSA generation frequency | **MEDIUM** |
| 2.4.3 | Hello and dead timers are consistent on both sides of every adjacency | **HIGH** |
| 2.4.4 | Timer values are documented and justified (default 10/40 is not always appropriate) | **LOW** |
| 2.4.5 | BFD is used for fast failure detection instead of aggressive hello timers | **MEDIUM** |

### Vendor Examples

**IOS/IOS-XE SPF throttle**:
```
router ospf 1
 timers throttle spf 50 200 5000
 timers throttle lsa all 50 200 5000
 ! Initial: 50ms, hold: 200ms, max-wait: 5000ms
```

**NX-OS SPF throttle**:
```
router ospf 1
  timers throttle spf 50 200 5000
  timers throttle lsa 50 200 5000
```

**JunOS SPF throttle**:
```
protocols {
    ospf {
        spf-options {
            delay 50;
            holddown 200;
            rapid-runs 3;
        }
    }
}
```

---

## 3. IS-IS (Intermediate System to Intermediate System)

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 3.1 | IS-IS authentication is enabled (ISO 10589 or RFC 5304 HMAC-MD5, RFC 5310 SHA) | **HIGH** |
| 3.2 | IS-IS level (L1, L2, L1/L2) is explicitly set per interface to prevent unnecessary adjacencies | **MEDIUM** |
| 3.3 | IS-IS metric style is set to `wide` (RFC 3784) for modern metric support | **MEDIUM** |
| 3.4 | IS-IS NET (Network Entity Title) follows a consistent addressing scheme | **LOW** |
| 3.5 | Passive interfaces are configured for non-adjacency links | **HIGH** |
| 3.6 | SPF throttle timers are tuned | **MEDIUM** |
| 3.7 | IS-IS overload bit (set-overload-bit) is configured for graceful insertion into the network | **MEDIUM** |
| 3.8 | BFD is enabled for IS-IS adjacencies | **MEDIUM** |

### Vendor Examples

**IOS/IOS-XE IS-IS authentication**:
```
key chain ISIS-KEY
 key 1
  key-string 7 <encrypted>
  cryptographic-algorithm hmac-sha-256

router isis CORE
 authentication mode md5 level-2
 authentication key-chain ISIS-KEY level-2
```

**JunOS IS-IS authentication**:
```
protocols {
    isis {
        level 2 {
            authentication-key "$9$encrypted";
            authentication-type md5;
        }
    }
}
```

---

## 4. EIGRP (Enhanced Interior Gateway Routing Protocol)

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 4.1 | EIGRP authentication is enabled on all interfaces (MD5 or SHA-256 with named mode) | **HIGH** |
| 4.2 | EIGRP named mode is used over classic mode for modern feature support | **MEDIUM** |
| 4.3 | Stub routing is configured on spoke routers in hub-and-spoke topologies | **MEDIUM** |
| 4.4 | Passive interfaces are configured on all non-adjacency interfaces | **HIGH** |
| 4.5 | Route summarization is configured on boundaries to reduce query scope | **MEDIUM** |
| 4.6 | EIGRP graceful restart (NSF) is configured where supported | **LOW** |
| 4.7 | Variance and offset-list are documented if used for unequal-cost load balancing | **LOW** |
| 4.8 | EIGRP network statements use specific masks, not classful boundaries | **MEDIUM** |

### What to Look For

```regex
# EIGRP without authentication
(?i)router\s+eigrp\s+\d+
# Verify: authentication mode or key chain configured

# Classic EIGRP (prefer named mode)
(?i)router\s+eigrp\s+\d+\s*\n\s*network\s+
# Suggest: router eigrp <name> / address-family ipv4 unicast autonomous-system <asn>

# Classful network statement
(?i)network\s+\d+\.\d+\.\d+\.\d+\s*$
# Missing: wildcard mask (e.g., network 10.0.0.0 0.0.0.255)
```

### Vendor Examples

**IOS/IOS-XE EIGRP named mode with SHA-256**:
```
router eigrp ENTERPRISE
 address-family ipv4 unicast autonomous-system 100
  af-interface default
   passive-interface
   authentication mode hmac-sha-256 <password>
  exit-af-interface
  af-interface GigabitEthernet0/0
   no passive-interface
  exit-af-interface
  network 10.0.0.0 0.0.255.255
  eigrp stub connected summary
```

---

## 5. Static Routing

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 5.1 | Static routes use next-hop IP addresses, not exit interfaces only (to avoid proxy-ARP issues on multi-access links) | **MEDIUM** |
| 5.2 | Floating static routes have administrative distances documented and validated | **MEDIUM** |
| 5.3 | Null route (null0/discard) is configured as a summary for originated prefixes to prevent routing loops | **HIGH** |
| 5.4 | Static routes include tags or descriptions for operational clarity | **LOW** |
| 5.5 | Recursive static routes are avoided or carefully validated (risk of route flapping) | **MEDIUM** |
| 5.6 | Static routes use BFD or IP SLA tracking for failure detection | **MEDIUM** |
| 5.7 | Default routes are minimal and intentional (not multiple conflicting defaults) | **HIGH** |
| 5.8 | Static routes do not silently override more specific dynamic routes without documentation | **MEDIUM** |

### What to Look For

```regex
# Static route with exit interface only on Ethernet (proxy-ARP risk)
(?i)ip\s+route\s+\S+\s+\S+\s+(Ethernet|GigabitEthernet|TenGigabitEthernet|FastEthernet)\S+\s*$

# Floating static without explicit AD
(?i)ip\s+route\s+\S+\s+\S+\s+\S+\s*$
# Note: default AD for static is 1; floating statics need higher AD

# Missing null route for aggregates
(?i)router\s+bgp\s+\d+\s*\n[^!]*network\s+(\S+)\s+mask\s+(\S+)
# Verify: ip route <network> <mask> null0 exists
```

### Vendor Examples

**IOS null route for BGP aggregate**:
```
! Originate 10.1.0.0/16 via BGP
ip route 10.1.0.0 255.255.0.0 Null0 name BGP-AGGREGATE
router bgp 65001
 network 10.1.0.0 mask 255.255.0.0
```

**IOS floating static with tracking**:
```
ip sla 1
 icmp-echo 10.0.0.1 source-interface GigabitEthernet0/0
 frequency 5
ip sla schedule 1 life forever start-time now

track 1 ip sla 1 reachability

ip route 0.0.0.0 0.0.0.0 10.0.0.1 track 1
ip route 0.0.0.0 0.0.0.0 10.0.1.1 250
! Floating static with AD 250 as backup
```

**JunOS static route with next-hop**:
```
routing-options {
    static {
        route 0.0.0.0/0 {
            next-hop 10.0.0.1;
            qualified-next-hop 10.0.1.1 {
                preference 250;
            }
        }
        route 10.1.0.0/16 {
            discard;
            tag 100;
        }
    }
}
```

---

## 6. Spanning Tree Protocol (STP)

### 6.1 Root Bridge Placement

| # | Checkpoint | Severity |
|---|---|---|
| 6.1.1 | Root bridge is explicitly configured (priority set), not left to default MAC-based election | **HIGH** |
| 6.1.2 | Primary and secondary root bridges are designated distribution/core switches | **HIGH** |
| 6.1.3 | Root bridge is a high-performance, centrally located switch (not an access switch) | **HIGH** |
| 6.1.4 | Root bridge priority is documented for each VLAN or MST instance | **MEDIUM** |
| 6.1.5 | Root guard is enabled on ports where root bridges should never be learned | **HIGH** |

### What to Look For

```regex
# STP root bridge explicitly set
(?i)spanning-tree\s+vlan\s+\S+\s+root\s+(primary|secondary)
(?i)spanning-tree\s+vlan\s+\S+\s+priority\s+\d+

# Missing root guard on access-facing downlinks
(?i)interface\s+\S+\s*\n[^!]*switchport\s+mode\s+access
# Verify: spanning-tree guard root on distribution ports facing access switches
```

### Vendor Examples

**IOS/IOS-XE root bridge**:
```
spanning-tree vlan 1-4094 root primary
! Or explicit priority:
spanning-tree vlan 1-4094 priority 4096
```

**NX-OS root bridge**:
```
spanning-tree vlan 1-4094 priority 4096
```

**EOS root bridge**:
```
spanning-tree vlan 1-4094 priority 4096
```

### 6.2 PortFast and BPDU Guard

| # | Checkpoint | Severity |
|---|---|---|
| 6.2.1 | PortFast (or edge port) is enabled on all access ports connecting to end hosts | **MEDIUM** |
| 6.2.2 | BPDU Guard is enabled globally or on all PortFast ports | **HIGH** |
| 6.2.3 | PortFast is NOT enabled on trunk ports or inter-switch links | **CRITICAL** |
| 6.2.4 | BPDU Guard err-disable recovery is configured with a reasonable interval | **MEDIUM** |
| 6.2.5 | BPDU Filter is NOT used on access ports (defeats STP protection) | **HIGH** |

### What to Look For

```regex
# PortFast on trunk port (dangerous)
(?i)interface\s+\S+\s*\n[^!]*switchport\s+mode\s+trunk\s*\n[^!]*spanning-tree\s+portfast

# Missing BPDU guard
(?i)spanning-tree\s+portfast
# Verify: spanning-tree bpduguard enable or global default

# BPDU filter (dangerous on access ports)
(?i)spanning-tree\s+bpdufilter\s+enable
```

### Vendor Examples

**IOS/IOS-XE global PortFast + BPDU Guard**:
```
spanning-tree portfast default
spanning-tree portfast bpduguard default

! Per-interface override for uplinks:
interface GigabitEthernet0/1
 spanning-tree portfast disable

! Err-disable recovery:
errdisable recovery cause bpduguard
errdisable recovery interval 300
```

**NX-OS PortFast + BPDU Guard**:
```
spanning-tree port type edge default
spanning-tree port type edge bpduguard default
```

**EOS PortFast + BPDU Guard**:
```
spanning-tree portfast default
spanning-tree portfast bpduguard default
```

### 6.3 Loop Guard and Other Protections

| # | Checkpoint | Severity |
|---|---|---|
| 6.3.1 | Loop Guard is enabled on non-designated ports (root and alternate ports) | **HIGH** |
| 6.3.2 | UDLD is enabled on fiber uplinks for unidirectional link detection | **MEDIUM** |
| 6.3.3 | Loop Guard and Root Guard are not applied on the same port (mutually exclusive) | **MEDIUM** |
| 6.3.4 | STP mode is RPVST+ (Rapid PVST+) or MST, not legacy PVST or CST | **MEDIUM** |
| 6.3.5 | MST configuration (name, revision, VLAN-to-instance mapping) is consistent across all switches | **CRITICAL** |
| 6.3.6 | STP diameter/max-age is appropriate for the topology (default 7 switch diameter) | **LOW** |

### What to Look For

```regex
# Legacy STP mode
(?i)spanning-tree\s+mode\s+pvst\b
# Should be: spanning-tree mode rapid-pvst or spanning-tree mode mst

# Inconsistent MST config (check across all switches)
(?i)spanning-tree\s+mst\s+configuration\s*\n\s*name\s+(\S+)\s*\n\s*revision\s+(\d+)
# All switches must match name and revision

# Missing loop guard
(?i)spanning-tree\s+loopguard\s+default
```

### Vendor Examples

**IOS/IOS-XE STP hardening**:
```
spanning-tree mode rapid-pvst
spanning-tree loopguard default
spanning-tree pathcost method long

! Root guard on distribution downlinks:
interface GigabitEthernet0/1
 spanning-tree guard root

! UDLD on fiber uplinks:
udld enable
interface TenGigabitEthernet0/1
 udld port aggressive
```

---

## 7. VTP (VLAN Trunking Protocol)

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 7.1 | VTP mode is `transparent` or `off` (not `server` or `client`) in modern deployments | **HIGH** |
| 7.2 | If VTP server mode is used, VTP version 3 with password authentication is configured | **HIGH** |
| 7.3 | VTP domain name is explicitly set and consistent | **MEDIUM** |
| 7.4 | VTP pruning is enabled if VTP server mode is in use | **MEDIUM** |
| 7.5 | VTP password is configured if VTP is active (prevents rogue switch VLAN wipe) | **HIGH** |

### What to Look For

```regex
# VTP server mode (risky in large networks)
(?i)vtp\s+mode\s+server
(?i)vtp\s+mode\s+client

# Missing VTP password
(?i)vtp\s+domain\s+\S+
# Verify: vtp password <password> exists

# VTP version 1 or 2 (prefer v3 or transparent mode)
(?i)vtp\s+version\s+[12]\b
```

---

## 8. VLAN Trunking

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 8.1 | Trunk ports use `switchport mode trunk` with `switchport nonegotiate` (disable DTP) | **HIGH** |
| 8.2 | Access ports use `switchport mode access` with DTP disabled | **HIGH** |
| 8.3 | Trunk allowed VLAN list is pruned to only required VLANs (not `all`) | **HIGH** |
| 8.4 | Native VLAN on trunks is changed from VLAN 1 to an unused VLAN | **HIGH** |
| 8.5 | Native VLAN is consistent on both ends of every trunk | **HIGH** |
| 8.6 | Native VLAN is tagged on trunks (802.1Q tag on native) | **MEDIUM** |
| 8.7 | Unused ports are shut down and assigned to a dead-end VLAN | **HIGH** |
| 8.8 | DTP (Dynamic Trunking Protocol) is disabled globally or per interface | **HIGH** |

### What to Look For

```regex
# DTP enabled (default on IOS)
(?i)interface\s+\S+\s*\n[^!]*switchport\s+mode\s+(dynamic\s+(auto|desirable))

# Missing nonegotiate on trunk
(?i)switchport\s+mode\s+trunk
# Verify: switchport nonegotiate present

# Default native VLAN
(?i)switchport\s+trunk\s+native\s+vlan\s+1\b

# Trunk allowing all VLANs
(?i)switchport\s+trunk\s+allowed\s+vlan\s+all
# Or missing: switchport trunk allowed vlan <list>

# Unused ports not shut down
(?i)interface\s+\S+\s*\n\s*!
# Verify: shutdown on unused ports
```

### Vendor Examples

**IOS/IOS-XE trunk hardening**:
```
interface GigabitEthernet0/1
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport nonegotiate
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30
```

**IOS/IOS-XE access port hardening**:
```
interface GigabitEthernet0/2
 switchport mode access
 switchport access vlan 10
 switchport nonegotiate
 spanning-tree portfast
 spanning-tree bpduguard enable
```

**Unused port lockdown**:
```
interface range GigabitEthernet0/45-48
 switchport mode access
 switchport access vlan 999
 shutdown
```

---

## 9. EtherChannel / Port-Channel

### Checkpoints

| # | Checkpoint | Severity |
|---|---|---|
| 9.1 | EtherChannel uses LACP (active/active or active/passive), not static `mode on` | **HIGH** |
| 9.2 | LACP rate is set to `fast` on critical links for sub-second failure detection | **MEDIUM** |
| 9.3 | All member interfaces have identical configuration (speed, duplex, VLAN, STP) | **CRITICAL** |
| 9.4 | Port-channel interface has the desired L2/L3 configuration (not just member ports) | **HIGH** |
| 9.5 | Load-balancing hash is configured for optimal traffic distribution (src-dst-ip or src-dst-port) | **MEDIUM** |
| 9.6 | LACP system priority is set on the device that should control port selection | **LOW** |
| 9.7 | Maximum and minimum bundle size is configured where supported | **MEDIUM** |
| 9.8 | mLACP or MC-LAG is used for multi-chassis link aggregation to avoid STP dependence | **MEDIUM** |

### What to Look For

```regex
# Static EtherChannel (risky)
(?i)channel-group\s+\d+\s+mode\s+on

# Missing LACP (no channel-group at all on bundled ports)
(?i)interface\s+(Port-channel|port-channel)\d+

# Mismatched member config (requires cross-interface comparison)
(?i)channel-group\s+(\d+)\s+mode
# Compare: speed, duplex, switchport mode, allowed VLANs across all members

# LACP slow rate (default)
(?i)lacp\s+rate\s+normal
# Prefer: lacp rate fast on critical links
```

### Vendor Examples

**IOS/IOS-XE LACP**:
```
interface range GigabitEthernet0/1-2
 channel-group 1 mode active
 lacp rate fast

interface Port-channel1
 switchport mode trunk
 switchport trunk native vlan 999
 switchport trunk allowed vlan 10,20,30

port-channel load-balance src-dst-ip
```

**NX-OS LACP with vPC**:
```
feature lacp
feature vpc

vpc domain 1
  role priority 100
  peer-keepalive destination 192.168.1.2

interface port-channel1
  vpc 1
  switchport mode trunk

interface Ethernet1/1-2
  channel-group 1 mode active
  lacp rate fast
```

**JunOS LACP**:
```
chassis {
    aggregated-devices {
        ethernet {
            device-count 4;
        }
    }
}

interfaces {
    ge-0/0/0 {
        ether-options {
            802.3ad ae0;
        }
    }
    ge-0/0/1 {
        ether-options {
            802.3ad ae0;
        }
    }
    ae0 {
        aggregated-ether-options {
            lacp {
                active;
                periodic fast;
            }
        }
        unit 0 {
            family ethernet-switching {
                interface-mode trunk;
                vlan members [vlan10 vlan20 vlan30];
            }
        }
    }
}
```

**EOS LACP**:
```
interface Ethernet1-2
   channel-group 1 mode active
   lacp rate fast

interface Port-Channel1
   switchport mode trunk
   switchport trunk allowed vlan 10,20,30

port-channel load-balance src-dst-ip-port
```

---

## Review Procedure Summary

When evaluating routing and switching configurations:

1. **BGP sessions**: Verify authentication, max-prefix, inbound/outbound filters, bogon
   filtering, and RPKI validation on every eBGP session. Check route reflector redundancy
   and cluster-ID consistency for iBGP.
2. **IGP design**: Confirm area topology, authentication, passive interfaces, SPF throttle,
   and BFD enablement for OSPF/IS-IS/EIGRP.
3. **Static routes**: Validate next-hop usage, null routes for aggregates, floating static
   administrative distances, and tracking mechanisms.
4. **STP hardening**: Verify root bridge placement, PortFast/BPDU Guard on access ports,
   loop guard on non-designated ports, and mode selection (RPVST+ or MST).
5. **VLAN trunking**: Confirm DTP is disabled, native VLAN is changed from 1, trunk allowed
   VLAN lists are pruned, and unused ports are shut down.
6. **EtherChannel**: Verify LACP is used (not static mode on), member configs are consistent,
   and load-balancing hash is appropriate.
7. **Classify each finding** using the severity levels defined in each section.
8. **Prioritize remediation**: CRITICAL findings (unauthenticated eBGP, PortFast on trunks,
   missing route filters) first, then HIGH, MEDIUM, LOW.

---

## Quick Reference: Severity Summary

| Severity | Routing & Switching Findings |
|----------|------------------------------|
| CRITICAL | Unauthenticated eBGP sessions; no inbound eBGP prefix filtering; PortFast on trunk/inter-switch links; inconsistent MST configuration across switches; mismatched EtherChannel member configs; bogon prefixes accepted on eBGP |
| HIGH | Missing max-prefix limits; no outbound eBGP filters; missing OSPF/IS-IS/EIGRP authentication; no passive-interface default; STP root bridge not explicitly set; BPDU Guard disabled; DTP enabled; native VLAN 1; trunk allowing all VLANs; static EtherChannel (mode on); conflicting default routes; VTP server mode without v3+password; unused ports not shut down |
| MEDIUM | Missing BFD; no SPF throttle tuning; route summarization absent; missing UDLD on fiber; LACP slow rate; no graceful restart; stub areas not optimized; STP mode legacy PVST; floating static AD undocumented; recursive static routes; BGP community misconfiguration |
| LOW | Missing interface descriptions; router-ID not explicit; EIGRP classic mode; STP diameter/max-age non-default without documentation; LACP system priority not set; missing route tags |
| INFO | Well-structured area design; proper use of stub/NSSA areas; correct bogon filtering; good use of RPKI ROV; well-documented route policies |

---

## Standards References

- **RFC 2385** - Protection of BGP Sessions via TCP MD5 Signature Option
- **RFC 4271** - A Border Gateway Protocol 4 (BGP-4)
- **RFC 4724** - Graceful Restart Mechanism for BGP
- **RFC 5880** - Bidirectional Forwarding Detection (BFD)
- **RFC 5925** - The TCP Authentication Option (TCP-AO)
- **RFC 7454** - BGP Operations and Security (BCP 194)
- **RFC 8092** - BGP Large Communities
- **RFC 8326** - Graceful BGP Session Shutdown
- **RFC 2328** - OSPF Version 2
- **RFC 5709** - OSPFv2 HMAC-SHA Cryptographic Authentication
- **RFC 5308** - Routing IPv6 with IS-IS
- **RFC 7166** - Supporting Authentication Trailer for OSPFv3
- **CIS Cisco IOS Benchmark** - Sections on routing protocol security
- **NIST SP 800-189** - Resilient Interdomain Traffic Exchange: BGP Security and DDoS Mitigation
- **MANRS** (Mutually Agreed Norms for Routing Security) - Filtering, anti-spoofing, coordination
